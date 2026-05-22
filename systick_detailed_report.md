# Detailed SysTick Analysis for FreeRTOS Compatibility

## Executive Summary
This report provides a deep dive into libDaisy's timing system, focusing on the relationship between SysTick, HAL timing functions, and the internal TimerHandle class. The analysis reveals that libDaisy has a dual timing system that complicates FreeRTOS integration.

## Core Findings

### 1. Dual Timing System Architecture

libDaisy employs **two distinct timing mechanisms**:

#### A. SysTick-based Timing (HAL-dependent)
- **Location**: `src/sys/system.cpp` lines 92-93, 273
- **Functions**: 
  - `SysTick_Handler()`: Calls `HAL_IncTick()` and `HAL_SYSTICK_IRQHandler()`
  - `System::GetNow()`: Returns `HAL_GetTick()`
  - `dsy_system_getnow()`: Also returns `HAL_GetTick()` (commented out)
  - `System::Delay()`: Uses `HAL_Delay()`

#### B. TimerHandle-based Timing (TIM2 hardware timer)
- **Location**: `src/sys/system.cpp` lines 235-242, 278-298
- **Initialization**: 
  ```cpp
  TimerHandle::Config timcfg;
  timcfg.periph = TimerHandle::Config::Peripheral::TIM_2;
  timcfg.dir    = TimerHandle::Config::CounterDir::UP;
  tim_.Init(timcfg);
  tim_.Start();
  ```
- **Functions**:
  - `System::GetUs()`: Returns `tim_.GetUs()`
  - `System::GetTick()`: Returns `tim_.GetTick()`
  - `System::DelayUs()`: Uses `tim_.DelayUs()`
  - `System::DelayTicks()`: Uses `tim_.DelayTick()`

### 2. Critical HAL Dependencies

Found **4 direct HAL timing function usages** in `system.cpp`:

| Line | Function | Context |
|------|----------|---------|
| 92 | `HAL_IncTick()` | Called in `SysTick_Handler()` |
| 93 | `HAL_SYSTICK_IRQHandler()` | Called in `SysTick_Handler()` |
| 273 | `HAL_GetTick()` | Returned by `System::GetNow()` |
| 645 | `HAL_GetTick()` | Commented out in `dsy_system_getnow()` |

### 3. External Dependencies

**SD Card Operations** (`src/util/sd_diskio.c`):
- Uses `HAL_GetTick()` for timeout calculations in multiple functions
- Lines 155-156, 165-167, 226-227, 236-238
- Critical for SD card read/write operations

## FreeRTOS Compatibility Issues

### Problem 1: SysTick Handler Conflict
- FreeRTOS **requires exclusive control** of SysTick for task scheduling
- libDaisy's `SysTick_Handler()` directly calls HAL functions
- Simple weak linking won't work due to HAL dependencies

### Problem 2: HAL Tick Variable Dependencies
- `HAL_GetTick()` returns a global variable (`uwTick`) maintained by `HAL_IncTick()`
- This variable is incremented in libDaisy's `SysTick_Handler()`
- Multiple libDaisy functions depend on this tick count

### Problem 3: Mixed Timing Sources
- Some functions use SysTick/HAL timing (`GetNow()`, `Delay()`)
- Others use TIM2 timing (`GetUs()`, `GetTick()`, `DelayUs()`)
- This dual system creates inconsistency for FreeRTOS integration

## Detailed Function Analysis

### Functions Depending on HAL/SysTick

#### `System::GetNow()` (line 272)
```cpp
uint32_t System::GetNow()
{
    return HAL_GetTick();  // Depends on HAL's uwTick variable
}
```
- **Purpose**: Returns milliseconds since system start
- **Dependency**: Directly returns HAL's tick counter
- **Impact**: Will break if FreeRTOS controls SysTick

#### `System::Delay(uint32_t delay_ms)` (line 284)
```cpp
void System::Delay(uint32_t delay_ms)
{
    HAL_Delay(delay_ms);  // Uses HAL's busy-wait with SysTick
}
```
- **Purpose**: Blocking delay in milliseconds
- **Dependency**: Uses HAL_Delay which relies on SysTick
- **Impact**: Will not work correctly with FreeRTOS SysTick

### Functions Using Independent Timer (TIM2)

#### `System::GetUs()` (line 277)
```cpp
uint32_t System::GetUs()
{
    return tim_.GetUs();  // Uses TIM2, not SysTick
}
```
- **Purpose**: Returns microseconds within timer cycle
- **Dependency**: Independent TIM2 hardware timer
- **Impact**: Should work fine with FreeRTOS

#### `System::GetTick()` (line 282)
```cpp
uint32_t System::GetTick()
{
    return tim_.GetTick();  // Uses TIM2, not SysTick
}
```
- **Purpose**: Returns ticks at (PCLk1 * 2)Hz
- **Dependency**: Independent TIM2 hardware timer
- **Impact**: Should work fine with FreeRTOS

## SD Card Timeout Mechanism

The SD card driver uses HAL_GetTick() for critical timeout calculations:

```cpp
// Example from sd_diskio.c line 155-156
timeout = HAL_GetTick();
while((ReadStatus == 0) && (SdErrorFlag == 0) && ((HAL_GetTick() - timeout) < SD_TIMEOUT)) {}
```

This creates a **hard dependency** on HAL's tick counter for:
- SD card read operations
- SD card write operations
- Error handling timeouts

## Solution Architecture

### Recommended Approach: Complete Timing System Replacement

#### Phase 1: Replace HAL Timing Functions
```cpp
// In system.cpp
#ifdef FREERTOS_ENABLED

uint32_t System::GetNow()
{
    // Use FreeRTOS tick count instead of HAL
    return xTaskGetTickCount() * portTICK_PERIOD_MS;
}

void System::Delay(uint32_t delay_ms)
{
    // Use FreeRTOS delay instead of HAL
    vTaskDelay(pdMS_TO_TICKS(delay_ms));
}

#else
// Original HAL-based implementations
uint32_t System::GetNow() { return HAL_GetTick(); }
void System::Delay(uint32_t delay_ms) { HAL_Delay(delay_ms); }
#endif
```

#### Phase 2: SD Card Timeout Replacement
```cpp
// In sd_diskio.c
#ifdef FREERTOS_ENABLED

// Replace HAL_GetTick() with FreeRTOS equivalent
timeout = xTaskGetTickCount();
while((ReadStatus == 0) && (SdErrorFlag == 0) 
      && ((xTaskGetTickCount() - timeout) < pdMS_TO_TICKS(SD_TIMEOUT))) {}

#endif
```

#### Phase 3: SysTick Handler Management
```cpp
// In system.cpp
#ifdef FREERTOS_ENABLED
// Don't define SysTick_Handler - let FreeRTOS own it
#else
__weak void SysTick_Handler(void)
{
    HAL_IncTick();
    HAL_SYSTICK_IRQHandler();
}
#endif
```

### Alternative Approach: Timer-Based Replacement

Replace all HAL timing with TIM2-based timing:

```cpp
uint32_t System::GetNow()
{
    // Convert TIM2 ticks to milliseconds
    return tim_.GetTick() / (SystemCoreClock / 1000);
}

void System::Delay(uint32_t delay_ms)
{
    // Use TIM2 for delay instead of SysTick
    uint32_t start = tim_.GetTick();
    uint32_t delay_ticks = delay_ms * (SystemCoreClock / 1000);
    while((tim_.GetTick() - start) < delay_ticks) {
        // Busy wait using TIM2
    }
}
```

## Implementation Roadmap

### Step 1: Add Build Configuration
- Add `FREERTOS_ENABLED` build flag
- Create conditional compilation blocks

### Step 2: Replace Core Timing Functions
- Modify `System::GetNow()`
- Modify `System::Delay()`
- Update `dsy_system_getnow()` and `dsy_system_delay()`

### Step 3: Update SD Card Driver
- Replace `HAL_GetTick()` with appropriate FreeRTOS or TIM2 equivalent
- Test SD card operations thoroughly

### Step 4: SysTick Handler Management
- Conditionally exclude libDaisy's SysTick handler
- Ensure FreeRTOS can define its own handler

### Step 5: Testing
- Verify all timing functions work correctly
- Test SD card operations
- Verify FreeRTOS task scheduling
- Test both FreeRTOS and non-FreeRTOS builds

## Critical Considerations

### 1. Precision Requirements
- `GetNow()` returns milliseconds - FreeRTOS tick precision must be sufficient
- `GetUs()` and `GetTick()` use TIM2 - should remain unaffected

### 2. Blocking vs Non-blocking
- `System::Delay()` is blocking - FreeRTOS version should use `vTaskDelay()`
- This changes the behavior from busy-wait to task yield

### 3. SD Card Performance
- Timeout calculations must remain accurate
- Consider using a high-resolution timer for SD operations

### 4. Backward Compatibility
- Non-FreeRTOS builds must continue to work
- Consider providing both implementations with runtime selection

## Files Requiring Modification

1. **`src/sys/system.cpp`** - Core timing functions and SysTick handler
2. **`src/sys/system.h`** - Function declarations (if signatures change)
3. **`src/util/sd_diskio.c`** - SD card timeout mechanisms
4. **Build system** - Add FREERTOS_ENABLED flag
5. **Possibly** - Create new timer-based implementations

## Conclusion

libDaisy's timing system is more complex than initially apparent, with:
- **Direct SysTick/HAL dependencies** in core timing functions
- **Dual timing architecture** (SysTick + TIM2)
- **External dependencies** in SD card operations
- **Mixed precision requirements** (ms vs μs timing)

The recommended solution involves **conditional compilation** to replace HAL timing functions with FreeRTOS equivalents when FreeRTOS is enabled, while maintaining the independent TIM2-based timing for high-precision operations. This approach provides the cleanest integration path while preserving libDaisy's functionality.