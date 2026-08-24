---
layout: default
title: "Part 2: The Fabric"
nav_order: 3
permalink: /fabric
---
# Building USoC Part 2: The Fabric

In Part 1, we broke down the standard Wishbone interface and established the
formal rules required to guarantee its correctness. Now, it's time to stitch
everything together into a functional system. 

Because we are on a tight timeline—next week we are diving straight into PHYs,
the PIPE interface, and building a UART—we need to assemble our core System-on-Chip
(SoC) fabric cleanly, elegantly, and without wasting any time on unnecessary
architectural bloat. 

## Decoders vs. Arbiters: The Multi-Master Problem

To understand how our interconnect works, we need to quickly contrast two
fundamental building blocks: 

* **Decoder:** Logic that allows a *single master* to talk to *multiple slaves*
by routing signals based on an address map.
* **Arbiter:** Logic that allows *multiple masters* to talk to a *single slave*
by deciding who gets access to the bus.

A traditional on-chip bus combines both arbiters and decoders. However,
arbitration is time-multiplexed by nature: only one master can occupy the
bus at any given moment, creating an engineering bottleneck. 

So why do we even want multiple masters? If a CPU has to manually read every
word from a high-speed Ethernet or USB peripheral and copy it into memory line
by line, it wastes massive amounts of processing cycles. This is why the Direct
Memory Access (DMA) engine was invented. A DMA is a peripheral that acts as a
slave when the CPU configures its control registers, but morphs into a bus master
 to copy packets directly to and from memory independently. 

In our USYS architecture, we took a radical approach: **every single device uses
a DMA.** 

Our core peripheral DMA is called UBUS. By pairing UBUS with our single RISC-V
core, our entire SoC contains exactly two bus masters. Because we restricted our
design to exactly two masters, we can bypass traditional bus arbiters entirely.
Instead, we route traffic through a standard dual-ported SRAM module. Dual-port
SRAM naturally allows for concurrent reads and writes from two separate masters
simultaneously, as long as they aren't writing to the exact same memory address. 

## Implementing the Wishbone Decoder

While we don't need an arbiter, our masters still require a decoder to route
transactions to the correct memory addresses or peripherals. 

First, we determine which slave the master is targeting by evaluating the address
lines: 

```python
slave0_match = (master.addr >= self.slave0_addr) & (master.addr < (self.slave0_addr + self.slave0_size))
```

If a match is found, we subtract the base address offset and dynamically bridge
the master and slave lines. 

But what happens if a master attempts to access an invalid address? In hardware
design, a bad address can cause an infinite stall, hanging the bus forever. To
solve this, our decoder catches invalid addresses, manually asserts an
acknowledgment signal (ack) to gracefully terminate the broken transaction, and
raises a bus_error flag directly to the CPU. 

Here is how the decoder looks in Amaranth HDL: 

```python
outstanding = Signal(8)
active_slave = Signal(2)
req_issued = master.cyc & master.stb & ~master.stall
res_rcvd   = master.cyc & master.ack

with m.If(master.cyc):
    m.d.sync += outstanding.eq(outstanding + req_issued - res_rcvd)
    
    # Route to Slave 0
    with m.If(slave0_match & ((active_slave == 1) | (active_slave == 0))):
        m.d.sync += active_slave.eq(1)
        m.d.comb += [
            slave0.cyc.eq(master.cyc),
            slave0.stb.eq(master.stb),
            slave0.we.eq(master.we),
            slave0.addr.eq(master.addr[:slave0_bits]),
            slave0.sel.eq(master.sel),
            slave0.dat_w.eq(master.dat_w),
            master.dat_r.eq(slave0.dat_r),
            master.ack.eq(slave0.ack),
            master.stall.eq(slave0.stall),
        ]
        
    # Route to Slave 1
    with m.Elif(slave1_match & ((active_slave == 2) | (active_slave == 0))):
        # ... Mapping logic for slave 1 goes here ...
        
    # Handle Invalid Addresses (Deadlock Prevention)
    with m.Else():
        m.d.sync += active_slave.eq(3)
        m.d.comb += [
            master.ack.eq(outstanding > 0),
            bus_error.eq(1),
        ]
with m.Else():
    m.d.sync += outstanding.eq(0)
    m.d.sync += active_slave.eq(0)
```

## The Dual-Port SRAM Module

With our decoder ready, we need an SRAM block to store data and instructions.
To hit our target performance metrics on low-cost FPGAs, we need the hardware
synthesis tools (like Yosys) to automatically infer hardware block RAM (BRAM)
rather than constructing memory out of thousands of power-hungry LUTs. 

To guarantee block RAM inference, our memory architecture relies on a predictable,
synchronous design featuring strict two-cycle read and write pipelines, paired
with byte-lane selection (sel) masks. 

Here is the core block configuration for Port A of our dual-port SRAM: 

```python
# Instantiate the raw synchronous memory core
mem = m.submodules.ram_core = memory.Memory(shape=32, depth=self.depth, init=[])

# Define our synchronous read and write ports
mem_r_a = mem.read_port(domain='sync')
mem_w_a = mem.write_port(domain='sync', granularity=8)

ack_a = Signal()
x_ack_a = Signal()

# Combinational port steering
m.d.comb += [
    port_a.stall.eq(0), # SRAM port never stalls the master
    port_a.ack.eq(port_a.cyc & ack_a),
    port_a.dat_r.eq(mem_r_a.data),
    mem_w_a.addr.eq(port_a.addr),
    mem_r_a.addr.eq(port_a.addr),
    mem_w_a.data.eq(port_a.dat_w),
    
    # Conditional Byte-lane Selection for Writes
    mem_w_a.en.eq(Mux(port_a.cyc & port_a.stb & port_a.we, port_a.sel, 0)),
    mem_r_a.en.eq(port_a.cyc & port_a.stb & ~port_a.we),
]

# Register-tracked two-cycle acknowledgment pipeline
m.d.sync += [
    ack_a.eq(port_a.cyc & x_ack_a),
    x_ack_a.eq(port_a.cyc & port_a.stb),
]
```

## The Final Assembly

We now have all our ingredients: Wishbone interfaces, decoders, and a dual-port
synchronous memory module. We can wire them together into our final system fabric
module: UsysFabric (the blue section). 

![](resources/architecture.png)

By laying out clean address spaces for Instruction SRAM (I_SRAM), Data SRAM
(D_SRAM), and Memory-Mapped I/O (MMIO), our two masters (cpu and ubus) can
execute concurrent operations seamlessly across the dual-port memory boundaries. 

```python
class UsysFabric(wiring.Component):
    def __init__(self):
        super().__init__(wiring.Signature({
            "cpu_ibus":      In(wb_sig),
            "cpu_dbus":      In(wb_sig),
            "ubus_master":   In(wb_sig),
            "ubus_slave":    Out(wb_sig),
            "cpu_bus_error": Out(1)
        }))

    def elaborate(self, platform):
        m = Module()

        # Define our absolute base address mappings
        I_SRAM_ADDR = 0x10000000
        D_SRAM_ADDR = 0x20000000
        MMIO_ADDR   = 0x30000000

        # Submodule Instantations
        m.submodules.i_sram = i_sram = WishboneDualPortSram(depth=self.depth_i)
        m.submodules.d_sram = d_sram = WishboneDualPortSram(depth=self.depth_d)
        
        m.submodules.dbus_decoder = dbus_decoder = WishboneDecoder(
            slave0_addr=D_SRAM_ADDR, slave0_size=self.depth_d,
            slave1_addr=MMIO_ADDR,   slave1_size=self.depth_mmio,
        )
        m.submodules.ubus_decoder = ubus_decoder = WishboneDecoder(
            slave0_addr=I_SRAM_ADDR, slave0_size=self.depth_i,
            slave1_addr=D_SRAM_ADDR, slave1_size=self.depth_d,
        )

        # Interconnect Wiring Matrix
        wiring.connect(m, i_sram.port_a, wiring.flipped(self.cpu_ibus))
        wiring.connect(m, dbus_decoder.master, wiring.flipped(self.cpu_dbus))
        wiring.connect(m, ubus_decoder.master, wiring.flipped(self.ubus_master))
        wiring.connect(m, d_sram.port_a, dbus_decoder.slave0)
        wiring.connect(m, wiring.flipped(self.ubus_slave), dbus_decoder.slave1)
        wiring.connect(m, i_sram.port_b, ubus_decoder.slave0)
        wiring.connect(m, d_sram.port_b, ubus_decoder.slave1)
        
        # Route decoder exceptions directly back to the processor core
        m.d.comb += self.cpu_bus_error.eq(dbus_decoder.bus_error)

        return m
```

With our structural fabric fully assembled and verified, our internal parallel
pathways are locked in place. Next week, we take a massive step forward: crossing
from internal silicon logic into the physical outside world. We will look at
implementing the **PHY PIPE interface** and using it to build a hardware UART
from scratch. Stay tuned!
