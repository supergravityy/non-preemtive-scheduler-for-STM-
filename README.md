# 1. Overview

This project is a **non-preemptive, period-based scheduler for ST microcontrollers**, designed with a primary focus on **predictability and system safety**.

If you would like to know more about this scheduler, please visit the '[site](https://velog.io/@smersh/%EB%B9%84%EC%84%A0%EC%A0%90%ED%98%95-%EC%8A%A4%EC%BC%80%EC%A4%84%EB%9F%AC-%E6%94%B9)'.
(Written in Korean)

Rather than simply executing tasks at fixed intervals, the scheduler is intended to **detect, diagnose, and respond to delays and exceptional conditions** that commonly arise in real-time embedded systems.

The core philosophy of this scheduler is not to eliminate all timing violations, but to **make such violations observable, traceable, and recoverable**.

## Features

### Deterministic Multi-Layer Diagnosis

Beyond basic task execution, the scheduler continuously tracks the overall health of the system.

- **Dual-Level Overrun Tracking**
    
    Overrun events are accumulated per task at two levels:
    
    - within the current major cycle, and
    - across the total system runtime.
    
    This separation allows transient load spikes to be clearly distinguished from persistent structural bottlenecks.
    

---

### Fail-Safe Oriented Protection

The scheduler includes optional protection mechanisms designed to prevent the system from entering an uncontrollable state.

- **Threshold-Based Reset**
    
    When the accumulated overrun count exceeds the configured threshold (`THRESHOLD_CNT`), the system deliberately transitions into a controlled reset sequence to avoid undefined behavior.
    
- **Watchdog Synchronization**
    
    The watchdog is refreshed **only after all registered tasks have executed successfully**.
    
    This ensures a reliable hardware reset in the presence of task deadlocks, infinite loops, or logical faults.
    

---

### Hook System

The internal state of a running system can be observed in real time **without halting execution using a debugger**.

- **Timing-Preserving Tracing**
    
    User-defined hook functions allow system state information to be emitted (e.g., via UART) at well-defined execution points, enabling runtime observation without disturbing system timing through breakpoints.
    
- **Lifecycle-Level Control**
    
    User logic can be injected at key lifecycle events—such as initialization completion, error detection, or major cycle transitions—allowing system-level behavior to be customized without modifying the scheduler core.
    

---

### Non-Invasive Weak-Symbol Extension

Scheduler behavior can be extended without modifying internal library code.

- **Isolated User Code**
    
    A callback-based design using `__weak` symbols allows user-defined handlers (logging, emergency stop routines, status indicators, etc.) to be attached while keeping the core engine intact.
    
- **Flexible Exception Handling**
    
    The provided interfaces make it straightforward to implement hardware-specific responses, such as LED indicators or emergency data backup routines, tailored to the target system.
    

---

### Flexible Macro-Based Customization

Scheduler functionality can be selectively enabled or disabled to match system constraints and available resources.

- **Feature Selection**
    
    Core features—such as overrun detection, watchdog feeding, and cycle measurement—can be individually controlled using `#define` options in the configuration header.
    
- **Overhead Control**
    
    Unused features can be disabled to minimize runtime overhead, leaving only the logic required for the target system.
# 2. Supported Environment

The following system configuration is recommended for stable operation.

| Name | Description | Required |
| --- | --- | --- |
| MCU | ST microcontroller | Yes |
| System Clock | 72 MHz | Yes |
| Timer | 1 ms periodic interrupt | Yes |
| IWDG | System recovery when out of control | Optional |
| GPIO | Major cycle timing measurement | Optional |

---

# 3. Quick Start

This section describes the minimum steps required to integrate the scheduler into a project.

## a. Scheduler Option Configuration

Configure the required options in `tasksch_config.h`.

```c
#define TASKSCH_WATCHDOG_ENABLE             (1)
#define TASKSCH_WATCHDOG_DISABLE            (0)
#define TASKSCH_TASK_WATCHDOG               (TASKSCH_WATCHDOG_ENABLE)// User-defined

#if (TASKSCH_TASK_WATCHDOG == TASKSCH_WATCHDOG_ENABLE)
// User-defined watchdog parameters
// NOTE: LSI frequency may vary by up to ±20%; allow sufficient margin.
#define TASKSCH_WATCHDOG_CLK_FREQ_HZ    (40000U)
#define TASKSCH_WATCHDOG_PRESCALER      (256U)
#define TASKSCH_WATCHDOG_PRSCLAER_BITS  (IWDG_PRESCALER_256)
#define TASKSCH_WATCHDOG_TIMEOUT_CNT    (2000U)

// Derived macros (do not modify)
#define TASKSCH_WATCHDOG_1COUNT_MS      (TASKSCH_WATCHDOG_PRESCALER / TASKSCH_WATCHDOG_CLK_FREQ_HZ)
#define TASKSCH_WATCHDOG_TIMEOUT_MS     (TASKSCH_WATCHDOG_TIMEOUT_CNT * TASKSCH_WATCHDOG_1COUNT_MS)
#endif
```

---

## b. Task Definition and Registration

Define application tasks in `tasksch_config.c` and register them using `tasksch_init_RegiTaskObj`.

```c
voidUserTask_1ms(void)  {/* task logic */ }
voidUserTask_10ms(void) {/* task logic */ }

voidtasksch_init_RegiTaskObj(void)
{
    vUserRegiTaskObj[0].regiTaskFunc_ptr = UserTask_1ms;
    vUserRegiTaskObj[0].regiTaskPeriod_ms =1;
    vUserRegiTaskObj[0].regiTaskOffset_ms =0;

    vUserRegiTaskObj[1].regiTaskFunc_ptr = UserTask_10ms;
    vUserRegiTaskObj[1].regiTaskPeriod_ms =10;
    vUserRegiTaskObj[1].regiTaskOffset_ms =5;
}
```

---

## c. Timer Interrupt Integration

Call the scheduler time manager from a 1 ms timer ISR.

```c
voidTIM1_UP_IRQHandler(void)
{
    tasksch_timeManager();
}
```

---

## d. Starting the Scheduler

```c
intmain(void)
{
    tasksch_init();

while (1)
    {
        tasksch_execTask();
    }
}
```

---

## e. System State Monitoring

System-level status can be monitored by implementing `tasksch_userMajorCycleHook`.

```c
voidtasksch_userMajorCycleHook(void)
{
printf("Total Overrun Count: %d\n", tasksch_getOverRunCount());
}
```
