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
