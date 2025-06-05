### Synchronization Problem Suite  
**Each problem is tailored to highlight a specific mechanism's strengths.**  

---

#### **1. Peterson's Solution**  
**Problem**: Two processes (P0 and P1) share a variable `counter=0`. Each must increment it 100 times.  
**Why Peterson?** Educational context (2-process mutual exclusion).  
```c
// Shared
int counter = 0;
bool flag[2] = {false, false};
int turn;

// Process i (i=0 or 1)
void process(int i) {
    for (int j=0; j<100; j++) {
        flag[i] = true;               // Declare interest
        turn = 1-i;                   // Yield to other
        while (flag[1-i] && turn==1-i); // Wait
        
        counter++;                    // Critical section
        
        flag[i] = false;              // Exit
    }
}
// Final counter = 200 ✅
```

---

#### **2. Test-and-Set (TAS)**  
**Problem**: 10 threads increment `counter=0` 100 times each. Fast, simple locking needed.  
**Why TAS?** Simple spinlock for short critical sections.  
```c
atomic_bool lock = false; // Shared lock

void thread_func() {
    for (int i=0; i<100; i++) {
        while (test_and_set(&lock)); // Spin until acquired
        counter++;                   // Critical section (fast)
        lock = false;                // Release
    }
}
// Final counter = 1000 ✅
```

---

#### **3. Compare-and-Swap (CAS)**  
**Problem**: Atomic counter that resets to 0 when reaching 50.  
**Why CAS?** Complex atomic condition ("increment if <50, else reset").  
```c
atomic_int counter = 0; // Shared

void increment() {
    int old_val, new_val;
    do {
        old_val = counter;
        new_val = (old_val < 50) ? old_val+1 : 0;
    } while (!compare_and_swap(&counter, old_val, new_val));
}
// Thread-safe resetting counter ✅
```

---

#### **4. Mutex Lock**  
**Problem**: Two threads write to a shared file.  
**Why Mutex?** Simple critical section (no conditions).  
```c
pthread_mutex_t file_mutex; // Shared mutex

void write_to_file(char* data) {
    pthread_mutex_lock(&file_mutex);
    // Critical: Append data to file
    pthread_mutex_unlock(&file_mutex);
}
// No interleaved writes ✅
```

---

#### **5. Semaphore**  
**Problem**: Producer-consumer with 5-slot buffer.  
**Why Semaphore?** Resource counting + waiting queues.  
```c
sem_t empty(5), full(0); // Counting semaphores
sem_t mutex(1);          // Binary semaphore

void producer() {
    while (true) {
        item = produce();
        sem_wait(&empty); // Wait for slot
        sem_wait(&mutex);
        buffer.add(item); // Critical
        sem_post(&mutex);
        sem_post(&full); // Signal new item
    }
}

void consumer() {
    sem_wait(&full);     // Wait for item
    sem_wait(&mutex);
    item = buffer.remove(); // Critical
    sem_post(&mutex);
    sem_post(&empty);
    consume(item);
}
// No overflow/underflow ✅
```

---

#### **6. Monitor**  
**Problem**: Thread-safe bank account with overdraft prevention.  
**Why Monitor?** Encapsulation + condition variables.  
```java
monitor BankAccount {
    private int balance = 0;
    condition sufficientFunds;

    void deposit(int amount) {
        balance += amount;
        signalAll(sufficientFunds); // Notify waiters
    }

    void withdraw(int amount) {
        while (balance < amount) 
            wait(sufficientFunds); // Auto-release lock
        balance -= amount;
    }
}
// No overdrafts + clean API ✅
```

---

### Key Takeaways:  
| **Mechanism**       | **Best When...**                          | **Avoid When...**               |  
|----------------------|-------------------------------------------|---------------------------------|  
| `Peterson's`         | Teaching 2-process synchronization        | Modern systems (>2 processes)   |  
| `Test-and-Set`       | Simple spinlocks (short waits)            | Long critical sections          |  
| `Compare-and-Swap`   | Lock-free conditions (e.g., "set if X")   | Simple locks (overkill)         |  
| `Mutex Lock`         | Basic critical sections (no conditions)   | Resource pools                  |  
| `Semaphore`          | Resource counting (buffers, instances)    | Complex OOP data structures     |  
| `Monitor`            | Encapsulated shared objects (OOP)         | Low-level kernel programming    |  

> 💡 **Rule of Thumb**:  
> - Use **high-level** (Monitor/Semaphore) for apps  
> - Use **low-level** (TAS/CAS) for OS/kernel  
> - Use **Peterson** only for education