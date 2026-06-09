# L476-FRTOS3-ShareAnUART
# STM32 CMSIS-RTOS2 Multi-Tasking UART Demo

## Overview

This project demonstrates the use of **CMSIS-RTOS2 (FreeRTOS)** on an STM32 microcontroller. Two independent tasks are created and scheduled by the RTOS. Each task periodically sends a message through UART2 to a serial terminal.

The project is intended as a basic example for learning:

* CMSIS-RTOS2 thread creation
* Task scheduling
* UART communication
* Periodic task execution using `osDelay()`
* Shared resource protection using Mutex

---

## Features

* Initializes STM32 hardware and UART2.
* Creates two RTOS tasks:

  * Task 1
  * Task 2
* Both tasks execute indefinitely.
* Messages are transmitted to a serial terminal via UART2.
* Demonstrates round-robin scheduling when tasks have the same priority.
* Demonstrates UART resource contention when no Mutex is used.
* Shows how a Mutex can prevent race conditions.

---

## Hardware Requirements

* STM32 Nucleo Board (or compatible STM32 board)
* USB cable
* Serial terminal software:

  * Tera Term
  * PuTTY
  * STM32CubeMonitor
  * Arduino Serial Monitor

---

## Software Requirements

* STM32CubeIDE
* STM32 HAL Drivers
* CMSIS-RTOS2
* FreeRTOS

---

## Project Structure

### UART Output Redirection

The standard `printf()` function is redirected to UART2.

```c
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```

### Message Printing Function

```c
void print_msg(int i)
{
    printf("Task %d\n\r", i);
}
```

---

## RTOS Tasks

### Task 1

```c
void StartTask01(void *argument)
{
    for(;;)
    {
        print_msg(1);
        osDelay(5000);
    }
}
```

Behavior:

* Prints "Task 1"
* Waits 5000 RTOS ticks
* Repeats forever

---

### Task 2

```c
void StartTask02(void *argument)
{
    for(;;)
    {
        print_msg(2);
        osDelay(5000);
    }
}
```

Behavior:

* Prints "Task 2"
* Waits 5000 RTOS ticks
* Repeats forever

---

## Task Configuration

Both tasks are configured with the same priority:

```c
.priority = osPriorityNormal
```

Threads are created as follows:

```c
myTask01Handle = osThreadNew(StartTask01, NULL, &myTask01_attributes);
myTask02Handle = osThreadNew(StartTask02, NULL, &myTask02_attributes);
```

Because both tasks have equal priority, the scheduler shares CPU time between them.

---

## UART Configuration

| Parameter    | Value |
| ------------ | ----- |
| Baud Rate    | 9600  |
| Data Bits    | 8     |
| Stop Bits    | 1     |
| Parity       | None  |
| Flow Control | None  |

---

## Expected Serial Output

After startup:

```text
This is a test message before FreeRTOS is running.

Task 1
Task 2
Task 1
Task 2
Task 1
Task 2
...
```

The exact ordering may vary depending on scheduling and timing.

---

# Demonstration of UART Resource Contention (Without Mutex)

## Purpose

This experiment demonstrates what happens when multiple RTOS tasks access a shared UART peripheral without synchronization.

Both Task 1 and Task 2 use `printf()` to transmit data through UART2. Since UART is a shared hardware resource, simultaneous access from multiple tasks may cause output corruption.

---

## Observed Output

The serial terminal produced output similar to:

```text
-Task
    1
Tk 1
    2
Task
    1
Tk 1
    2
```

---

## Expected Output

The intended output was:

```text
Task 1
Task 2
Task 1
Task 2
Task 1
Task 2
```

---

## Cause of the Problem

Without a mutex, both tasks can access the UART peripheral at nearly the same time.

A possible execution sequence is:

1. Task 1 starts transmitting `"Task 1"`.
2. The RTOS scheduler switches to Task 2.
3. Task 2 starts transmitting `"Task 2"`.
4. Characters from both tasks become interleaved.
5. The terminal displays corrupted output.

For example:

```text
Task 1 → T a s
Task 2 → T a s k 2
Task 1 → k 1
```

Result:

```text
TasTask 2k 1
```

This behavior is a classic **race condition** caused by unsynchronized access to a shared resource.

---

## Why UART Requires Protection

UART hardware can process only one transmission stream at a time.

When multiple tasks attempt to transmit simultaneously, data may become mixed together.

Common shared resources that require synchronization include:

* UART peripherals
* SPI buses
* I2C buses
* Shared memory
* File systems
* Communication buffers

---

## Solution: Using a Mutex

A mutex ensures that only one task can access the UART at a time.

### Create a Mutex

```c
osMutexId_t uartMutex;

uartMutex = osMutexNew(NULL);
```

### Protect UART Access

```c
osMutexAcquire(uartMutex, osWaitForever);

print_msg(1);

osMutexRelease(uartMutex);
```

Task 2 should use the same protection:

```c
osMutexAcquire(uartMutex, osWaitForever);

print_msg(2);

osMutexRelease(uartMutex);
```

---

## Expected Result with Mutex

After protecting UART access with a mutex:

```text
Task 1
Task 2
Task 1
Task 2
Task 1
Task 2
```

Each task completes its entire UART transmission before another task can access the UART.

---

## Learning Objectives

This project demonstrates:

### Thread Creation

```c
osThreadNew()
```

### Periodic Task Execution

```c
osDelay()
```

### UART Debugging

```c
printf()
```

### RTOS Scheduling

Multiple tasks running concurrently under FreeRTOS.

### Shared Resource Protection

```c
osMutexAcquire()
osMutexRelease()
```

### Race Condition Analysis

Understanding how unsynchronized access to shared resources can cause data corruption.

---

## Conclusion

This project demonstrates the fundamentals of CMSIS-RTOS2 task management on STM32 and highlights the importance of synchronization when multiple tasks share hardware resources.

The experiment clearly shows that:

* Multiple tasks can run concurrently under FreeRTOS.
* UART is a shared resource.
* Unsynchronized UART access may lead to corrupted output.
* Mutexes prevent race conditions and ensure reliable communication.

By adding synchronization mechanisms such as Mutexes, developers can build robust and predictable embedded real-time applications.

