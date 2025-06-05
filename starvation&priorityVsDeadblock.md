
### 1. Starvation (Indefinite Blocking)
**Definition**: When a thread is perpetually denied access to resources it needs, even though the resources are available. It remains queued indefinitely while others are served.

**Real-World Example**:  
Imagine a hospital triage system:
- **Critical patients** (high-priority) always jump the queue
- **Stable patients** (low-priority) keep getting pushed back
- One stable patient waits 12 hours while new critical cases arrive continuously  
*Result*: The stable patient is "starved" of medical attention despite resources being available.

**Code Example** (C):
```c
#include <pthread.h>
#include <unistd.h>

pthread_mutex_t resource_lock = PTHREAD_MUTEX_INITIALIZER;

void* high_priority_thread(void* arg) {
    while(1) {
        pthread_mutex_lock(&resource_lock);
        printf("High-priority thread using resource\n");
        usleep(100000); // Hold resource for 100ms
        pthread_mutex_unlock(&resource_lock);
    }
}

void* low_priority_thread(void* arg) {
    while(1) {
        pthread_mutex_lock(&resource_lock);
        printf("Low-priority thread FINALLY accessed resource\n");
        pthread_mutex_unlock(&resource_lock);
        sleep(1);
    }
}

int main() {
    pthread_t hi_prio, lo_prio;
    
    // Set thread priorities (real code would use pthread_setschedparam)
    pthread_create(&hi_prio, NULL, high_priority_thread, NULL);
    sleep(1); // Ensure high-priority starts first
    pthread_create(&lo_prio, NULL, low_priority_thread, NULL);

    pthread_join(hi_prio, NULL);
    pthread_join(lo_prio, NULL);
}
```

**Behavior**:
- High-priority thread constantly reacquires the lock
- Low-priority thread rarely (if ever) gets access
- Output shows endless "High-priority..." messages

**Solution**:  
Use fair scheduling mechanisms like:
```c
// Replace mutex with:
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);  // Priority inheritance
pthread_mutex_init(&resource_lock, &attr);
```

---

### 2. Priority Inversion
**Definition**: When a medium-priority thread preempts a low-priority thread that's holding a resource needed by a high-priority thread, causing the high-priority thread to wait unnecessarily.

**Real-World Example (Mars Pathfinder Incident)**:
1. **High-priority**: Communications task (critical)
2. **Medium-priority**: Data processing task
3. **Low-priority**: Weather monitoring task (holds shared bus)
- *Scenario*:  
  - Low-priority task acquires bus  
  - High-priority task needs bus → blocks  
  - Medium-priority task preempts low-priority task  
  - High-priority task waits for medium-priority to finish  
- *Result*: System reset due to watchdog timeout (actual NASA failure)

**Code Example** (C):
```c
#include <pthread.h>
#include <unistd.h>

pthread_mutex_t resource_lock = PTHREAD_MUTEX_INITIALIZER;

void* low_priority_thread(void* arg) {
    pthread_mutex_lock(&resource_lock);
    printf("Low-priority thread acquired resource\n");
    sleep(2);  // Intentionally long hold time
    pthread_mutex_unlock(&resource_lock);
    return NULL;
}

void* medium_priority_thread(void* arg) {
    printf("Medium-priority thread running\n");
    sleep(10); // CPU-intensive work
    return NULL;
}

void* high_priority_thread(void* arg) {
    usleep(100000); // Ensure low-priority starts first
    printf("High-priority thread waiting for resource\n");
    pthread_mutex_lock(&resource_lock); // Blocked here!
    printf("High-priority thread acquired resource\n");
    pthread_mutex_unlock(&resource_lock);
    return NULL;
}

int main() {
    pthread_t low, med, high;
    
    pthread_create(&low, NULL, low_priority_thread, NULL);
    usleep(50000);
    pthread_create(&high, NULL, high_priority_thread, NULL);
    pthread_create(&med, NULL, medium_priority_thread, NULL);
    
    pthread_join(low, NULL);
    pthread_join(med, NULL);
    pthread_join(high, NULL);
}
```

**Behavior**:
1. Low-priority thread locks resource
2. High-priority thread blocks on lock
3. Medium-priority thread preempts low-priority thread
4. High-priority waits for medium-priority to finish first

**Solution: Priority Inheritance Protocol**  
```c
// Initialize mutex with priority inheritance:
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&resource_lock, &attr);
```
*Effect*: When high-priority thread waits, low-priority thread temporarily inherits high priority, preventing medium-priority preemption.


### Priority Inheritance Implementation for example
a complete program demonstrating priority inversion and its solution using priority inheritance. The code creates three threads with different priorities and shows how priority inheritance prevents high-priority thread starvation:

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <sched.h>
#include <errno.h>

pthread_mutex_t resource_lock;

// Thread function prototype
void* thread_function(void* arg);

// Configure mutex with priority inheritance
void configure_mutex(int use_priority_inheritance) {
    if (use_priority_inheritance) {
        pthread_mutexattr_t attr;
        pthread_mutexattr_init(&attr);
        pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
        pthread_mutex_init(&resource_lock, &attr);
        printf("\nMutex configured WITH PRIORITY INHERITANCE\n");
    } else {
        pthread_mutex_init(&resource_lock, NULL);
        printf("\nMutex configured WITHOUT PRIORITY INHERITANCE\n");
    }
}

int main() {
    // Create thread attributes
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
    pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
    
    // Create thread parameters
    struct sched_param low_param = {.sched_priority = 10};
    struct sched_param med_param = {.sched_priority = 50};
    struct sched_param high_param = {.sched_priority = 99};
    
    pthread_t low_thread, med_thread, high_thread;
    
    // Run without priority inheritance
    configure_mutex(0);
    create_threads(&low_thread, &med_thread, &high_thread, &attr, &low_param, &med_param, &high_param);
    pthread_join(low_thread, NULL);
    pthread_join(med_thread, NULL);
    pthread_join(high_thread, NULL);
    
    // Run with priority inheritance
    configure_mutex(1);
    create_threads(&low_thread, &med_thread, &high_thread, &attr, &low_param, &med_param, &high_param);
    pthread_join(low_thread, NULL);
    pthread_join(med_thread, NULL);
    pthread_join(high_thread, NULL);

    pthread_attr_destroy(&attr);
    pthread_mutex_destroy(&resource_lock);
    return 0;
}

// Create and start threads with given priorities
void create_threads(pthread_t *low, pthread_t *med, pthread_t *high, 
                   pthread_attr_t *attr, struct sched_param *low_param, 
                   struct sched_param *med_param, struct sched_param *high_param) {
    pthread_attr_setschedparam(attr, low_param);
    pthread_create(low, attr, thread_function, "LOW   ");
    
    usleep(10000); // Small delay
    
    pthread_attr_setschedparam(attr, high_param);
    pthread_create(high, attr, thread_function, "HIGH  ");
    
    usleep(10000); // Small delay
    
    pthread_attr_setschedparam(attr, med_param);
    pthread_create(med, attr, thread_function, "MEDIUM");
}

// Thread work function
void* thread_function(void* arg) {
    const char* name = (const char*)arg;
    
    if (name[0] == 'L') {  // Low-priority thread
        printf("[%s] LOCKING resource\n", name);
        pthread_mutex_lock(&resource_lock);
        printf("[%s] WORKING with resource (should finish quickly with PI)\n", name);
        sleep(2);  // Simulate work
        pthread_mutex_unlock(&resource_lock);
        printf("[%s] UNLOCKED resource\n", name);
    }
    else if (name[0] == 'H') {  // High-priority thread
        usleep(100000); // Ensure low-priority locks first
        printf("[%s] WAITING for resource\n", name);
        pthread_mutex_lock(&resource_lock);
        printf("[%s] ACQUIRED resource (should happen faster with PI)\n", name);
        pthread_mutex_unlock(&resource_lock);
    }
    else {  // Medium-priority thread
        usleep(150000); // Start after high-priority is blocked
        printf("[%s] DOING non-critical work\n", name);
        sleep(3);  // Simulate CPU-intensive work
        printf("[%s] FINISHED work\n", name);
    }
    return NULL;
}
```

### Key Components Explained:

1. **Priority Inheritance Configuration**:
```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&resource_lock, &attr);
```
This creates a mutex that automatically boosts the priority of any thread holding it when a higher-priority thread starts waiting.

2. **Thread Priority Setup**:
```c
struct sched_param high_param = {.sched_priority = 99};
pthread_attr_setschedparam(&attr, &high_param);
```
Sets real-time priorities (1-99, higher = more priority) using `SCHED_FIFO` policy.

3. **Demonstration Workflow**:
- First run: Without priority inheritance
- Second run: With priority inheritance
- Each run creates three threads:
  - LOW: Locks resource and works
  - HIGH: Waits for same resource
  - MEDIUM: Does CPU-intensive work

### Expected Output:
```
Mutex configured WITHOUT PRIORITY INHERITANCE
[LOW   ] LOCKING resource
[LOW   ] WORKING with resource...
[HIGH  ] WAITING for resource
[MEDIUM] DOING non-critical work
[MEDIUM] FINISHED work
[LOW   ] UNLOCKED resource
[HIGH  ] ACQUIRED resource

Mutex configured WITH PRIORITY INHERITANCE
[LOW   ] LOCKING resource
[LOW   ] WORKING with resource...
[HIGH  ] WAITING for resource
[LOW   ] UNLOCKED resource   # Finishes BEFORE medium starts!
[HIGH  ] ACQUIRED resource
[MEDIUM] DOING non-critical work
[MEDIUM] FINISHED work
```

### How to Run:
1. Compile:
```bash
gcc -o priority_demo priority_demo.c -lpthread
```



### Key Differences Shown:
1. **Without Priority Inheritance**:
   - Medium-priority thread runs while low holds lock
   - High-priority thread starves until both finish
   - Execution order: Low → Medium → High

2. **With Priority Inheritance**:
   - Low-priority thread temporarily gets high priority
   - Medium-priority can't preempt
   - High-priority gets resource immediately after low finishes
   - Execution order: Low → High → Medium



---

### Comparison: Starvation vs Priority Inversion vs Deadlock

| Characteristic       | Deadlock               | Starvation               | Priority Inversion         |
|----------------------|------------------------|--------------------------|----------------------------|
| **Core Issue**       | Circular dependency    | Unfair resource access   | Priority scheduling flaw   |
| **Progress**         | All threads blocked    | One thread blocked       | High-priority blocked      |
| **Duration**         | Permanent              | Potentially indefinite   | Temporary                  |
| **Required Conditions** | 4 conditions (mutex, hold/wait, etc) | Priority imbalance | Priority system + shared resource |
| **Real-World Analogy** | Traffic gridlock      | Never-called customer    | 911 call blocked by spam   |
| **Solution Example** | Lock ordering          | Fair scheduling          | Priority inheritance       |
| **Code Fix**         | Consistent lock order  | pthread_mutexattr_setprioceiling() | PTHREAD_PRIO_INHERIT |

---

### Key Differences:
1. **Nature of Blockage**  
   - Deadlock: *All* participants are frozen
   - Starvation: *Some* threads progress while others don't
   - Priority Inversion: *High-priority* threads wait for lower ones

2. **Resolution Difficulty**  
   - Deadlock: Requires system intervention
   - Starvation: Needs policy change
   - Priority Inversion: Fixed via OS mechanisms

3. **System State**  
   - Deadlock: Entire system stuck
   - Starvation: Partial functionality remains
   - Priority Inversion: System works but violates priority guarantees

**Critical Insight**: Priority inversion is often the *cause* of starvation in priority-based systems, while deadlock is a separate systemic failure. All three violate liveness but through different mechanisms.