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
