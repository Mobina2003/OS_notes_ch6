
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