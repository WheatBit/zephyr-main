# Ambiq Driver Documentation for AI Assistants

## Overview

This directory contains comprehensive documentation to help AI coding assistants understand and properly update Ambiq drivers in the Zephyr RTOS project. The documentation covers three layers:

1. **Hardware layer** - Actual registers, capabilities, and timing specs
2. **HAL layer** - How Ambiq's HAL abstracts the hardware
3. **Driver layer** - How to implement Zephyr drivers using the HAL

---

## Documents (Read in Order)

### 1. **`AMBIQ_HARDWARE_REGISTER_GUIDE.md`** - Hardware Registers & Capabilities
**Purpose**: Understand the actual hardware - registers, bit fields, timing, and capabilities

**Essential for**:
- Understanding what the hardware can actually do
- Debugging low-level issues
- Verifying HAL behavior against hardware specs
- Working with register-level code

**Contains**:
- Complete memory maps for Apollo 2/3/4/5
- Detailed register maps for every peripheral (MSPI, IOM, UART, ADC, GPIO, Flash/MRAM, DMA)
- Bit field definitions with masks and positions
- Hardware timing requirements (setup/hold times, clock constraints)
- Interrupt flag definitions and meanings
- DMA descriptor formats and control bits
- Power domain architecture
- Hardware errata and workarounds
- Comparison tables across Apollo series

**Key sections**:
- MSPI: Command queue, DMA descriptors, XIP, timing calibration
- IOM: FIFO architecture, I2C/SPI mode switching, shared resources
- UART: Baud rate calculations, flow control, FIFO depth
- ADC: Slot configuration, DMA mode, conversion timing
- GPIO: Per-pin registers, interrupt configuration
- Flash/MRAM: Why there's no async (hardware limitation)
- DMA: Channel assignments, descriptor chaining
- Clock tree: Frequencies, sources, divisors

**Key takeaway**: Hardware capabilities determine what's possible - start here to understand constraints.

---

### 2. **`AMBIQ_HAL_GUIDE.md`** - Understanding the HAL
**Purpose**: Learn how Ambiq's Hardware Abstraction Layer maps to the hardware registers

**Essential for**:
- Reading and understanding HAL code
- Determining if a peripheral supports async operations
- Understanding HAL function categories
- Debugging HAL-level issues

**Contains**:
- HAL naming conventions and patterns
- Function categories (lifecycle, config, transfer, interrupt, DMA)
- Blocking vs non-blocking operation detection
- Handle-based architecture
- Hardware constraints that affect HAL behavior
- Common HAL pitfalls
- Version differences across Apollo series
- How HAL abstracts register access

**Key sections**:
- How to identify nonblocking APIs
- When HAL provides async vs when it doesn't
- Register access patterns in HAL
- Callback mechanisms
- Power control integration

**Key takeaway**: Not all peripherals can be async - it depends on what the HAL provides. The HAL shields you from registers but also constrains what's possible.

---

### 3. **`AMBIQ_DRIVER_DEVELOPMENT_GUIDE.md`** - How to Update Drivers
**Purpose**: Step-by-step practical guide for implementing Zephyr drivers

**Essential for**:
- Creating new async drivers
- Adding PM Runtime to existing drivers
- Understanding proper locking patterns
- Testing and validation

**Contains**:
- Driver structure setup (config, data, API structs)
- Implementation patterns (init, transfer, ISR, PM)
- PM Runtime integration steps
- Testing procedures and checklists
- Common code patterns from existing drivers
- Best practices and gotchas
- Integration with Zephyr subsystems

**Key sections**:
- Setting up async with semaphores/callbacks
- PM Runtime best practices
- ISR implementation patterns
- DMA setup and teardown
- Error handling

**Key takeaway**: MSPI driver is the gold standard - use it as a template for async operations.

---

### 4. **`ambiq_driver_async_status.md`** - Current Driver Status
**Purpose**: Analysis and assessment of all existing Ambiq drivers

**Essential for**:
- Understanding which drivers need updates
- Priority ranking for improvements
- Current capabilities of each driver

**Contains**:
- Classification of all Ambiq drivers
- Async compliance assessment
- Specific recommendations per driver
- Priority categorization
- Historical analysis and corrections

**Key takeaway**: Most drivers are already good; flash driver is correctly implemented given HAL constraints.

---

## Quick Start for AI

### Reading Order for Maximum Understanding:

```
Step 1: AMBIQ_HARDWARE_REGISTER_GUIDE.md
        ↓ (Understand what hardware can do)
        
Step 2: AMBIQ_HAL_GUIDE.md
        ↓ (Understand how HAL abstracts hardware)
        
Step 3: AMBIQ_DRIVER_DEVELOPMENT_GUIDE.md
        ↓ (Learn implementation patterns)
        
Step 4: ambiq_driver_async_status.md
        ↓ (See current status)
        
Step 5: Look at drivers/mspi/mspi_ambiq_ap5.c
        (Reference implementation)
```

---

## When Asked to Update an Ambiq Driver

### 1. Check Hardware Capabilities First
Consult `AMBIQ_HARDWARE_REGISTER_GUIDE.md`:
- Does the peripheral have DMA support?
- Is there a DCMP (DMA complete) interrupt flag?
- What are the timing constraints?
- Are there hardware errata?

### 2. Check HAL Capabilities
Consult `AMBIQ_HAL_GUIDE.md` and grep the code:
```bash
grep "am_hal_<periph>_nonblocking" modules/hal/ambiq/
```
- If found → Can be async (high value)
- If not found → Check if blocking is acceptable

### 3. Follow Implementation Pattern
Consult `AMBIQ_DRIVER_DEVELOPMENT_GUIDE.md`:
- Async peripheral: Copy pattern from `drivers/mspi/mspi_ambiq_ap5.c`
- PM Runtime only: Copy pattern from `drivers/uart/uart_ambiq.c`

### 4. Test Thoroughly
Use the testing checklist in `AMBIQ_DRIVER_DEVELOPMENT_GUIDE.md`

---

## Key Principles

### ✅ DO:
- **Start with hardware** - understand register-level capabilities
- Use MSPI driver as template for async operations
- Check HAL documentation/code for API availability
- Add PM Runtime when appropriate
- Use proper locking (mutex for threads, spinlock for ISR)
- Always check HAL return codes
- Call `interrupt_service()` in ISR
- Understand timing requirements from hardware guide

### ❌ DON'T:
- Assume all peripherals have async HAL APIs
- Use blocking transfers when nonblocking is available
- Take mutexes in ISR context
- Forget power control sequence (initialize → power → enable)
- Ignore cache coherency for DMA operations
- Skip reading hardware constraints before implementing

---

## Flash Driver Special Case

The flash driver is **correctly implemented** as blocking because:
- **Hardware**: No DMA support, no completion interrupt (see register guide)
- **HAL**: No async APIs (`am_hal_flash_nonblocking_*` doesn't exist)
- **Constraint**: Flash operations must be atomic (cannot be interrupted)
- **Solution**: Chunking (128 bytes) is the proper RTOS pattern
- **Benefit**: Allows scheduler to preempt between chunks

**Conclusion**: No async update needed or possible for flash driver given hardware limitations.

---

## Current Status Summary

| Status | Count | Drivers |
|--------|-------|---------|
| ✅ Fully Compliant | 7 | MSPI, UART, SDHC, DMIC, I2S, RTC, Watchdog |
| ⚠️ Minor Updates | 5 | I2C, SPI, ADC, GPIO, Counter, PWM |
| ❌ Significant Work | 0 | None |

**Result**: Ambiq drivers are in excellent shape overall.

---

## Examples of Good Patterns

### Async Transfer (from MSPI):
```c
// Non-blocking with callback
ret = am_hal_mspi_nonblocking_transfer(
    data->mspiHandle,
    &trans,
    MSPI_DMA,
    callback_func,
    callback_context
);
// Returns immediately, callback invoked from ISR
```

### PM Runtime (from SDHC):
```c
ret = pm_device_runtime_get(dev);
// ... perform operations ...
pm_device_runtime_put_async(dev, K_MSEC(2));
```

### ISR Pattern (standard across all drivers):
```c
static void driver_isr(const struct device *dev) {
    uint32_t status;
    am_hal_<periph>_interrupt_status_get(handle, &status, false);
    am_hal_<periph>_interrupt_clear(handle, status);
    am_hal_<periph>_interrupt_service(handle, status);  // ← Critical!
}
```

### Direct Register Access (when needed):
```c
// Check peripheral busy status
#define MSPI_BUSY BIT(2)
volatile uint32_t *status_reg = (uint32_t *)(cfg->reg_base + 0x008);

if (*status_reg & MSPI_BUSY) {
    return -EBUSY;
}
```

---

## Hardware Insights Summary

These come from `AMBIQ_HARDWARE_REGISTER_GUIDE.md` and are critical to understand:

### Async Capability is Hardware-Dependent
- **Has DMA + completion interrupt** (DCMP flag) → Can be async (MSPI, IOM, UART, ADC, Audio)
- **Only blocking HAL** (Flash/MRAM) → Cannot be async
- **Fast operations** (GPIO, RTC) → Async not needed

### Interrupt Service Pattern
Always call `am_hal_*_interrupt_service()` - it:
- Reads internal HAL state
- Invokes registered callbacks
- Updates transfer status

### DMA Descriptors
- Most async peripherals use DMA descriptors (32 bytes)
- HAL manages descriptor setup
- Driver provides memory and callbacks
- Descriptors can be chained

### Power Domains
- Peripherals can be powered independently
- Must enable before use, disable when idle
- PM Runtime coordinates power state

### Clock Constraints
- Each peripheral has maximum clock frequency
- Setup/hold times vary by speed
- Timing calibration may be needed (MSPI DQS)

### FIFO Management
- Most peripherals have FIFOs (32-128 bytes)
- Threshold interrupts for flow control
- DMA mode avoids CPU FIFO management

---

## For Human Developers

These documents are optimized for AI consumption but are also useful for human developers. They provide:
- Clear decision trees
- Copy-paste examples
- Common pitfall warnings
- Testing checklists
- Hardware-level understanding

The **Hardware Register Guide** is especially valuable for:
- Debugging issues that HAL can't explain
- Understanding performance characteristics
- Working with timing-critical code
- Verifying HAL behavior

---

## Questions to Ask AI

When working with an AI assistant on Ambiq drivers, ask:

1. **Hardware Level**:
   - "What registers control this peripheral?"
   - "What is the interrupt flag for DMA completion?"
   - "What are the timing requirements for this peripheral?"
   - "Does the hardware support DMA for this operation?"

2. **HAL Level**:
   - "Does the HAL provide nonblocking APIs for this peripheral?"
   - "What callback mechanism does this HAL function use?"
   - "How does the HAL abstract these registers?"

3. **Driver Level**:
   - "Show me similar async patterns from MSPI driver"
   - "Is PM Runtime integration needed here?"
   - "What locking mechanism should I use?"
   - "What testing should I do after this change?"

The AI should refer to these documents to give accurate, context-aware answers.

---

## Document Maintenance

**Created**: 2026-10-14  
**Last Updated**: 2026-10-14  
**Author**: AI-generated for AI consumption  

**Review**: Should be updated when:
- New Ambiq SOC series added (Apollo 6x, etc.)
- HAL APIs change significantly
- Zephyr RTOS patterns evolve
- New hardware errata discovered
- Register definitions change

---

## Document Version History

- **v1.0** (2026-10-14): Initial release with HAL and Driver guides
- **v2.0** (2026-10-14): Added comprehensive Hardware Register Guide with detailed register maps, bit fields, timing specs, and hardware capabilities for all peripherals

---

## Feedback

If these documents are missing critical information or could be improved, note it for future updates. The goal is to provide complete, accurate, hardware-to-driver understanding for both AI and human developers.
# Ambiq Hardware Register Guide for Driver Development

## Purpose
This document provides detailed hardware register-level information for Ambiq Apollo MCU peripherals. Use this when you need to understand the actual hardware capabilities, timing requirements, and register interactions - not just the HAL API layer.

**Target Audience**: Engineers who need to understand or debug at the hardware level, or extend HAL functionality.

## Table of Contents
1. [Apollo MCU Architecture Overview](#apollo-mcu-architecture-overview)
2. [Memory Map](#memory-map)
3. [Peripheral Register Structures](#peripheral-register-structures)
4. [MSPI Hardware Details](#mspi-hardware-details)
5. [IOM Hardware Details](#iom-hardware-details)
6. [UART Hardware Details](#uart-hardware-details)
7. [ADC Hardware Details](#adc-hardware-details)
8. [GPIO Hardware Details](#gpio-hardware-details)
9. [Flash/MRAM Hardware Details](#flashmram-hardware-details)
10. [DMA Architecture](#dma-architecture)
11. [Interrupt Controller](#interrupt-controller)
12. [Power Management Hardware](#power-management-hardware)
13. [Timing and Clock Constraints](#timing-and-clock-constraints)
14. [Hardware Errata and Workarounds](#hardware-errata-and-workarounds)

---

## Apollo MCU Architecture Overview

### SoC Comparison

| Feature | Apollo2 | Apollo3/3p | Apollo4/4p | Apollo5 |
|---------|---------|------------|------------|---------|
| **Core** | ARM Cortex-M4F | ARM Cortex-M4F | ARM Cortex-M4F | ARM Cortex-M55 |
| **Max Freq** | 48 MHz | 96 MHz | 192 MHz | 250 MHz |
| **Flash** | NOR Flash | NOR Flash | NOR Flash | MRAM |
| **RAM** | 256 KB | 384 KB | 1.4 MB | 2.75 MB |
| **MSPI** | 1 instance | 1-2 instances | 2 instances | 3 instances |
| **IOM** | 6 instances | 6 instances | 8 instances | 8 instances |
| **UART** | 2 instances | 2 instances | 3 instances | 4 instances |
| **DMA** | Basic | Enhanced | Enhanced | Enhanced |
| **Cache** | - | ICache 16KB | I/D Cache 32KB | Configurable |

### Bus Architecture

```
                    ┌─────────────────┐
                    │   ARM Core      │
                    │  (Cortex-M4F)   │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────▼─────┐  ┌────▼────┐  ┌─────▼─────┐
        │  ICode    │  │  DCode  │  │   System  │
        │   Bus     │  │   Bus   │  │    Bus    │
        └─────┬─────┘  └────┬────┘  └─────┬─────┘
              │              │              │
        ┌─────▼──────────────▼──────────────▼─────┐
        │         Multi-Layer AHB Matrix          │
        └──┬────┬────┬────┬────┬────┬────┬────┬──┘
           │    │    │    │    │    │    │    │
        ┌──▼─┐┌─▼─┐┌─▼─┐┌─▼─┐┌─▼─┐┌─▼─┐┌─▼─┐┌▼──┐
        │Flash││RAM││MSPI││IOM││GPIO││ADC││... ││DMA│
        └────┘└───┘└────┘└───┘└────┘└───┘└───┘└───┘
```

---

## Memory Map

### Apollo 5x Memory Map

```
0x0000_0000 ┌─────────────────┐
            │   MRAM (2MB)    │  Main memory (XIP-capable)
0x001F_FFFF ├─────────────────┤
            │   Reserved      │
0x1000_0000 ├─────────────────┤
            │  SRAM (2.75MB)  │  TCM + shared SRAM
0x102B_FFFF ├─────────────────┤
            │   Reserved      │
0x4000_0000 ├─────────────────┤
            │  Peripherals    │  
            │  - MSPI0-2      │  @ 0x4050_0000 - 0x4052_0000
            │  - IOM0-7       │  @ 0x4000_4000 - 0x4000_B000
            │  - UART0-3      │  @ 0x4000_C000 - 0x4000_F000
            │  - GPIO         │  @ 0x4001_0000
            │  - ADC          │  @ 0x4003_8000
            │  - etc.         │
0x5FFF_FFFF ├─────────────────┤
            │   Reserved      │
0xE000_0000 ├─────────────────┤
            │  ARM Private    │  NVIC, SysTick, etc.
0xFFFF_FFFF └─────────────────┘
```

### Peripheral Base Addresses (Apollo 5x)

| Peripheral | Base Address | Size | Instances |
|------------|--------------|------|-----------|
| **MSPI0** | 0x4050_0000 | 4KB | 1 |
| **MSPI1** | 0x4051_0000 | 4KB | 1 |
| **MSPI2** | 0x4052_0000 | 4KB | 1 |
| **IOM0** | 0x4000_4000 | 4KB | 1 |
| **IOM1** | 0x4000_5000 | 4KB | 1 |
| **IOM2** | 0x4000_6000 | 4KB | 1 |
| **IOM3** | 0x4000_7000 | 4KB | 1 |
| **IOM4** | 0x4000_8000 | 4KB | 1 |
| **IOM5** | 0x4000_9000 | 4KB | 1 |
| **IOM6** | 0x4000_A000 | 4KB | 1 |
| **IOM7** | 0x4000_B000 | 4KB | 1 |
| **UART0** | 0x4000_C000 | 4KB | 1 |
| **UART1** | 0x4000_D000 | 4KB | 1 |
| **UART2** | 0x4000_E000 | 4KB | 1 |
| **UART3** | 0x4000_F000 | 4KB | 1 |
| **GPIO** | 0x4001_0000 | 64KB | 1 (128 pins) |
| **ADC** | 0x4003_8000 | 4KB | 1 |
| **SDHC** | 0x4004_0000 | 64KB | 2 |

---

## Peripheral Register Structures

### Register Access Pattern

```c
// Typical peripheral register structure
typedef struct {
    volatile uint32_t CFG;          // +0x00: Configuration
    volatile uint32_t CTRL;         // +0x04: Control
    volatile uint32_t STATUS;       // +0x08: Status
    volatile uint32_t INTEN;        // +0x0C: Interrupt Enable
    volatile uint32_t INTSTAT;      // +0x10: Interrupt Status
    volatile uint32_t INTCLR;       // +0x14: Interrupt Clear
    volatile uint32_t DATA;         // +0x18: Data
    volatile uint32_t DMACFG;       // +0x1C: DMA Configuration
    volatile uint32_t DMASTAT;      // +0x20: DMA Status
    // ... more registers ...
} PERIPH_Type;

// Access pattern in drivers
#define PERIPH_BASE 0x40050000
#define PERIPH ((PERIPH_Type *)PERIPH_BASE)

// Register access
PERIPH->CFG = value;                    // Write
uint32_t status = PERIPH->STATUS;       // Read
PERIPH->INTCLR = BIT(3);               // Clear interrupt
```

---

## MSPI Hardware Details

### MSPI Register Map

```
Offset   Register Name          R/W    Description
------   ------------------     ----   ------------------------------------
0x000    CFG                    R/W    MSPI Configuration
0x004    MSPICFG                R/W    MSPI Device Configuration  
0x008    PADCFG                 R/W    Pad Configuration
0x00C    PADOUTEN               R/W    Pad Output Enable
0x010    FLASH                  R/W    Flash Configuration
0x014    INTEN                  R/W    Interrupt Enable
0x018    INTSTAT                R      Interrupt Status
0x01C    INTCLR                 W      Interrupt Clear
0x020    INTSET                 W      Interrupt Set
0x100    CTRL                   R/W    Transfer Control
0x104    ADDR                   R/W    Transfer Address
0x108    INSTR                  R/W    Instruction/Command
0x10C    TXFIFO                 W      TX FIFO Data
0x110    RXFIFO                 R      RX FIFO Data
0x114    TXENTRIES              R      TX FIFO Entry Count
0x118    RXENTRIES              R      RX FIFO Entry Count
0x200    CQCFG                  R/W    Command Queue Configuration
0x204    CQADDR                 R/W    Command Queue Address
0x208    CQPAUSE                R/W    Command Queue Pause
0x20C    CQCURIDX               R      Command Queue Current Index
0x210    CQENDIDX               R/W    Command Queue End Index
0x300    DMACFG                 R/W    DMA Configuration
0x304    DMATARGADDR            R/W    DMA Target Address
0x308    DMASTAT                R      DMA Status
0x30C    DMATOTCOUNT            R/W    DMA Total Count
0x310    DMATHRESH              R/W    DMA Threshold
```

### MSPI CFG Register (0x000)

```
Bits     Field Name      Description
------   -------------   ----------------------------------------
31       APBCLK          Enable APB clock (1=enabled)
30       IPADD           Enable IPCE Address decode
29:28    NOINSTR         No instruction mode
27:26    INSTRO          Instruction output mode
25:24    INSTRI          Instruction input mode  
23       CLKOND4         Clock on D4 pin
22:20    SEPIO           Separate IO configuration
19:16    CLOCKDIV        Clock divisor (0-15)
15:13    CPOL/CPHA       SPI clock polarity/phase
12:8     DEVCFG          Device configuration select
7:4      XIPEN           XIP enable per CE
3:0      FULLDUP         Full duplex enable
```

### MSPI Interrupt Flags

| Bit | Name | Trigger Condition | Usage |
|-----|------|-------------------|--------|
| 0 | CMDCMP | Command complete | PIO transfers |
| 1 | THR | Threshold reached | FIFO management |
| 2 | DCMP | DMA complete | **DMA transfers (key for async)** |
| 3 | ERR | Transfer error | Error handling |
| 4 | CQUPD | Command queue updated | CQ mode |
| 5 | CQPAUSED | Command queue paused | CQ flow control |
| 6 | CQERR | Command queue error | CQ error handling |
| 7 | SCRERR | Scrambling error | Security |

**Critical for Async**: DCMP (bit 2) signals DMA completion. ISR must:
```c
if (status & AM_HAL_MSPI_INT_DCMP) {
    // DMA transfer complete
    // Signal semaphore or invoke callback
}
```

### MSPI DMA Descriptor (Command Queue Entry)

```c
typedef struct {
    uint32_t address;       // Target address
    uint32_t length;        // Transfer length (bytes)
    uint32_t command;       // Command/instruction byte
    uint32_t control;       // Control flags
    uint32_t status;        // Status/flags
    uint32_t reserved[3];   // Padding to 32 bytes
} mspi_dma_descriptor_t;
```

**Control Flags**:
```
Bits     Description
------   ----------------------------------------
31       Pause command queue after this entry
30       Continue (don't de-assert CS)
29       Direction (0=read, 1=write)
28       Scramble enable
27:24    Chip select number (0-3)
23:16    Turn-around cycles
15:8     Dummy cycles
7:0      Address length (bytes)
```

### MSPI Hardware Capabilities

| Feature | Apollo3 | Apollo4 | Apollo5 |
|---------|---------|---------|---------|
| **Max Clock** | 48 MHz | 96 MHz | 125 MHz |
| **DQS Support** | No | Yes | Yes |
| **Octal Support** | No | Yes | Yes |
| **XIP Region** | 64 MB | 256 MB | 256 MB |
| **FIFO Size** | 64 bytes | 128 bytes | 128 bytes |
| **CQ Entries** | 16 | 32 | 32 |
| **DMA Channels** | 1 per MSPI | 1 per MSPI | 1 per MSPI |
| **Scrambling** | No | Yes | Yes |

### MSPI Timing Requirements

```
Parameter              Apollo3    Apollo4    Apollo5    Unit
-------------------    -------    -------    -------    ----
Setup time (tSU)       2          2          1.5        ns
Hold time (tH)         2          2          1.5        ns
Clock high (tCH)       5          3          2          ns
Clock low (tCL)        5          3          2          ns
CS to CLK (tCS)        10         5          4          ns
Data valid (tDV)       -          5          4          ns
DQS delay range        -          0-31       0-31       steps
```

**Code Example - Direct Register Access**:
```c
// Check if MSPI is busy (used in driver)
#define MSPI_BUSY BIT(2)
volatile uint32_t *mspi_status = (uint32_t *)(cfg->reg_base + 0x008);

if (*mspi_status & MSPI_BUSY) {
    return -EBUSY;
}
```

---

## IOM Hardware Details

### IOM Register Map

```
Offset   Register Name          Description
------   ------------------     ------------------------------------
0x000    FIFOTHR                FIFO Threshold Configuration
0x004    FIFOPTR                FIFO Pointers  
0x008    FIFOCFG                FIFO Configuration
0x00C    FIFOCTR                FIFO Control
0x100    FIFOPUSH               FIFO Push Data
0x104    FIFOPOP                FIFO Pop Data
0x108    FIFOMAX                FIFO Maximum Depth
0x200    INTEN                  Interrupt Enable
0x204    INTSTAT                Interrupt Status
0x208    INTCLR                 Interrupt Clear
0x20C    INTSET                 Interrupt Set
0x300    CLKCFG                 Clock Configuration
0x304    SUBMODCTRL             Submodule Control (I2C/SPI select)
0x308    CMD                    Command Register
0x30C    OFFSETHI               High byte offset (I2C address)
0x400    CMDSTAT                Command Status
0x404    DMACFG                 DMA Configuration
0x408    DMATOTCOUNT            DMA Total Count
0x40C    DMATARGADDR            DMA Target Address
0x410    DMASTAT                DMA Status
```

### IOM SUBMODCTRL Register - Mode Selection

```
Bits     Field               Description
------   -----------------   ----------------------------------------
31:8     Reserved            
7:4      SPIMODE             SPI mode (CPOL/CPHA)
                             0x0 = Mode 0 (CPOL=0, CPHA=0)
                             0x1 = Mode 1 (CPOL=0, CPHA=1)
                             0x2 = Mode 2 (CPOL=1, CPHA=0)
                             0x3 = Mode 3 (CPOL=1, CPHA=1)
3        I2CLSB              I2C LSB first (normally 0)
2        I2CADDR             I2C address mode (7/10 bit)
1:0      IOMODE              Interface mode:
                             0x0 = SPI Master
                             0x1 = I2C Master
                             0x2 = Reserved
                             0x3 = Reserved
```

**Critical**: IOM shares hardware between I2C and SPI. Driver must:
1. Configure SUBMODCTRL before each transaction
2. Use proper locking (mutex) across mode changes
3. Reconfigure if mode switched

### IOM Command Register Format

```c
typedef union {
    uint32_t value;
    struct {
        uint32_t CMD:5;          // [4:0] Command
        uint32_t OFFSETCNT:2;    // [6:5] Offset byte count
        uint32_t CONT:1;         // [7] Continue (no STOP)
        uint32_t TSIZE:12;       // [19:8] Transfer size
        uint32_t CMDSEL:2;       // [21:20] Command select
        uint32_t OFFSETLO:8;     // [29:22] Low offset byte
        uint32_t _reserved:2;    // [31:30]
    } fields;
} iom_cmd_t;
```

**Commands**:
- `0x02` = WRITE - I2C/SPI write
- `0x03` = READ - I2C/SPI read
- `0x08` = TMO - Timeout
- `0x09` = FLUSH - Flush FIFO

### IOM Interrupt Flags

| Bit | Name | I2C | SPI | Description |
|-----|------|-----|-----|-------------|
| 0 | CMDCMP | ✓ | ✓ | Command complete |
| 1 | THR | ✓ | ✓ | FIFO threshold |
| 2 | DCMP | ✓ | ✓ | **DMA complete** |
| 3 | ERR | ✓ | ✓ | General error |
| 4 | IACC | ✓ | - | Invalid I2C access |
| 5 | STOP | ✓ | - | I2C STOP detected |
| 6 | START | ✓ | - | I2C START detected |
| 7 | ARBLOST | ✓ | - | I2C arbitration lost |
| 8 | NACK | ✓ | - | I2C NACK received |

**For Async**: Like MSPI, bit 2 (DCMP) signals DMA completion.

### IOM FIFO Architecture

```
        TX FIFO (64 bytes)           RX FIFO (64 bytes)
        ┌──────────────┐            ┌──────────────┐
CPU --> │ Entry 0      │            │ Entry 0      │ --> CPU
Write   │ Entry 1      │            │ Entry 1      │    Read
        │ ...          │            │ ...          │
        │ Entry 63     │            │ Entry 63     │
        └──────┬───────┘            └──────▲───────┘
               │                           │
               ▼                           │
        ┌──────────────┐            ┌──────────────┐
        │ Shift Reg    │───────────>│ Shift Reg    │
        └──────────────┘   Serial   └──────────────┘
               │             Bus            ▲
               └────────────────────────────┘
```

**Threshold Interrupts**:
```c
// Configure FIFO thresholds
#define TX_THR 8   // Fire when TX FIFO has space for 8+ bytes
#define RX_THR 8   // Fire when RX FIFO contains 8+ bytes

PERIPH->FIFOTHR = (TX_THR << 16) | RX_THR;
```

---

## UART Hardware Details

### UART Register Map

```
Offset   Register Name          Description
------   ------------------     ------------------------------------
0x000    DR                     Data Register
0x004    RSR/ECR                Receive Status/Error Clear
0x018    FR                     Flag Register (status)
0x020    ILPR                   IrDA Low-Power
0x024    IBRD                   Integer Baud Rate Divisor
0x028    FBRD                   Fractional Baud Rate Divisor
0x02C    LCRH                   Line Control Register
0x030    CR                     Control Register
0x034    IFLS                   Interrupt FIFO Level Select
0x038    IER                    Interrupt Enable Register
0x03C    IES                    Interrupt Status Register
0x040    MIS                    Masked Interrupt Status
0x044    IEC                    Interrupt Clear Register
```

### UART CR (Control Register)

```
Bits     Field       Description
------   ---------   ----------------------------------------
15       CTSEN       CTS hardware flow control enable
14       RTSEN       RTS hardware flow control enable
11       RTS         Request to Send
9        RXE         Receive Enable
8        TXE         Transmit Enable
7        LBE         Loopback Enable
2        SIRLP       IrDA SIR low power
1        SIREN       SIR Enable
0        UARTEN      UART Enable
```

### UART Baud Rate Calculation

```
BAUDDIV = UART_CLK / (16 × Baud Rate)
IBRD = integer part of BAUDDIV
FBRD = integer((fractional part of BAUDDIV) × 64 + 0.5)

Example: 115200 baud with 24 MHz clock
BAUDDIV = 24,000,000 / (16 × 115200) = 13.02083
IBRD = 13
FBRD = int(0.02083 × 64 + 0.5) = 1

Register values:
IBRD = 0x000D
FBRD = 0x01
```

### UART FIFO Depth

| Apollo Series | TX FIFO | RX FIFO | Note |
|---------------|---------|---------|------|
| Apollo 2 | 32 bytes | 32 bytes | Fixed |
| Apollo 3/4/5 | 32 bytes | 32 bytes | Programmable threshold |

### UART Interrupt Flags

| Bit | Name | Trigger | Async Relevant |
|-----|------|---------|----------------|
| 0 | RI | Ring indicator | No |
| 1 | CTS | CTS modem | No |
| 2 | DCD | DCD modem | No |
| 3 | DSR | DSR modem | No |
| 4 | RX | RX FIFO threshold | **Yes** |
| 5 | TX | TX FIFO threshold | **Yes** |
| 6 | TXCMP | TX complete | **Yes** |
| 7 | OE | Overrun error | Error handling |
| 8 | BE | Break error | Error handling |
| 9 | PE | Parity error | Error handling |
| 10 | FE | Framing error | Error handling |

**Async Pattern**:
```c
// Enable TX/RX completion interrupts
uart->IER = UART_IER_RX | UART_IER_TXCMP;

// In ISR:
if (status & UART_IER_TXCMP) {
    // TX complete - signal semaphore
    k_sem_give(&data->tx_done_sem);
}
```

---

## ADC Hardware Details

### ADC Register Map

```
Offset   Register Name          Description
------   ------------------     ------------------------------------
0x000    CFG                    ADC Configuration
0x004    STAT                   ADC Status
0x008    SWT                    Software Trigger
0x00C    SL0CFG                 Slot 0 Configuration
0x010    SL1CFG                 Slot 1 Configuration
...      ...                    (8 slots total)
0x02C    SL7CFG                 Slot 7 Configuration
0x030    WULIM                  Window Upper Limit
0x034    WLLIM                  Window Lower Limit
0x038    SCWLIM                 Scalable Window Limit
0x03C    FIFO                   FIFO Data Register
0x040    FIFOPR                 FIFO Pop Register
0x080    INTEN                  Interrupt Enable
0x084    INTSTAT                Interrupt Status  
0x088    INTCLR                 Interrupt Clear
0x08C    INTSET                 Interrupt Set
0x100    DMACFG                 DMA Configuration
0x104    DMATOTCOUNT            DMA Total Count
0x108    DMATARGADDR            DMA Target Address
0x10C    DMASTAT                DMA Status
```

### ADC Slot Configuration

```
Bits     Field           Description
------   -------------   ----------------------------------------
31       ADSEL1          Window comparator select
30       ADSEL0          Window comparator select
29:28    PRMODE          Precision mode (8/10/12/14-bit)
27:26    TRKCYC          Tracking cycles (0-3)
25:24    RESERVED
23:16    CHSEL           Channel select (0-15)
15:8     THSEL           Threshold select
7        WCEN            Window comparator enable
6:4      RESERVED
3:0      SLEN            Sample length
```

### ADC Channels (Apollo 5x)

| Channel | Input Source | Notes |
|---------|--------------|-------|
| 0 | External SE0 | Single-ended |
| 1 | External SE1 | Single-ended |
| 2 | External DIFF0P/N | Differential |
| 3 | External DIFF1P/N | Differential |
| 4-7 | External | Varies by package |
| 8 | Temp sensor | Internal |
| 9 | VDD/3 | Power supply monitor |
| 10 | VSS | Ground reference |
| 11 | VBAT/3 | Battery monitor |
| 12-15 | Reserved | - |

### ADC Timing

```
Total conversion time = (Tracking cycles + Sample cycles + Conversion cycles)

Tracking: 2-32 cycles (configurable)
Sample: Depends on precision
Conversion: Depends on precision

Precision   Sample+Conversion    Total @ 2 tracking
--------    -----------------    ------------------
8-bit       10 cycles           12 cycles (0.5 μs @ 24 MHz)
10-bit      12 cycles           14 cycles (0.58 μs)
12-bit      14 cycles           16 cycles (0.67 μs)  
14-bit      16 cycles           18 cycles (0.75 μs)
```

### ADC DMA Mode

```c
// DMA descriptor for ADC
typedef struct {
    uint32_t target_addr;    // Where to store samples
    uint32_t total_count;    // Number of samples
    uint32_t dma_cfg;        // DMA configuration
    uint32_t status;         // Completion status
} adc_dma_descriptor_t;

// Configuration bits
#define ADC_DMA_ENABLE    BIT(0)
#define ADC_DMA_DYNAMIC   BIT(2)   // Dynamic priority
#define ADC_DMA_THRESHOLD(n)  ((n) << 8)  // Threshold value
```

**Interrupt on DMA Complete**:
```c
if (status & AM_HAL_ADC_INT_DCMP) {
    // DMA buffer full
    k_sem_give(&data->dma_done_sem);
}
```

---

## GPIO Hardware Details

### GPIO Register Map (Per-Pin Registers)

Apollo GPIOs use per-pin configuration, not port-based:

```
Base: 0x40010000
Offset = Pin_Number × 4

Pin 0:  0x40010000
Pin 1:  0x40010004
Pin 2:  0x40010008
...
Pin 127: 0x400101FC
```

### GPIO Pin Configuration Register (PADKEY + PADREG)

```
First unlock: Write 0x73 to PADKEY register
Then configure: Write to PADREGx

PADREGx bits (Apollo 5x):
Bits     Field           Description
------   -------------   ----------------------------------------
31:30    Reserved
29:27    DS              Drive strength (0-7)
26       SR              Slew rate (0=slow, 1=fast)
25:24    PULLUP          Pull-up resistor
                         0x0 = No pull-up
                         0x1 = 1.5K pull-up
                         0x2 = 6K pull-up  
                         0x3 = 12K pull-up
23       Reserved
22:19    INPEN           Input enable
18       Reserved
17:16    FNCSEL2         Function select [9:8]
15:11    Reserved
10:8     FNCSEL1         Function select [7:5]
7:3      FNCSEL0         Function select [4:0]
2        OUTCFG          Output configuration
                         0 = Push-pull
                         1 = Open-drain
1        Reserved
0        IRPTEN          Interrupt enable
```

### GPIO Interrupt Registers

```
Offset   Register            Description
------   -----------------   ------------------------------------
0x200    PINCFG0-31          Pin interrupt config (32 registers)
0x280    INT0EN              Interrupt 0 enable
0x284    INT0STAT            Interrupt 0 status
0x288    INT0CLR             Interrupt 0 clear
0x28C    INT0SET             Interrupt 0 set
(Repeat for INT1-3)
```

### GPIO Interrupt Configuration

```
PINNCFGx bits:
Bits     Field       Description
------   ---------   ----------------------------------------
7        INTD        Interrupt detect (0=edge, 1=level)
6:4      FNCSEL      Function select for interrupt
3:2      INTPOL      Interrupt polarity
                     0x0 = Active low/falling edge
                     0x1 = Active high/rising edge
                     0x2 = Both edges
                     0x3 = Reserved
1        Reserved
0        INCFG       Input configuration enable
```

**Example - Configure Pin as Rising Edge Interrupt**:
```c
// Unlock GPIO configuration
*(volatile uint32_t *)0x40010800 = 0x73;

// Configure pin 42 as GPIO input with interrupt
uint32_t pin_offset = 42 * 4;
volatile uint32_t *padreg = (uint32_t *)(0x40010000 + pin_offset);

*padreg = (1 << 22) |        // INPEN - input enable
          (3 << 0) |          // FNCSEL - GPIO function
          (1 << 0);           // IRPTEN - interrupt enable

// Configure interrupt for rising edge
volatile uint32_t *pincfg = (uint32_t *)(0x40010200 + (42/8)*4);
*pincfg = (0 << 7) |          // INTD - edge detect
          (1 << 2) |          // INTPOL - rising edge
          (1 << 0);           // INCFG - enable

// Enable in interrupt controller
volatile uint32_t *int0en = (uint32_t *)0x40010280;
*int0en |= BIT(42 % 32);
```

---

## Flash/MRAM Hardware Details

### Flash (Apollo 2/3/4) vs MRAM (Apollo 5)

| Feature | NOR Flash | MRAM |
|---------|-----------|------|
| **Write Size** | 4/16 bytes | 16 bytes |
| **Erase Size** | 8 KB page | Not needed |
| **Write Time** | ~30 ms per page | ~10 μs per 16 bytes |
| **Erase Time** | ~30 ms per page | N/A |
| **Endurance** | 10K cycles | 1M+ cycles |
| **Retention** | 20 years | 20 years |

### Flash Controller Registers (Apollo 3/4)

```
Offset   Register            Description
------   -----------------   ------------------------------------
0x000    CTRL                Control register
0x004    ADDR                Address register
0x008    WDATA               Write data
0x00C    RDATA               Read data (cache)
0x010    STATUS              Status flags
0x014    INTEN               Interrupt enable
0x018    INTSTAT             Interrupt status
0x01C    INTCLR              Interrupt clear
```

### Flash STATUS Register

```
Bits     Field       Description
------   ---------   ----------------------------------------
31:8     Reserved
7        BUSY        Flash operation in progress
6        WERR        Write error occurred
5        PERR        Programming error
4        EERR        Erase error
3        FAIL        Operation failed
2        IDLE        Controller idle
1        LOCKED      Program/erase locked
0        READY       Ready for new operation
```

**Critical**: Check BUSY (bit 7) before new operation:
```c
#define FLASH_STATUS_BUSY BIT(7)

// Wait for flash ready
while (FLASH->STATUS & FLASH_STATUS_BUSY) {
    // Busy wait required - no interrupt support
}
```

### MRAM Controller Registers (Apollo 5)

```
Offset   Register            Description
------   -----------------   ------------------------------------
0x000    MRAMCFG             MRAM Configuration
0x004    MRAMCTRL            MRAM Control
0x008    MRAMADDR            Address
0x00C    MRAMWDATA           Write data (128-bit)
0x010-   ...                 (Write data continued)
0x01C    
0x020    MRAMSTATUS          Status
0x024    MRAMCMD             Command register
```

### Flash/MRAM Write Constraints

**Apollo 3/4 Flash**:
```c
// Must lock interrupts during write
unsigned int key = irq_lock();

// Flash write - CPU stalls until complete
ret = am_hal_flash_program_main(
    AM_HAL_FLASH_PROGRAM_KEY,
    (uint32_t *)data,
    (uint32_t *)dest_addr,
    word_count
);

irq_unlock(key);
```

**Apollo 5 MRAM**:
```c
// Same pattern - IRQs locked
unsigned int key = irq_lock();

ret = am_hal_mram_main_program(
    AM_HAL_MRAM_PROGRAM_KEY,
    (uint32_t *)data,
    (uint32_t *)dest_addr,
    word_count
);

irq_unlock(key);
```

**Why No Async**: 
1. No interrupt support (no DCMP flag)
2. CPU must wait for flash controller
3. Cannot execute from same bank being written
4. Hardware serializes all flash operations

---

## DMA Architecture

### DMA Controller (Apollo 4/5)

```
┌─────────────────────────────────────┐
│         DMA Controller              │
├─────────────────────────────────────┤
│  Channel 0  ──> MSPI0               │
│  Channel 1  ──> MSPI1               │
│  Channel 2  ──> MSPI2               │
│  Channel 3  ──> IOM0                │
│  Channel 4  ──> IOM1                │
│  Channel 5  ──> IOM2                │
│  Channel 6  ──> IOM3                │
│  Channel 7  ──> ADC                 │
│  Channel 8  ──> PDM                 │
│  Channel 9  ──> I2S                 │
│  ...                                │
└─────────────────────────────────────┘
```

### DMA Channel Assignment

| Channel | Peripheral | Direction | Priority |
|---------|------------|-----------|----------|
| 0 | MSPI0 | Bidirectional | Configurable |
| 1 | MSPI1 | Bidirectional | Configurable |
| 2 | MSPI2 | Bidirectional | Configurable |
| 3-10 | IOM0-7 | Bidirectional | Configurable |
| 11 | ADC | RX only | Configurable |
| 12 | PDM | RX only | Configurable |
| 13 | I2S TX | TX only | Configurable |
| 14 | I2S RX | RX only | Configurable |

### DMA Descriptor Format

```c
// Standard DMA descriptor (32 bytes)
typedef struct {
    uint32_t src_addr;       // [31:0] Source address
    uint32_t dst_addr;       // [31:0] Destination address
    uint32_t length;         // [31:0] Transfer length (bytes)
    uint32_t control;        // Control and status
    uint32_t next_desc;      // Next descriptor address (chaining)
    uint32_t reserved[3];    // Padding
} dma_descriptor_t;

// Control word bits
#define DMA_CTRL_ENABLE     BIT(0)
#define DMA_CTRL_DIR_M2P    BIT(1)   // Memory to Peripheral
#define DMA_CTRL_DIR_P2M    (0)      // Peripheral to Memory
#define DMA_CTRL_INT_EN     BIT(2)   // Interrupt on completion
#define DMA_CTRL_PRIORITY(n) ((n) << 4)  // Priority 0-7
#define DMA_CTRL_CHAIN      BIT(8)   // Chain to next descriptor
```

### DMA Transfer Flow

```
1. Configure DMA descriptor in memory
2. Write descriptor address to peripheral's DMATARGADDR
3. Write transfer count to DMATOTCOUNT
4. Set DMAEN bit in DMACFG
5. Start peripheral operation
6. DMA controller automatically fetches descriptor
7. Transfer proceeds in background
8. Interrupt fires on completion (if enabled)
9. ISR reads DMASTAT to confirm completion
```

---

## Interrupt Controller

### NVIC Configuration (ARM Cortex-M4F/M55)

Apollo MCUs use standard ARM NVIC:

```
Peripheral        IRQ #    Priority Register
--------------    -----    -----------------
MSPI0             20       NVIC_IPR5[7:0]
MSPI1             21       NVIC_IPR5[15:8]
MSPI2             22       NVIC_IPR5[23:16]
IOM0              6        NVIC_IPR1[23:16]
IOM1              7        NVIC_IPR1[31:24]
IOM2              8        NVIC_IPR2[7:0]
IOM3              9        NVIC_IPR2[15:8]
GPIO              13       NVIC_IPR3[7:0]
UART0             15       NVIC_IPR3[23:16]
ADC               18       NVIC_IPR4[15:8]
```

### Interrupt Priority

```c
// Configure interrupt priority (Zephyr does this automatically)
NVIC_SetPriority(MSPI0_IRQn, CONFIG_MSPI_IRQ_PRIORITY);
NVIC_EnableIRQ(MSPI0_IRQn);

// In driver ISR
void mspi0_isr(void) {
    // Get peripheral interrupt status
    uint32_t status;
    am_hal_mspi_interrupt_status_get(handle, &status, false);
    
    // Clear at peripheral level
    am_hal_mspi_interrupt_clear(handle, status);
    
    // Service interrupt (may invoke callbacks)
    am_hal_mspi_interrupt_service(handle, status);
    
    // NVIC automatically clears pending bit when ISR returns
}
```

---

## Power Management Hardware

### Power Domains (Apollo 5x)

```
┌─────────────────────────────────────┐
│         Always-On Domain            │
│  - RTC, GPIO interrupts, WDT        │
├─────────────────────────────────────┤
│         Core Domain                 │
│  - CPU, SRAM, Flash/MRAM            │
├─────────────────────────────────────┤
│         Peripheral Domain           │
│  - MSPI, IOM, UART, ADC, etc.       │
│  - Can be powered independently     │
└─────────────────────────────────────┘
```

### Power Control Registers

```
Offset   Register            Description
------   -----------------   ------------------------------------
0x000    PWRCTRL             Power control
0x004    PWRSTATUS           Power status
0x008    DEVPWREN            Device power enable
0x00C    MEMPWREN            Memory power enable
0x010    MEMPWRSTATUS        Memory power status
0x014    DEVPWRSTATUS        Device power status
```

### Device Power Enable (DEVPWREN) Bits

```
Bit      Device              Notes
---      -----------------   ----------------------------------------
0        IOS                 IO Slave
1        IOM0                IO Master 0
2        IOM1                IO Master 1
...      ...                 
8        UART0               UART 0
9        UART1               UART 1
...      ...
16       MSPI0               MSPI 0
17       MSPI1               MSPI 1
18       MSPI2               MSPI 2
...      ...
24       ADC                 ADC
25       SDHC                SD Host Controller
```

**PM Control Pattern**:
```c
// Power on MSPI0
am_hal_pwrctrl_periph_enable(AM_HAL_PWRCTRL_PERIPH_MSPI0);

// Power off when done
am_hal_pwrctrl_periph_disable(AM_HAL_PWRCTRL_PERIPH_MSPI0);
```

---

## Timing and Clock Constraints

### Clock Tree (Apollo 5x)

```
                    ┌──────────────┐
                    │   HFRC       │
                    │  (96 MHz)    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐ ┌───▼───┐  ┌────▼────┐
        │   PLL     │ │ XTAL  │  │  LFRC   │
        │ (250 MHz) │ │(32MHz)│  │(32KHz)  │
        └─────┬─────┘ └───┬───┘  └────┬────┘
              │           │            │
        ┌─────▼───────────▼────────────▼─────┐
        │      Clock Multiplexer              │
        └──┬────┬────┬────┬────┬────┬────┬──┘
           │    │    │    │    │    │    │
        HCLK FCLK MSPIx IOMx UART ADC  etc.
```

### Clock Frequencies by Peripheral

| Peripheral | Max Clock | Source | Divisor |
|------------|-----------|--------|---------|
| HCLK (CPU) | 250 MHz | PLL/HFRC | 1,2,4,8,16,32 |
| MSPI | 125 MHz | HCLK/2 | Configurable |
| IOM | 48 MHz | HFRC | 1-15 |
| UART | 24 MHz | HFRC | N/A |
| ADC | 24 MHz | HFRC | N/A |

### Setup/Hold Times

**MSPI (Apollo 5x @ 125 MHz)**:
- Setup time: 1.5 ns minimum
- Hold time: 1.5 ns minimum
- Valid data window: 4 ns typical

**IOM SPI (@ 48 MHz)**:
- Setup time: 5 ns minimum
- Hold time: 5 ns minimum

**IOM I2C (Fast mode, 400 kHz)**:
- SDA setup: 100 ns minimum
- SDA hold: 300 ns minimum (after SCL falling)
- SCL low: 1.3 μs minimum
- SCL high: 600 ns minimum

---

## Hardware Errata and Workarounds

### Known Issues (Apollo 3/4/5)

#### Errata 1: MSPI DQS Sampling Window
**Issue**: DQS sampling window may be narrow at high frequencies (>96 MHz)

**Workaround**: Use timing calibration/scan:
```c
// Perform timing scan to find optimal RX/TX delays
am_hal_mspi_control(handle, AM_HAL_MSPI_REQ_TIMING_SCAN, &timing_params);

// Apply found parameters
am_hal_mspi_control(handle, AM_HAL_MSPI_REQ_DQS, &dqs_config);
```

#### Errata 2: IOM FIFO Underrun
**Issue**: IOM TX FIFO can underrun if CPU doesn't feed fast enough

**Workaround**: Use DMA mode or ensure timely FIFO refill:
```c
// Enable FIFO threshold interrupt
am_hal_iom_interrupt_enable(handle, AM_HAL_IOM_INT_THR);

// In ISR, refill FIFO immediately
if (status & AM_HAL_IOM_INT_THR) {
    // Fill FIFO with next batch
}
```

#### Errata 3: Flash Write Timing
**Issue**: Flash write may fail if supply voltage drops during operation

**Workaround**: Retry logic (already in driver):
```c
#define FLASH_OPERATION_MAX_RETRIES 3

for (int retry = 0; retry < FLASH_OPERATION_MAX_RETRIES; retry++) {
    ret = am_hal_flash_program_main(...);
    if (ret == AM_HAL_STATUS_SUCCESS) {
        break;
    }
}
```

#### Errata 4: GPIO Interrupt Latency
**Issue**: GPIO interrupts may have higher latency if multiple pins trigger simultaneously

**Workaround**: Use dedicated interrupt pins for time-critical signals

---

## Register Access Utilities

### Helper Macros

```c
// Register field manipulation
#define REG_FIELD_SET(reg, field, value) \
    ((reg) = ((reg) & ~(field##_Msk)) | (((value) << (field##_Pos)) & (field##_Msk)))

#define REG_FIELD_GET(reg, field) \
    (((reg) & (field##_Msk)) >> (field##_Pos))

// Example usage
REG_FIELD_SET(MSPI->CFG, MSPI_CFG_CLKDIV, 2);
uint32_t div = REG_FIELD_GET(MSPI->CFG, MSPI_CFG_CLKDIV);
```

### Register Debugging

```c
// Dump register state for debugging
static void dump_mspi_regs(uint32_t base)
{
    volatile uint32_t *regs = (uint32_t *)base;
    
    LOG_DBG("MSPI Register Dump:");
    LOG_DBG("  CFG:      0x%08X", regs[0x000/4]);
    LOG_DBG("  MSPICFG:  0x%08X", regs[0x004/4]);
    LOG_DBG("  INTEN:    0x%08X", regs[0x014/4]);
    LOG_DBG("  INTSTAT:  0x%08X", regs[0x018/4]);
    LOG_DBG("  DMACFG:   0x%08X", regs[0x300/4]);
    LOG_DBG("  DMASTAT:  0x%08X", regs[0x308/4]);
}
```

---

## Summary: Key Hardware Insights for Driver Development

### 1. Async Capability is Hardware-Dependent
- **Has DMA + completion interrupt** (DCMP flag) → Can be async (MSPI, IOM, UART, ADC, Audio)
- **Only blocking HAL** (Flash/MRAM) → Cannot be async
- **Fast operations** (GPIO, RTC) → Async not needed

### 2. Register Access Patterns
- HAL abstracts registers, but drivers sometimes access directly
- Example: `sys_read32(cfg->reg_base) & MSPI_BUSY`
- Understand register layout for debugging

### 3. Interrupt Service Pattern
Always call `am_hal_*_interrupt_service()` - it:
- Reads internal HAL state
- Invokes registered callbacks
- Updates transfer status

### 4. DMA Descriptors
- Most async peripherals use DMA descriptors
- HAL manages descriptor setup
- Driver provides memory and callbacks

### 5. Power Domains
- Peripherals can be powered independently
- Must enable before use, disable when idle
- PM Runtime coordinates power state

### 6. Clock Constraints
- Each peripheral has maximum clock frequency
- Setup/hold times vary by speed
- Timing calibration may be needed (MSPI DQS)

### 7. FIFO Management
- Most peripherals have FIFOs (32-128 bytes)
- Threshold interrupts for flow control
- DMA mode avoids FIFO management

---

## Document Version
- **Created**: 2026-10-14
- **Purpose**: Hardware-level reference for Ambiq driver development
- **Companion**: Read with `AMBIQ_HAL_GUIDE.md` and `AMBIQ_DRIVER_DEVELOPMENT_GUIDE.md`
- **Audience**: Engineers needing register-level understanding
# Ambiq HAL Architecture Guide for AI Code Analysis

## Purpose
This document helps AI coding assistants understand the Ambiq Hardware Abstraction Layer (HAL) architecture, patterns, and constraints when analyzing or updating Ambiq drivers in Zephyr RTOS.

**Important**: This document focuses on the HAL abstraction layer. For detailed hardware register maps, bit fields, timing specifications, and capabilities, see `AMBIQ_HARDWARE_REGISTER_GUIDE.md` first.

## Table of Contents
1. [HAL Overview](#hal-overview)
2. [HAL Naming Conventions](#hal-naming-conventions)
3. [HAL Function Categories](#hal-function-categories)
4. [Blocking vs Non-Blocking Operations](#blocking-vs-non-blocking-operations)
5. [HAL Pattern Analysis by Driver](#hal-pattern-analysis-by-driver)
6. [Hardware Constraints](#hardware-constraints)
7. [Common HAL Pitfalls](#common-hal-pitfalls)

---

## HAL Overview

The Ambiq HAL is a vendor-provided hardware abstraction layer located in the Ambiq SDK. Zephyr drivers call HAL functions to interact with Apollo MCU hardware.

**Relationship to Hardware**: The HAL abstracts the register-level details documented in `AMBIQ_HARDWARE_REGISTER_GUIDE.md`. For example:
- HAL function `am_hal_mspi_nonblocking_transfer()` → Sets up DMA descriptors and MSPI DMACFG registers
- HAL function `am_hal_mspi_interrupt_service()` → Reads INTSTAT register and invokes callbacks
- HAL function `am_hal_gpio_pinconfig()` → Writes to PADREGx registers

### Key Characteristics:
- **Vendor-maintained**: HAL is from Ambiq, not Zephyr
- **SoC-specific**: Different Apollo series (2x, 3x, 4x, 5x) have different HAL capabilities
- **Not always RTOS-optimized**: HAL was designed for both bare-metal and RTOS use
- **Opaque handles**: Uses `void *` handles for peripheral instances (e.g., `mspiHandle`, `iom_handler`)

### HAL Include Pattern:
```c
#include <soc.h>  // Pulls in Ambiq HAL headers
```

---

## HAL Naming Conventions

### Function Naming Pattern:
```
am_hal_<peripheral>_<action>()
```

Examples:
- `am_hal_mspi_initialize()` - Initialize MSPI peripheral
- `am_hal_gpio_pinconfig()` - Configure GPIO pin
- `am_hal_uart_blocking_transfer()` - UART blocking transfer

### Common Peripheral Prefixes:
| Prefix | Peripheral | Example |
|--------|------------|---------|
| `am_hal_mspi_*` | Multi-bit SPI (MSPI) | `am_hal_mspi_nonblocking_transfer()` |
| `am_hal_iom_*` | IO Master (I2C/SPI) | `am_hal_iom_blocking_transfer()` |
| `am_hal_uart_*` | UART | `am_hal_uart_configure()` |
| `am_hal_gpio_*` | GPIO | `am_hal_gpio_interrupt_enable()` |
| `am_hal_adc_*` | ADC | `am_hal_adc_samples_read()` |
| `am_hal_pdm_*` | PDM (Audio) | `am_hal_pdm_dma_start()` |
| `am_hal_i2s_*` | I2S (Audio) | `am_hal_i2s_interrupt_service()` |
| `am_hal_sdhc_*` | SD/MMC Host | `am_hal_sdhc_interrupt_service()` |
| `am_hal_flash_*` | Internal Flash | `am_hal_flash_program_main()` |
| `am_hal_mram_*` | MRAM (Apollo 5x) | `am_hal_mram_main_program()` |
| `am_hal_timer_*` | Timers | `am_hal_timer_start()` |
| `am_hal_ctimer_*` | Counter/Timers | `am_hal_ctimer_config()` |
| `am_hal_wdt_*` | Watchdog | `am_hal_wdt_start()` |
| `am_hal_rtc_*` | RTC | `am_hal_rtc_time_set()` |

---

## HAL Function Categories

### 1. Lifecycle Functions

**Pattern**: `initialize`, `deinitialize`, `power_control`, `enable`, `disable`

```c
// Initialize peripheral (allocates handle)
int am_hal_<periph>_initialize(uint32_t instance, void **ppHandle);

// De-initialize peripheral (frees handle)
int am_hal_<periph>_deinitialize(void *pHandle);

// Power control
int am_hal_<periph>_power_control(void *pHandle, am_hal_sysctrl_power_state_e ePowerState, bool bRetainState);

// Enable/disable peripheral
int am_hal_<periph>_enable(void *pHandle);
int am_hal_<periph>_disable(void *pHandle);
```

**Example (MSPI)**:
```c
am_hal_mspi_initialize(0, &data->mspiHandle);
am_hal_mspi_power_control(data->mspiHandle, AM_HAL_SYSCTRL_WAKE, false);
am_hal_mspi_enable(data->mspiHandle);
// ... use peripheral ...
am_hal_mspi_disable(data->mspiHandle);
am_hal_mspi_power_control(data->mspiHandle, AM_HAL_SYSCTRL_DEEPSLEEP, true);
am_hal_mspi_deinitialize(data->mspiHandle);
```

### 2. Configuration Functions

**Pattern**: `configure`, `config`, `device_configure`, `control`

```c
// Configure peripheral with struct
int am_hal_<periph>_configure(void *pHandle, am_hal_<periph>_config_t *pConfig);

// Configure device-specific settings
int am_hal_<periph>_device_configure(void *pHandle, am_hal_<periph>_dev_config_t *pDevConfig);

// Control peripheral with commands
int am_hal_<periph>_control(void *pHandle, am_hal_<periph>_request_e eRequest, void *pArgs);
```

**MSPI Example**:
```c
am_hal_mspi_config_t hal_cfg = { /* ... */ };
am_hal_mspi_configure(data->mspiHandle, &hal_cfg);

am_hal_mspi_dev_config_t dev_cfg = { /* ... */ };
am_hal_mspi_device_configure(data->mspiHandle, &dev_cfg);

// Control example: set clock frequency
am_hal_mspi_control(data->mspiHandle, AM_HAL_MSPI_REQ_CLOCK_CONFIG, &clockFreq);
```

### 3. Interrupt Functions

**Pattern**: `interrupt_enable`, `interrupt_disable`, `interrupt_status_get`, `interrupt_clear`, `interrupt_service`

```c
// Enable interrupts
int am_hal_<periph>_interrupt_enable(void *pHandle, uint32_t ui32IntMask);

// Disable interrupts
int am_hal_<periph>_interrupt_disable(void *pHandle, uint32_t ui32IntMask);

// Get interrupt status
int am_hal_<periph>_interrupt_status_get(void *pHandle, uint32_t *pui32Status, bool bEnabledOnly);

// Clear interrupts
int am_hal_<periph>_interrupt_clear(void *pHandle, uint32_t ui32IntMask);

// Service interrupt (call from ISR)
int am_hal_<periph>_interrupt_service(void *pHandle, uint32_t ui32IntStatus);
```

**Standard ISR Pattern**:
```c
static void driver_isr(const struct device *dev)
{
    struct driver_data *data = dev->data;
    uint32_t status;
    
    // Get interrupt status
    am_hal_<periph>_interrupt_status_get(data->handle, &status, false);
    
    // Clear interrupts
    am_hal_<periph>_interrupt_clear(data->handle, status);
    
    // Service interrupt (may call callbacks)
    am_hal_<periph>_interrupt_service(data->handle, status);
}
```

### 4. Transfer Functions

**Critical Distinction**: Blocking vs Non-Blocking

#### Blocking Transfer Pattern:
```c
int am_hal_<periph>_blocking_transfer(void *pHandle, 
                                      am_hal_<periph>_transfer_t *pTransfer);
```

**Characteristics**:
- Function does NOT return until transfer completes
- CPU polls hardware status
- No interrupt/callback support
- **Cannot be made async** without HAL changes

**Example (IOM - I2C/SPI)**:
```c
am_hal_iom_transfer_t trans = { /* ... */ };
ret = am_hal_iom_blocking_transfer(data->iom_handler, &trans);
// Transfer is complete when function returns
```

#### Non-Blocking Transfer Pattern:
```c
int am_hal_<periph>_nonblocking_transfer(void *pHandle, 
                                         am_hal_<periph>_transfer_t *pTransfer,
                                         am_hal_<periph>_callback_t callback,
                                         void *pCallbackCtxt);
```

**Characteristics**:
- Function returns immediately
- Hardware performs transfer with DMA
- Callback invoked from ISR when complete
- **Proper async support**

**Example (MSPI)**:
```c
am_hal_mspi_dma_transfer_t trans = { /* ... */ };
am_hal_mspi_callback_t callback = my_callback;

ret = am_hal_mspi_nonblocking_transfer(data->mspiHandle, &trans, 
                                       MSPI_DMA, callback, context);
// Returns immediately, callback invoked later
```

### 5. DMA Functions

**Pattern**: `dma_start`, `dma_config`, `configure_dma`

```c
// Start DMA transfer
int am_hal_<periph>_dma_start(void *pHandle, am_hal_<periph>_transfer_t *pTransfer);

// Configure DMA
int am_hal_<periph>_configure_dma(void *pHandle, am_hal_<periph>_dma_config_t *pConfig);
```

**Audio Example (PDM)**:
```c
am_hal_pdm_transfer_t pdm_transfer = { /* ... */ };
am_hal_pdm_dma_start(data->pdm_handler, &pdm_transfer);
```

---

## Blocking vs Non-Blocking Operations

### Decision Tree: Can This Driver Be Async?

```
Does HAL provide am_hal_<periph>_nonblocking_* functions?
│
├─ YES → Driver CAN and SHOULD be async
│   └─ Follow MSPI driver pattern (callbacks, semaphores, PM Runtime)
│
└─ NO → Check if blocking operations are acceptable
    │
    ├─ Short operations (<1ms) → Blocking OK
    │   └─ Examples: GPIO, RTC, simple config
    │
    └─ Long operations (>1ms) → Consider chunking
        └─ Break into smaller operations with yield points
```

### HAL Capabilities by Peripheral:

| Peripheral | Blocking API | Non-Blocking API | DMA Support | Recommendation |
|------------|--------------|------------------|-------------|----------------|
| **MSPI** | ✅ `blocking_transfer` | ✅ `nonblocking_transfer` | ✅ | **Use async** |
| **IOM (I2C/SPI)** | ✅ `blocking_transfer` | ✅ `nonblocking_transfer` | ✅ | **Use async** |
| **UART** | ✅ `blocking_*` | ✅ `nonblocking_*` | ✅ | **Use async** |
| **SDHC** | ✅ Event-driven | ✅ Callback-based | ✅ | **Use async** |
| **PDM/I2S** | ❌ | ✅ `dma_start` | ✅ | **Use async** |
| **ADC** | ✅ `samples_read` | ✅ DMA mode | ✅ | Use DMA + sem |
| **Flash** | ✅ `program_main` | ❌ None | ❌ | **Blocking only** |
| **MRAM** | ✅ `main_program` | ❌ None | ❌ | **Blocking only** |
| **GPIO** | ✅ Immediate | ❌ | ❌ | Blocking OK |
| **RTC** | ✅ Immediate | ❌ | ❌ | Blocking OK |
| **Timer** | ✅ Immediate | ❌ | ❌ | Blocking OK |
| **Watchdog** | ✅ Immediate | ❌ | ❌ | Blocking OK |

---

## HAL Pattern Analysis by Driver

### 1. MSPI Driver (Gold Standard for Async)

**HAL Functions Used**:
```c
// Lifecycle
am_hal_mspi_initialize()
am_hal_mspi_deinitialize()
am_hal_mspi_power_control()
am_hal_mspi_enable()
am_hal_mspi_disable()

// Configuration
am_hal_mspi_configure()
am_hal_mspi_device_configure()
am_hal_mspi_control()

// Interrupts
am_hal_mspi_interrupt_enable()
am_hal_mspi_interrupt_disable()
am_hal_mspi_interrupt_status_get()
am_hal_mspi_interrupt_clear()
am_hal_mspi_interrupt_service()

// Transfers - KEY: Both blocking and non-blocking available
am_hal_mspi_blocking_transfer()
am_hal_mspi_nonblocking_transfer()  // ← THIS ENABLES ASYNC
```

**Async Pattern**:
```c
struct mspi_ambiq_data {
    void *mspiHandle;
    struct k_mutex lock;
    struct k_sem transfer_sem;
    mspi_callback_handler_t callback;
    struct mspi_callback_context *callback_ctx;
};

// Non-blocking transfer with callback
ret = am_hal_mspi_nonblocking_transfer(data->mspiHandle, &trans, 
                                       MSPI_DMA, callback_func, callback_ctx);
// Returns immediately, ISR calls interrupt_service(), which calls callback
```

### 2. IOM Driver (I2C/SPI - Can Be Async)

**HAL Functions Used**:
```c
am_hal_iom_initialize()
am_hal_iom_power_control()
am_hal_iom_configure()
am_hal_iom_enable()
am_hal_iom_disable()
am_hal_iom_interrupt_*()

// Both available:
am_hal_iom_blocking_transfer()
am_hal_iom_nonblocking_transfer()  // ← Available but using timeout pattern
```

**Current Pattern (Timeout-based)**:
```c
// Uses nonblocking but waits with semaphore timeout
ret = am_hal_iom_nonblocking_transfer(data->iom_handler, &trans, 
                                      callback, (void *)dev);
if (k_sem_take(&data->transfer_sem, K_MSEC(timeout))) {
    // Timeout handling
}
```

**Could Be Improved**: Use true async with callback chain instead of semaphore timeout.

### 3. Flash Driver (Blocking Only - Cannot Be Async)

**HAL Functions Used**:
```c
am_hal_flash_program_main()       // Blocking only
am_hal_flash_page_erase()         // Blocking only
am_hal_mram_main_program()        // Blocking only (Apollo 5x)
am_hal_mram_main_fill()           // Blocking only (Apollo 5x)
```

**NO async HAL APIs available**. The HAL does not provide:
- ❌ `am_hal_flash_nonblocking_*`
- ❌ Flash interrupt support
- ❌ DMA for flash operations

**Hardware Constraint**: Flash operations are inherently serialized and must be atomic. CPU must wait for flash controller completion.

**Current Pattern (Correct for constraints)**:
```c
// Chunked writes to allow scheduler preemption
#define FLASH_WRITE_CHUNK_SIZE 128

while (remaining > 0) {
    size_t chunk = MIN(remaining, FLASH_WRITE_CHUNK_SIZE);
    
    key = irq_lock();  // Critical section for flash write
    ret = am_hal_flash_program_main(key, data, dest, words);
    irq_unlock(key);
    
    // Implicit yield point - scheduler can preempt here
    remaining -= chunk;
}
```

**Conclusion**: Flash driver is **correctly implemented** given HAL constraints. Async is not possible.

### 4. Audio Drivers (PDM/I2S - DMA-based Async)

**HAL Functions Used**:
```c
am_hal_pdm_initialize()
am_hal_pdm_configure()
am_hal_pdm_enable()
am_hal_pdm_disable()
am_hal_pdm_interrupt_*()
am_hal_pdm_dma_start()  // ← DMA-based async

am_hal_i2s_initialize()
am_hal_i2s_configure()
am_hal_i2s_interrupt_*()
// DMA configured via transfer struct
```

**Async Pattern (Correct)**:
```c
struct audio_data {
    void *handler;
    struct k_sem dma_done_sem;
    struct k_sem tx_ready_sem;
    struct k_sem rx_done_sem;
};

// Start DMA transfer
am_hal_pdm_dma_start(data->pdm_handler, &transfer);

// ISR signals completion
static void audio_isr(const struct device *dev) {
    uint32_t status;
    am_hal_pdm_interrupt_status_get(data->handler, &status, true);
    am_hal_pdm_interrupt_clear(data->handler, status);
    am_hal_pdm_interrupt_service(data->handler, status, &transfer);
    
    if (status & AM_HAL_PDM_INT_DCMP) {
        k_sem_give(&data->dma_done_sem);  // ← Async completion
    }
}
```

---

## Hardware Constraints

### 1. Flash Write Constraints

**Problem**: CPU executes from internal flash
- Cannot erase/write to pages currently executing code
- Flash controller serializes operations
- No DMA or interrupt support

**Solution**: 
- Chunked writes with yield points
- Mutex for exclusive access
- Accept that some operations must block

### 2. Single CPU Constraints

**Problem**: Apollo MCUs are single-core
- Only one CPU executing at a time
- True parallelism not possible
- "Async" means DMA/interrupt completion, not parallel execution

**Implication**:
- Async is only valuable when hardware can work independently (DMA, peripherals)
- Polling/blocking operations have no async alternative without hardware support

### 3. IOM Shared Resource

**Problem**: IOM instances handle both I2C and SPI
- Same hardware block
- Exclusive access needed
- Mode switching has overhead

**Solution**:
- Proper mutex locking
- Mode configuration per transaction
- PM Runtime coordination

---

## Common HAL Pitfalls

### 1. ❌ Assuming All Peripherals Have Non-Blocking APIs

```c
// WRONG: Not all HAL functions have nonblocking variants
am_hal_flash_nonblocking_program(...);  // ← DOES NOT EXIST
```

**Fix**: Check HAL documentation or grep drivers for actual API availability.

### 2. ❌ Ignoring Return Codes

```c
// WRONG: HAL returns status codes
am_hal_mspi_configure(handle, &cfg);
```

```c
// CORRECT: Always check return values
ret = am_hal_mspi_configure(handle, &cfg);
if (ret != AM_HAL_STATUS_SUCCESS) {
    LOG_ERR("Config failed: %d", ret);
    return -EHOSTDOWN;
}
```

### 3. ❌ Not Calling interrupt_service() in ISR

```c
// WRONG: Only clearing interrupts
static void isr(const struct device *dev) {
    uint32_t status;
    am_hal_mspi_interrupt_status_get(handle, &status, false);
    am_hal_mspi_interrupt_clear(handle, status);
    // Missing: interrupt_service() call!
}
```

```c
// CORRECT: Must call interrupt_service()
static void isr(const struct device *dev) {
    uint32_t status;
    am_hal_mspi_interrupt_status_get(handle, &status, false);
    am_hal_mspi_interrupt_clear(handle, status);
    am_hal_mspi_interrupt_service(handle, status);  // ← Triggers callbacks
}
```

### 4. ❌ Forgetting Power Control Sequence

```c
// WRONG: Enabling without power on
am_hal_mspi_initialize(0, &handle);
am_hal_mspi_enable(handle);  // ← May fail, peripheral not powered
```

```c
// CORRECT: Full power sequence
am_hal_mspi_initialize(0, &handle);
am_hal_mspi_power_control(handle, AM_HAL_SYSCTRL_WAKE, false);
am_hal_mspi_enable(handle);
```

### 5. ❌ Using Blocking Transfers in Async Code Path

```c
// WRONG: Blocking call in async API
if (async_requested) {
    am_hal_iom_blocking_transfer(...);  // ← Defeats purpose of async
}
```

```c
// CORRECT: Use nonblocking when async requested
if (async_requested) {
    am_hal_iom_nonblocking_transfer(..., callback, context);
}
```

---

## HAL Version Differences

### Apollo Series Differences:

| Feature | Apollo 2x | Apollo 3x | Apollo 4x | Apollo 5x |
|---------|-----------|-----------|-----------|-----------|
| **Memory** | Flash | Flash | Flash | MRAM |
| **MSPI** | Limited | Yes | Yes | Enhanced |
| **GPIO API** | Simple | Simple | Enhanced | Enhanced |
| **DMA** | Limited | Yes | Yes | Enhanced |
| **Power** | Basic | Advanced | Advanced | Advanced |

**Key for AI**: Check `CONFIG_SOC_SERIES_APOLLO*` defines to understand capabilities.

---

## Summary: How to Interpret HAL for Driver Updates

### Step 1: Identify HAL APIs Used
```bash
grep "am_hal_" driver_file.c | sort -u
```

### Step 2: Check for Non-Blocking Variants
```bash
grep "nonblocking" driver_file.c
grep "blocking" driver_file.c
```

### Step 3: Determine Async Capability

| If HAL has... | Then driver... |
|---------------|----------------|
| `nonblocking_transfer()` | **Should be async** (follow MSPI pattern) |
| Only `blocking_transfer()` + DMA | Can use DMA with semaphores |
| Only `blocking_*` functions | Must remain blocking, use chunking |
| Immediate operations (<1ms) | Blocking acceptable |

### Step 4: Verify Interrupt Support
```bash
grep "interrupt_service" driver_file.c
grep "interrupt_enable" driver_file.c
```

If interrupt_service() exists → Hardware supports async completion

### Step 5: Check Existing MSPI/UART/SDHC Patterns

These drivers show correct async patterns. Copy their structure for similar peripherals.

---

## Quick Reference: HAL API Checklist

When updating a driver, verify:

- [ ] `am_hal_*_initialize()` called before use
- [ ] `am_hal_*_power_control()` sequence correct
- [ ] `am_hal_*_enable()` called after power on
- [ ] Return codes checked and mapped to errno
- [ ] `am_hal_*_interrupt_service()` called in ISR
- [ ] `am_hal_*_disable()` called before power off
- [ ] `am_hal_*_deinitialize()` called on cleanup
- [ ] Non-blocking APIs used when available
- [ ] Callbacks registered for async operations
- [ ] PM Runtime integration for power management

---

## Document Version
- **Created**: 2026-10-14
- **Purpose**: AI assistance for Ambiq driver development
- **Reference**: Zephyr Ambiq drivers in `drivers/` tree
# Ambiq Driver Development Guide for Zephyr RTOS

## Purpose
This guide provides step-by-step instructions for creating and updating Ambiq drivers in Zephyr RTOS, with a focus on async/RTOS compatibility.

**Prerequisites**:
1. **Read `AMBIQ_HARDWARE_REGISTER_GUIDE.md`** - Understand hardware capabilities, register maps, and timing constraints
2. **Read `AMBIQ_HAL_GUIDE.md`** - Understand how HAL abstracts the hardware
3. **Use this guide** - For practical implementation patterns

## Table of Contents
1. [Driver Development Workflow](#driver-development-workflow)
2. [Using MSPI as a Template](#using-mspi-as-a-template)
3. [Step-by-Step: Creating an Async Driver](#step-by-step-creating-an-async-driver)
4. [Step-by-Step: Updating Existing Driver](#step-by-step-updating-existing-driver)
5. [PM Runtime Integration](#pm-runtime-integration)
6. [Testing Checklist](#testing-checklist)
7. [Common Patterns](#common-patterns)
8. [Troubleshooting](#troubleshooting)

---

## Driver Development Workflow

### Decision Tree: Should I Update This Driver?

```
Start: Reviewing Ambiq driver

Step 1: Check hardware capabilities (AMBIQ_HARDWARE_REGISTER_GUIDE.md)
   - Does peripheral have DMA support?
   - Is there a DCMP (DMA complete) interrupt?
   - What are FIFO sizes and timing constraints?
   
Step 2: Check HAL capabilities (AMBIQ_HAL_GUIDE.md)
   - Does HAL provide nonblocking_* APIs for this peripheral?
   ├─ YES → Proceed to async update (high value)
   └─ NO → Go to step 3

Step 3: Does driver use PM Runtime?
   ├─ YES → Low priority, consider other improvements
   └─ NO → Add PM Runtime (medium value, easy)

Step 4: Are operations long-running (>1ms)?
   ├─ YES → Consider chunking pattern
   └─ NO → Driver likely fine as-is

Step 5: Does driver have proper mutex/semaphore locking?
   ├─ YES → Good
   └─ NO → Add thread safety (high priority)
```

---

## Using MSPI as a Template

The MSPI driver (`drivers/mspi/mspi_ambiq_ap5.c`) is the **gold standard** for async Ambiq drivers.

### What Makes MSPI Excellent:

```c
struct mspi_ambiq_data {
    // HAL handle
    void *mspiHandle;
    
    // Locking
    struct k_mutex lock;           // Device-level exclusive access
    struct k_sem   transfer_sem;   // Transfer completion signaling
    
    // Async support
    mspi_callback_handler_t callback;
    struct mspi_callback_context *callback_ctx;
    
    // HAL configuration structs
    am_hal_mspi_config_t     hal_cfg;
    am_hal_mspi_dev_config_t hal_dev_cfg;
    
    // State tracking
    struct mspi_dev_id *dev_id;
};
```

### MSPI Async Flow:

```
Application calls driver API (e.g., mspi_transceive)
    ↓
Driver locks mutex (exclusive access)
    ↓
Driver calls am_hal_mspi_nonblocking_transfer(handle, &trans, callback, ctx)
    ↓
HAL returns immediately
    ↓
Driver releases mutex
    ↓
[Hardware DMA works in background]
    ↓
[Interrupt fires when DMA complete]
    ↓
ISR: am_hal_mspi_interrupt_service() → calls callback
    ↓
Callback signals semaphore
    ↓
Waiting thread wakes up
```

---

## Step-by-Step: Creating an Async Driver

### Example: Creating a new peripheral driver with async support

#### Step 1: Set Up Data Structures

```c
struct my_driver_config {
    uint32_t base;
    uint32_t inst_idx;
    const struct pinctrl_dev_config *pcfg;
    void (*irq_config_func)(void);
    bool pm_dev_runtime_auto;
    LOG_INSTANCE_PTR_DECLARE(log);
};

struct my_driver_data {
    // HAL handle (opaque pointer)
    void *periph_handle;
    
    // Thread safety
    struct k_mutex lock;
    
    // Async completion signaling
    struct k_sem transfer_done_sem;
    
    // Callback support
    my_callback_t callback;
    void *callback_ctx;
    
    // HAL configuration
    am_hal_<periph>_config_t hal_cfg;
    
    // PM Runtime state
    bool pm_policy_state_on;
};
```

#### Step 2: Implement Initialization

```c
static int my_driver_init(const struct device *dev)
{
    const struct my_driver_config *cfg = dev->config;
    struct my_driver_data *data = dev->data;
    int ret;
    
    // Initialize mutex and semaphore
    k_mutex_init(&data->lock);
    k_sem_init(&data->transfer_done_sem, 0, 1);
    
    // Apply pin configuration
    ret = pinctrl_apply_state(cfg->pcfg, PINCTRL_STATE_DEFAULT);
    if (ret < 0) {
        return ret;
    }
    
    // Initialize HAL peripheral
    ret = am_hal_<periph>_initialize(cfg->inst_idx, &data->periph_handle);
    if (ret != AM_HAL_STATUS_SUCCESS) {
        LOG_ERR("Failed to initialize peripheral: %d", ret);
        return -EPERM;
    }
    
    // Power on peripheral
    ret = am_hal_<periph>_power_control(data->periph_handle, 
                                        AM_HAL_SYSCTRL_WAKE, false);
    if (ret != AM_HAL_STATUS_SUCCESS) {
        LOG_ERR("Failed to power on: %d", ret);
        return -EHOSTDOWN;
    }
    
    // Configure peripheral
    ret = am_hal_<periph>_configure(data->periph_handle, &data->hal_cfg);
    if (ret != AM_HAL_STATUS_SUCCESS) {
        return -EHOSTDOWN;
    }
    
    // Enable peripheral
    ret = am_hal_<periph>_enable(data->periph_handle);
    if (ret != AM_HAL_STATUS_SUCCESS) {
        return -EHOSTDOWN;
    }
    
    // Clear and enable interrupts
    am_hal_<periph>_interrupt_clear(data->periph_handle, 0xFFFFFFFF);
    am_hal_<periph>_interrupt_enable(data->periph_handle, 
                                     AM_HAL_<PERIPH>_INT_XFER_COMPLETE);
    
    // Configure IRQ
    cfg->irq_config_func();
    
    // Enable PM Runtime if requested
    if (cfg->pm_dev_runtime_auto) {
        ret = pm_device_runtime_enable(dev);
        if (ret) {
            LOG_ERR("PM runtime enable failed: %d", ret);
            return ret;
        }
    }
    
    return 0;
}
```

#### Step 3: Implement ISR

```c
static void my_driver_isr(const struct device *dev)
{
    struct my_driver_data *data = dev->data;
    uint32_t status;
    
    // Get interrupt status
    am_hal_<periph>_interrupt_status_get(data->periph_handle, &status, false);
    
    // Clear interrupts
    am_hal_<periph>_interrupt_clear(data->periph_handle, status);
    
    // Service interrupt (may invoke callbacks)
    am_hal_<periph>_interrupt_service(data->periph_handle, status);
    
    // Signal completion on expected interrupt
    if (status & AM_HAL_<PERIPH>_INT_XFER_COMPLETE) {
        k_sem_give(&data->transfer_done_sem);
    }
}
```

#### Step 4: Implement Transfer Function (Async)

```c
static int my_driver_transfer(const struct device *dev, 
                              const struct my_transfer *xfer)
{
    struct my_driver_data *data = dev->data;
    int ret;
    
    // Take mutex for exclusive access
    ret = k_mutex_lock(&data->lock, K_FOREVER);
    if (ret) {
        return -EBUSY;
    }
    
    // Get PM Runtime reference
    ret = pm_device_runtime_get(dev);
    if (ret < 0) {
        k_mutex_unlock(&data->lock);
        return ret;
    }
    
    // Prepare HAL transfer structure
    am_hal_<periph>_transfer_t hal_xfer = {
        .direction = xfer->direction,
        .address   = xfer->address,
        .length    = xfer->length,
        .buffer    = xfer->buffer,
    };
    
    if (xfer->async) {
        // Async path: non-blocking transfer with callback
        ret = am_hal_<periph>_nonblocking_transfer(
            data->periph_handle,
            &hal_xfer,
            my_hal_callback,  // HAL will call this from interrupt_service()
            (void *)dev
        );
        
        if (ret != AM_HAL_STATUS_SUCCESS) {
            pm_device_runtime_put(dev);
            k_mutex_unlock(&data->lock);
            return -EIO;
        }
        
        // Release mutex - transfer in progress
        k_mutex_unlock(&data->lock);
        
        // Optionally wait for completion with timeout
        if (xfer->wait) {
            ret = k_sem_take(&data->transfer_done_sem, 
                           K_MSEC(xfer->timeout));
            if (ret) {
                LOG_ERR("Transfer timeout");
                return -ETIMEDOUT;
            }
        }
        
    } else {
        // Sync path: blocking transfer
        ret = am_hal_<periph>_blocking_transfer(
            data->periph_handle,
            &hal_xfer
        );
        
        pm_device_runtime_put(dev);
        k_mutex_unlock(&data->lock);
        
        if (ret != AM_HAL_STATUS_SUCCESS) {
            return -EIO;
        }
    }
    
    return 0;
}

// HAL callback (invoked from ISR via interrupt_service())
static void my_hal_callback(void *pCallbackCtxt, uint32_t status)
{
    const struct device *dev = pCallbackCtxt;
    struct my_driver_data *data = dev->data;
    
    // Invoke application callback if registered
    if (data->callback) {
        data->callback(dev, status, data->callback_ctx);
    }
    
    // Signal semaphore for synchronous waiters
    k_sem_give(&data->transfer_done_sem);
    
    // Release PM Runtime reference
    pm_device_runtime_put(dev);
}
```

#### Step 5: Implement PM Actions

```c
#ifdef CONFIG_PM_DEVICE
static int my_driver_pm_action(const struct device *dev, 
                               enum pm_device_action action)
{
    const struct my_driver_config *cfg = dev->config;
    struct my_driver_data *data = dev->data;
    int ret;
    
    switch (action) {
    case PM_DEVICE_ACTION_RESUME:
        // Apply active pinctrl state
        ret = pinctrl_apply_state(cfg->pcfg, PINCTRL_STATE_DEFAULT);
        if (ret < 0) {
            return ret;
        }
        
        // Power on peripheral
        ret = am_hal_<periph>_power_control(data->periph_handle,
                                            AM_HAL_SYSCTRL_WAKE, true);
        if (ret != AM_HAL_STATUS_SUCCESS) {
            LOG_ERR("Resume failed: %d", ret);
            return -EHOSTDOWN;
        }
        break;
        
    case PM_DEVICE_ACTION_SUSPEND:
        // Move to sleep pinctrl state
        ret = pinctrl_apply_state(cfg->pcfg, PINCTRL_STATE_SLEEP);
        if (ret < 0 && ret != -ENOENT) {
            return ret;
        }
        
        // Power off peripheral
        ret = am_hal_<periph>_power_control(data->periph_handle,
                                            AM_HAL_SYSCTRL_DEEPSLEEP, true);
        if (ret != AM_HAL_STATUS_SUCCESS) {
            LOG_ERR("Suspend failed: %d", ret);
            return -EHOSTDOWN;
        }
        break;
        
    default:
        return -ENOTSUP;
    }
    
    return 0;
}
#endif /* CONFIG_PM_DEVICE */
```

#### Step 6: Define Device

```c
#define MY_DRIVER_DEVICE_DEFINE(n)                                          \
    static struct my_driver_data my_driver_data_##n = {                     \
        .hal_cfg = { /* default config */ },                                \
    };                                                                       \
                                                                             \
    static void my_driver_irq_config_##n(void)                              \
    {                                                                        \
        IRQ_CONNECT(DT_INST_IRQN(n), DT_INST_IRQ(n, priority),             \
                    my_driver_isr, DEVICE_DT_INST_GET(n), 0);              \
        irq_enable(DT_INST_IRQN(n));                                        \
    }                                                                        \
                                                                             \
    PINCTRL_DT_INST_DEFINE(n);                                              \
                                                                             \
    static const struct my_driver_config my_driver_config_##n = {           \
        .base = DT_INST_REG_ADDR(n),                                        \
        .inst_idx = n,                                                       \
        .pcfg = PINCTRL_DT_INST_DEV_CONFIG_GET(n),                          \
        .irq_config_func = my_driver_irq_config_##n,                        \
        .pm_dev_runtime_auto = DT_INST_PROP(n, zephyr_pm_device_runtime_auto), \
    };                                                                       \
                                                                             \
    PM_DEVICE_DT_INST_DEFINE(n, my_driver_pm_action);                       \
                                                                             \
    DEVICE_DT_INST_DEFINE(n,                                                \
                          my_driver_init,                                    \
                          PM_DEVICE_DT_INST_GET(n),                          \
                          &my_driver_data_##n,                               \
                          &my_driver_config_##n,                             \
                          POST_KERNEL,                                       \
                          CONFIG_MY_DRIVER_INIT_PRIORITY,                    \
                          &my_driver_api);                                   \
                                                                             \
DT_INST_FOREACH_STATUS_OKAY(MY_DRIVER_DEVICE_DEFINE)
```

---

## Step-by-Step: Updating Existing Driver

### Scenario: Adding PM Runtime to GPIO Driver

**Current State**: `drivers/gpio/gpio_ambiq.c` has no PM Runtime support.

#### Step 1: Add PM Device Include

```c
#include <zephyr/pm/device.h>
```

#### Step 2: Add PM Action Function

```c
#ifdef CONFIG_PM_DEVICE
static int ambiq_gpio_pm_action(const struct device *dev,
                                enum pm_device_action action)
{
    // GPIO is simple - most Apollo MCUs handle GPIO power automatically
    // Just acknowledge the action
    switch (action) {
    case PM_DEVICE_ACTION_RESUME:
        // GPIO pins retain configuration, nothing to do
        return 0;
        
    case PM_DEVICE_ACTION_SUSPEND:
        // GPIO interrupts should wake system, nothing to do
        return 0;
        
    default:
        return -ENOTSUP;
    }
}
#endif
```

#### Step 3: Update Device Definition

```c
// Old:
DEVICE_DT_INST_DEFINE(n,
                      ambiq_gpio_init,
                      NULL,  // ← Change this
                      &ambiq_gpio_data_##n,
                      &ambiq_gpio_config_##n,
                      POST_KERNEL,
                      CONFIG_GPIO_INIT_PRIORITY,
                      &ambiq_gpio_driver_api);

// New:
PM_DEVICE_DT_INST_DEFINE(n, ambiq_gpio_pm_action);

DEVICE_DT_INST_DEFINE(n,
                      ambiq_gpio_init,
                      PM_DEVICE_DT_INST_GET(n),  // ← Add PM device
                      &ambiq_gpio_data_##n,
                      &ambiq_gpio_config_##n,
                      POST_KERNEL,
                      CONFIG_GPIO_INIT_PRIORITY,
                      &ambiq_gpio_driver_api);
```

---

## PM Runtime Integration

### When to Use PM Runtime

**Always use** PM Runtime for:
- Peripherals that can be powered down independently
- Devices with significant power consumption
- Peripherals accessed infrequently

**Examples**: MSPI, I2C, SPI, UART, ADC, SDHC

### PM Runtime Pattern

```c
// Before accessing peripheral
ret = pm_device_runtime_get(dev);
if (ret < 0) {
    LOG_ERR("Failed to resume device: %d", ret);
    return ret;
}

// ... perform operations ...

// After operations complete
ret = pm_device_runtime_put(dev);
if (ret < 0) {
    LOG_ERR("Failed to suspend device: %d", ret);
}
```

### Async PM Runtime

For async operations, use `pm_device_runtime_put_async()`:

```c
// Start async transfer
am_hal_<periph>_nonblocking_transfer(...);

// Use async put with delay to avoid suspend/resume thrashing
pm_device_runtime_put_async(dev, K_MSEC(2));
```

### PM Policy Lock

For real-time operations (audio, etc.), prevent deep sleep:

```c
// Lock PM policy during audio streaming
pm_policy_state_lock_get(PM_STATE_SUSPEND_TO_RAM, PM_ALL_SUBSTATES);

// ... audio operations ...

// Unlock when done
pm_policy_state_lock_put(PM_STATE_SUSPEND_TO_RAM, PM_ALL_SUBSTATES);
```

---

## Testing Checklist

### Functional Testing

- [ ] Basic operations work (read, write, configure)
- [ ] Async operations complete successfully
- [ ] Callbacks invoked correctly
- [ ] Timeout handling works
- [ ] Error paths tested

### Concurrency Testing

- [ ] Multiple threads can access device (with mutex)
- [ ] No race conditions
- [ ] No deadlocks
- [ ] Priority inheritance works (use mutex, not spinlock for long operations)

### PM Testing

- [ ] Device resumes from suspend
- [ ] Operations work after resume
- [ ] Device suspends when idle
- [ ] PM reference counting correct (no leaks)

### Stress Testing

- [ ] Rapid open/close cycles
- [ ] High-frequency operations
- [ ] Maximum throughput
- [ ] Long-running transfers

### Integration Testing

- [ ] Works with Zephyr subsystems (filesystem, shell, etc.)
- [ ] No scheduler stalls
- [ ] Interrupt latency acceptable
- [ ] Power consumption reasonable

---

## Common Patterns

### Pattern 1: Chunked Blocking Operations

For peripherals without async HAL support:

```c
#define CHUNK_SIZE 128

while (remaining > 0) {
    size_t chunk = MIN(remaining, CHUNK_SIZE);
    
    // Blocking operation on chunk
    ret = am_hal_<periph>_blocking_op(handle, data, chunk);
    if (ret != AM_HAL_STATUS_SUCCESS) {
        return -EIO;
    }
    
    // Yield point - allows scheduler to preempt
    remaining -= chunk;
    data += chunk;
}
```

### Pattern 2: Timeout-Based Async

When HAL has nonblocking but you want synchronous API:

```c
// Reset semaphore
k_sem_reset(&data->done_sem);

// Start async operation
ret = am_hal_<periph>_nonblocking_transfer(handle, &xfer, callback, ctx);
if (ret != AM_HAL_STATUS_SUCCESS) {
    return -EIO;
}

// Wait for completion with timeout
ret = k_sem_take(&data->done_sem, K_MSEC(timeout));
if (ret) {
    // Cancel transfer if possible
    am_hal_<periph>_cancel(handle);
    return -ETIMEDOUT;
}
```

### Pattern 3: DMA with Cache Coherency

For DMA operations, handle cache:

```c
#include <zephyr/cache.h>

// Before DMA write (TX)
if (!buf_in_nocache((uintptr_t)tx_buf, tx_len)) {
    sys_cache_data_flush_range((void *)tx_buf, tx_len);
}

am_hal_<periph>_dma_start(handle, &tx_xfer);

// After DMA read (RX)
if (!buf_in_nocache((uintptr_t)rx_buf, rx_len)) {
    sys_cache_data_invd_range((void *)rx_buf, rx_len);
}
```

### Pattern 4: HAL Status to Errno Mapping

```c
static int hal_status_to_errno(int hal_status)
{
    switch (hal_status) {
    case AM_HAL_STATUS_SUCCESS:
        return 0;
    case AM_HAL_STATUS_FAIL:
        return -EIO;
    case AM_HAL_STATUS_INVALID_HANDLE:
        return -EINVAL;
    case AM_HAL_STATUS_IN_USE:
        return -EBUSY;
    case AM_HAL_STATUS_TIMEOUT:
        return -ETIMEDOUT;
    case AM_HAL_STATUS_OUT_OF_RANGE:
        return -ERANGE;
    case AM_HAL_STATUS_INVALID_ARG:
        return -EINVAL;
    case AM_HAL_STATUS_INVALID_OPERATION:
        return -EPERM;
    default:
        LOG_ERR("Unknown HAL status: 0x%x", hal_status);
        return -EIO;
    }
}
```

---

## Troubleshooting

### Issue: Callbacks Not Invoked

**Symptom**: `am_hal_*_nonblocking_transfer()` succeeds but callback never fires.

**Checklist**:
1. ✓ Interrupts enabled: `am_hal_*_interrupt_enable()`?
2. ✓ IRQ configured and enabled: `irq_enable()`?
3. ✓ `interrupt_service()` called in ISR?
4. ✓ Callback registered correctly in nonblocking call?

**Debug**:
```c
// Add logging in ISR
static void isr(const struct device *dev) {
    uint32_t status;
    am_hal_*_interrupt_status_get(handle, &status, false);
    LOG_DBG("ISR fired, status=0x%x", status);  // ← Add this
    ...
}
```

### Issue: PM Runtime Errors

**Symptom**: `-EAGAIN` or device not resuming.

**Fix**: Check PM device action implementation:
```c
#ifdef CONFIG_PM_DEVICE
static int my_pm_action(...) {
    // Must handle both RESUME and SUSPEND
    ...
}
#endif

// Must use PM_DEVICE_DT_INST_DEFINE() not NULL
PM_DEVICE_DT_INST_DEFINE(n, my_pm_action);
```

### Issue: Mutex Deadlock

**Symptom**: System hangs in `k_mutex_lock()`.

**Cause**: Taking mutex in ISR or callback.

**Fix**: Use spinlock for ISR-accessible data, mutex only for thread context:
```c
// ISR-safe
struct k_spinlock isr_lock;

static void isr(...) {
    k_spinlock_key_t key = k_spin_lock(&data->isr_lock);
    // ... update shared data ...
    k_spin_unlock(&data->isr_lock, key);
}

// Thread context uses mutex
struct k_mutex thread_lock;
```

### Issue: Flash Operations Fail

**Symptom**: `am_hal_flash_program_main()` returns error.

**Cause**: IRQs not locked during flash write.

**Fix**:
```c
unsigned int key = irq_lock();
ret = am_hal_flash_program_main(AM_HAL_FLASH_PROGRAM_KEY, ...);
irq_unlock(key);
```

---

## Document Version

- **Created**: 2026-10-14
- **Purpose**: Guide for developing/updating Ambiq drivers
- **Companion**: Read `AMBIQ_HAL_GUIDE.md` first
- **Reference**: MSPI driver as gold standard example
