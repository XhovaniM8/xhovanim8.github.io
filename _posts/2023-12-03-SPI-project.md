---
title: "My First SPI Slave"
date: 2023-06-28 13:00:00 +0000
categories: [project, fpga]
tags: [verilog, fpga, spi, hardware]
excerpt: "Walkthrough of building a custom SPI slave in Verilog on the Arty S7-25 FPGA with debug features."
---


# Building a Custom SPI Slave Module in Verilog

I recently implemented a Serial Peripheral Interface (SPI) slave module targeting the Arty S7-25 FPGA. This project taught me a lot about both SPI protocol implementation and FPGA development. Here's a deep dive into the design and key learnings.

## Core Features

- Full SPI slave interface implementation
- 24-bit transaction support
- Multiple addressable registers
- Built-in debugging capabilities
- LED status indicators
- 10-bit debug bus

## Design Architecture

### Protocol Implementation

The module handles 24-bit transactions that include:
- 10-bit address space
- 3-bit command opcode (010 for Write, 011 for Read)
- 8-bit data payload
- Additional padding bits

### Register Map

The module implements several addressable registers:
```
reg_015  (0x15)  - General purpose register
reg_215  (0x215) - Extended address register
reg_012  (0x12)  - General purpose register
reg_A    (0xA)   - General purpose register
```

### Key Components

#### SPI Interface
```verilog
input  i_SPI_CS,    // Chip Select
input  i_SPI_CLK,   // SPI Clock
input  MOSI_in,     // Master Out Slave In
output MISO_out     // Master In Slave Out
```

#### Shift Register Logic
The heart of the module is a 24-bit shift register that handles incoming data on the MOSI line. It's synchronized to the SPI clock and handles data capture on positive edges:

```verilog
always @(posedge i_SPI_CLK, posedge i_SPI_CS) begin
    if (i_SPI_CS)
        begin
            bits <= 0;
            counter <= 0;
        end
    else
        begin
            bits <= bits << 1;
            bits[0] <= MOSI_in;
            counter <= counter + 1;
        end
end
```

### Debug Features

#### LED Indicators
- Green LED: Switch enabled
- Red LED: Switch disabled

#### Debug Bus
A 10-bit debug bus provides real-time visibility into:
- Free-running counter status
- MISO output state
- Chip select status
- Address matching
- Data validation
- Command decoding

## Interesting Challenges

1. **Clock Domain Crossing**: Handling the transition between SPI clock and FPGA system clock domains required careful consideration of synchronization.

2. **Command Timing**: Ensuring proper timing for read/write operations while maintaining protocol compliance was tricky. The solution involved careful state management around the chip select signal.

3. **Register Access**: Implementing clean register read/write logic while maintaining proper synchronization with the SPI clock required several iterations.

## Future Improvements

- Add FIFO buffers for improved data handling
- Implement error detection
- Add configuration registers for flexible operation
- Enhance debug capabilities with transaction counters
- Add support for different SPI modes

## Lessons Learned

1. Always add more debug signals than you think you'll need
2. Careful state machine design is crucial for reliable operation
3. Simulation saves hours of hardware debugging time
4. Clock domain crossing requires special attention
5. LED indicators are invaluable for basic debugging

## Conclusion

This project provided valuable hands-on experience with both SPI protocol implementation and FPGA development. The modular design approach made debugging easier and will allow for future enhancements. Looking forward to expanding its capabilities in future iterations!

Stay tuned for more FPGA development adventures!
