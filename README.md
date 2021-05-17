# Starve-Free-Reader-Writer-Problem

The Reader-Writer Problem is a classical problem in Computer Science in which a data structure like database, storage area, etc is being accessed simultaneously by multiple processes concurrently. The critcal section is such that it can be accessed by only one one writer at any point of time while multiple readers can simultaneously access the critical section. We use semaphores to achieve this such that there would be no conflicts while accessing by writers and readers and process synchronization is properly met. But, this may give rise to starvation of either the readers or the writers, depending on their priorities. I have described a solution to the Reader-Writer problem that is starve-free and tackles the problem efficiently.

Firstly, I have decribed the classical solution followed by the starve-free solution to the same.

## Data Structures Used : 

Semaphores are used to solve the problem of process synchronization. The semaphore is linked to a critical section and contains a Queue (FIFO structure) that stores the list of processes that are blocked and waiting to acquire the semaphore. While entering the `blockedQueue` the process is blocked and once a process signals a semaphore, the process in the front of the `blockedQueue` gets activated. The following code describes the implementation of the same.

```cpp
// SEMAPHORE //

struct process {  //    an analogous of the process control block
    process* next;
    int ID;
    bool state = true; 
    //  state represents whether the process is blocked or active
    //  true represents active while false represents inactive(blocked)

    //  also other things maybe present here, like the return address, subprocesses, etc
};

class blockedQueue {
    int size = 0;
    int MAX_SIZE = 100;
    // max number of waiting processes

    process *front, *back;

    public:

        void push(int id) { 
        //  I have made this function in such a way that it blocks the entering process as well

            if(size == MAX_SIZE){
                return;
            }

            process* P;
            P->ID = id;
            P->state = false;   
            //  the process is blocked before pushing into the blockedQueue
            //  in reality this is done using system calls
            
            ++size;

            if(size == 0) {
                front = P;
                back = P;
                return;
            }        
            
            back->next = P;
            back = P;
        }

        process nullprocess = {nullptr,-1,false};   
        //  null process to be returned when the queue is empty
        
        process* pop() {
            if(size == 0){
                return &nullprocess;
            }

            process* nextProcess = front;
            front = front->next;
            --size;

            return nextProcess;
        }

};

struct semaphore {
//  structure of the semaphore, as you can see we have linked it to a queue
//  this queue would store the list of proessess waiting to acquire the semaphore
    int value = 1;
    blockedQueue* blocked_queue = new blockedQueue();
    
    semaphore(int n) {
        value = n;
        //  here n is the number of resources of that type available
    }

};

void wakeUp(process* p) {
    p->state = true;
}
//  in reality a system call is used instead of this function
//  i have used this function as an analogy of the syscall

void signal(semaphore* sem) {
    ++sem->value;

    if(sem->value <= 0) {
        process* nextProcess = sem->blocked_queue->pop();
        wakeUp(nextProcess);    
        //  wakeUp the next process such that it's ready for entering critical section
    }
}

void wait(semaphore *sem, int id) {
    --sem->value;

    if(sem->value < 0) {
        sem->blocked_queue->push(id);
    }
}

```

### Initial values of Semaphores

The classical solution uses two semaphores `read_mutex` and `rw_mutex` for the Reader-Writer problem. I would be using the same semaphores described below along with some more for the starve-free approach.

```cpp
// INITIALIZATION //

int resources = 1;
//  number of resources the processes are competing for
//  in case of reader writer's problem since they are are accessing only one critical section
//  thus, the number of resources is 1

semaphore* rw_mutex = new semaphore(resources);
//  this is the main semaphore
//  a writer with this semaphore has access to the critical section
//  also the first reader has to acquire this and the last reader has to let go of this

int read_count = 0; //  represents the number of readers in the critical section

semaphore* read_mutex = new semaphore(resources);
//  this semaphore is used to access the read_count variable
//  this variable prevents the conflicts while changing the read_count variable
//  conflicts might have arose when two readers access the variable simulataneously

```

## The Classical Solution

In the classical solution we use two semaphores `read_mutex` and the `rw_mutex` and one variable `read_count`. The `read_count` variable represents the number of readers currently accessing the critical section. The `read_mutex` is a used to write-access to the `read_count` variable. The any reader that enters or exits the critical section has to acquire this `read_mutex`. The `rw_mutex` is used to gain access to the critical section by both the readers and the writers. It has to be obtained by every writer before entering the critical section and signalled every time it exits the section, but only the first reader has to acquire this semaphore to enter the critcal section. The last reader has to signal the `rw_mutex` to show that it is free and available to writers now.

### Reader Implementation (Classical)
Following is the code for reader in classical solution. I have made a function for modularising the implementation. This function would be called in a when any new process asking for access to the critical section arrives. As you can see that the 
```cpp
//  READER'S CODE
void classicalReader(int processId) {
    // this function would be called everytime a new reader arrives

    wait(read_mutex, processId);
    //  the next section executes if the process with processId is not blocked

    ++read_count;
    
    if(read_count == 1) {   // this implies that the first reader tries to access
        
        wait(rw_mutex, processId); 
        //  this will wait until the process is activated 
        //  due to freeing of the rw_mutex by a signal from writer 

    }

    signal(read_mutex); // this call happens here, as unless a reader enters, others shouldn't modify the read_count

        // ***** CRITICAL SECTION ***** //

        //  this can be accessed by readers directly if read_count != 1 (they are not the first reader)
        //  also after the first reader has acquired the rw_mutex

    wait(read_mutex, processId);
    --read_count;
    signal(read_mutex);

    if(read_count == 0){
        signal(rw_mutex);
        // the last reader frees the rw_mutex
    }

}

```
### Writer Implementation (Classical)
The writer code that would be called every time a new writer arrives. 
```cpp
//  WRITER'S CODE
void classicalWriter(int processId) {
    
    wait(rw_mutex, processId);
    //  the writer process would be blocked while waiting
    //  once free it will acquire the rw_mutex and enter the critical section

        // ***** CRITICAL SECTION ***** //

        //  the critical section can be accessed by a writer only if they have acquired the rw_mutex

    signal(rw_mutex);
}

```
As you can see the problem with the above implementation is that the writer's may starve while waiting for access to the critical section. This would happen if readers keep coming one after other, thus the writers would never get a chance to acquire the `rw_mutex`, thus being starved. This can be tackled by using another semaphore, which I call `entry_mutex`. The starve-free solution below describes how the issue can be solved. 

## Starve-Free Solution

In the starve-free solution, the `entry_mutex` semaphore is used in addition to the `read_mutex` and `write_mutex` semaphore. This is first required to be obained before anyone (the reader or the writer) accesses the `rw_mutex` or before anyone (any reader) enters the critcal section directly. This solves the problem of starvation as if readers keep coming one after another, then this won't starve the writers as it used to above. Here, if a writer comes in between two readers, and even if some readers are still present in the critical section, the next reader if it comes after a writer, the writer would have already acquired the `entry_mutex` and thus the reader can't acquire it and thus after the existing readers exit the critical section, the writer that was waiting would be at front of the `blockedQueue` for the `rw_mutex` and thus acquires it. Now the writer can enter the critical section and the same process would repeat. Thus readers and writers are now at equal priority and none would starve. Doing this also preserves the advantage of readers not having to acquire the `rw_mutex` everytime, when some other reader is already present. Thus an effiecient and starve-free solution to the Reader-Writer problem.

The following is the code for new semaphores and the reader and writer.

### Semaphores and initialization

```cpp
// INITIALIZATION //

int resources = 1;

semaphore* rw_mutex = new semaphore(resources);

int read_count = 0; 

semaphore* read_mutex = new semaphore(resources);

//  The above three data structures are the same as above in the classical solution

semaphore* entry_mutex = new semaphore(resources); // The new semaphore

//  This semaphore is used at the begining of both the reader and writer codes
//  A reader/writer first has to acquire this semaphore in order to enter the critical section
//  Both the reader and writer have equal priority for obtaining this semaphore

```
### Reader Implementation (Starve-Free)
The following is the implementation of the reader code using the starve-free approach.

```cpp
//  STARVE-FREE READER CODE
void starveFreeReader(int processId) {
    // this function would be called everytime a new reader arrives
    
    wait(entry_mutex, processId);
    //  the next section executes if the process with processId is not blocked by this semaphore
    //  this is same for both readers and writers
    
    wait(read_mutex, processId);


    ++read_count;

    if(read_count == 1) {   // this implies that the first reader tries to access
        
        wait(rw_mutex, processId); 
    }
        
    signal(read_mutex); // now readers that have acquired the entry_mutex can access the read_count

    signal(entry_mutex);
    //  Once the reader is ready to enter the critical section, it frees the entry_mutex semaphore
    //  This entry_mutex semaphore can be acquired by either a reader or writer, whichever arrives first

        // ***** CRITICAL SECTION ***** //

        //  this can be accessed by readers directly if read_count != 1 (they are not the first reader)
        //  also after the first reader has acquired the rw_mutex

    wait(read_mutex, processId);
    --read_count;
    signal(read_mutex);

    if(read_count == 0){
        signal(rw_mutex);
        // the last reader frees the rw_mutex
    }

}

```

### Writer Implementation (Starve-Free)
The following is the implementation of the reader code using the starve-free approach.

```cpp
//  STARVE-FREE WRITER CODE
void classicalWriter(int processId) {
    
    wait(entry_mutex, processId);
    //  the writer also waits for the entry mutex first
    //  it can acquire this mutex even if a reader is already present in critical section

    wait(rw_mutex, processId);
    //  the writer process would be blocked while waiting
    //  once free it will acquire the rw_mutex and enter the critical section

    signal(entry_mutex);
    //  once the writer is ready to enter the critical section it frees the entry_mutex
    //  this can now be acquired by any reader or writer which came first

        // ***** CRITICAL SECTION ***** //

        //  the critical section can be accessed by a writer only if they have acquired the rw_mutex

    signal(rw_mutex);
}
```
As you can see above the starvation problem has been taken care of and neither the readers nor the writers would starve. Thus, a starve-free approach to the Reader-Writer problem.


## References

1. *Operating System Concepts*, Ninth Edition, Silberschatz, Galvin, Gagne 
2. [Cornell University Paper on Reader-Writer Problem](https://arxiv.org/abs/1309.4507)


