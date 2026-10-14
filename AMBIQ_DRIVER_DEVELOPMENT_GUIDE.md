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
