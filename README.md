# rtl-dpram-design-verification

<img width="1387" height="970" alt="image" src="https://github.com/user-attachments/assets/aca9158f-2f38-45c7-9ac1-c34d1f006246" />

# Introduction

A Dual Port RAM is a memory structure that provides two independent ports for accessing a single underlying storage array. Unlike single-port memory, which can only perform one operation (read or write) per clock cycle, a dual-port RAM allows two operations to occur simultaneously — typically one write and one read — without resource contention.
In the simplest and most common configuration (sometimes called a simple dual-port RAM), Port A is dedicated to writing and Port B is dedicated to reading. Each port has its own clock, address bus, and data bus. This separation is what makes dual-port RAM the natural storage element inside asynchronous FIFOs, where the write clock and read clock are independent.

# The Storage Array

At the core of the DPRAM is a two-dimensional array of storage elements. Each element holds one bit. The array is organized as DEPTH rows (locations) by DATA_WIDTH columns (bits per word). The DEPTH is determined by 2^ADDR_WIDTH, where ADDR_WIDTH is the number of address bits.
For example, a DPRAM with ADDR_WIDTH=4 and DATA_WIDTH=8 provides 16 locations, each 8 bits wide, for a total storage capacity of 128 bits (16 bytes).

# Write Port Operation

The write port is a synchronous interface controlled by the write clock (wr_clk). On each rising edge of wr_clk, if the write enable signal (wr_en) is asserted, the data present on wr_data is stored into the memory location addressed by wr_addr. The write is atomic — all DATA_WIDTH bits are captured in a single clock edge. If wr_en is deasserted, the memory contents remain unchanged regardless of what appears on wr_addr or wr_data.

# Read Port Operation

The read port in this design uses a combinational (asynchronous) read. The data at the memory location addressed by rd_addr is continuously driven onto rd_data without any clock dependency. This means rd_data updates immediately when rd_addr changes, with only combinational propagation delay through the memory array and output multiplexer.
This combinational read style is standard for Cummings-style asynchronous FIFOs. For FPGA targets where block RAM requires registered outputs, a pipeline register can be added downstream.

# Independent Clock Domains

The critical architectural feature of this DPRAM is that the write port and read port operate on completely independent clocks. The write clock (wr_clk) and read clock (rd_clk) may differ in frequency, phase, and duty cycle. There is no timing relationship assumed between them.
This independence is what makes the DPRAM suitable as the storage backbone of an asynchronous FIFO. The FIFO’s pointer logic and synchronizers handle the clock domain crossing; the DPRAM itself simply provides a shared storage array that both domains can access through their respective ports.

# Simultaneous Access Behavior

When both ports access the memory simultaneously, the behavior depends on the address relationship:
•	Different addresses: Both operations proceed independently with no conflict. The write port stores data at its addressed location while the read port retrieves data from a different location. This is the normal operating mode.
•	Same address: This constitutes a read-write collision. In this design, the write port takes priority — the new data is written first, and the read port returns the previously stored value (the value that was in the location before the current write). This is the “write-first” or “read-during-write returns old data” behavior, which is the safest default for CDC applications.

# Why Dual Port RAM Matters

The dual-port architecture is not just a convenience — it is a fundamental enabler for several classes of digital design:
1.	Clock Domain Crossing: By allowing independent clocks on each port, DPRAM enables data transfer between clock domains without requiring the memory itself to handle synchronization. The synchronization responsibility is delegated to the surrounding logic (Gray-coded pointers and two-flop synchronizers in the case of an async FIFO).

2.	Throughput: A single-port RAM forces serialization — you cannot write and read in the same cycle. DPRAM eliminates this bottleneck, enabling full-duplex data transfer.

3.	FPGA Optimization: Modern FPGAs include dedicated Block RAM (BRAM) primitives that natively support dual-port access. A well-written DPRAM module is automatically inferred as BRAM by the synthesis tool, consuming dedicated resources rather than general-purpose logic.

4.	Scalability: The parameterized design allows the same module to serve as a 16-byte buffer in a UART interface or a 64KB frame buffer in a video pipeline, simply by changing DATA_WIDTH and ADDR_WIDTH.

