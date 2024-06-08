## Introduction
To process multiple tasks in parallel, dataset is distributed across multiple machines for processing, and results are merged. Also, tasks fail for many reasons e.g. machine crash, network loss. To achieve fault tolerance, system should recover from these failures.

## MapReduce ##
- Map function
    - input: key/value
        - e.g. filename/content
    - output: generate intermediate key/value pairs which are stored in the corresponding partition
- Reduce function
    - input: intermediate key/value pairs for a given key
    - output: merge the value for an intermediate key and store the results in a reduce partition file.
- Tasks are assigned by a coordinator
    - It simplifies the design. Worker doesn't have to remember the status of other works by over-talking on the network. Only coordinator knows status of all the workers.
- Locality: worker reads input file from nearby machine
    - Network bandwidth is more likely the bottleneck
        - Modern CPU speed ~GHz
    - Map worker reads input file on GFS(ideally, worker machine should have replica of the file) and generates output file in local machine
    - Reduce worker reads input file on nearby map worker machine.
- When tasks fail, simply re-run them.
    - Note that input and output are immutable
- M splits
    - input data are split into M slots
- R splits
    - intermediate key/value pairs are partitioned
        - hash(intermediate_key) % R
    - Each reduce worker processes data from a partition. Output is stored in one partition, too.

## MapReduce Lab ##

In this lab, tasks are not run on multiple machines but rather on multiple processes on a machine.
- Every input/output files can be found on the local disk. You don't have to worry about picking the right(nearby) worker machine for the tasks.

```golang
const MAP = 0
const REDUCE = 1
const EXIT = 2

const IDLE = 0
const IN_PROGRESS = 1
const COMPLETED = 2

type Task struct {
	TaskNumber int
	Files      []string
	JobType    int
	Status     int
	UnixTime   int64
	index      int // position at the priority queue
	Assigned   int
}
```
Tasks are made, changed, and assigned by the coordinator. It has fields:
- **Status** can be in three state: **IDLE**, **IN_PROGRESS**, **COMPLETED**. Tasks are moving between queue based on this field.
- **JobType**
    1. Can be either **MAP** or **REDUCE**. A worker either execute map function or reduce function according to JobType
    2. When no task is available, send **EXIT** task to close worker.
- **Files** give location of the files that  worker may have to work on.
- **UnixTime** reflects the time of last status change. It is updated to current when task changes it status. Coordinator health check the in-progress task by how much time elapses since **UnixTime**.
- **TaskNumber** is a unique label for the task. Worker notifies coordinator about completion with task number. Then, coordinator can mark the task with **COMPLETED** status and remove it from the queue.
- **Assigned**: worker process id that request for the task
    - When a worker is slow to reply and coordinator has already assigned the task to other worker, worker_id != Assigned. We ignore the reply.
    - How about extend expiry of in progress task and let task forget about this field?
        - We can't have meaningful length of expiry since it's hard to control network reliability, input size, machine resource.

```golang
type Coordinator struct {
	Tasks            []*Task
	IdleQ            SafeHeap
	ProcessPQ        SafeHeap
	NumOfReduceTasks int
	NumOfMapTasks    int
	Partitions       SafeMap
	Counter          SafeCounter
}
```
- **Tasks**
    1. lookup the task when worker reports back with a task number. e.g. Tasks[taskNumber]
- **IdleQ** and **ProcessPQ**(both are priority queues, )
    1. Store idle and in-progress task.
    2. A task is popped from **IdleQ** when a worker asks for it. Then, task is pushed to the **ProcessPQ**
    3. A task is removed from **ProcessPQ** and push to **IdleQ** when a in-progress task has been stale for too long
    4. A task is removed from **ProcessPQ** when a task is completed.
    5. Priority Queue should be thread safe.
        - Multiple workers can ask coordinator for the task concurrently through RPC
- **Partitions**
    1. Collects map output file location and its corresponding partitions when a map worker notify coordinator that task is completed.
    2. Make reduce tasks for each partition when all map tasks are completed
- **SafeCounter**
    1. Tracks number of tasks that have completed
    2. Close worker and coordinator when all tasks are finished.
    3. Make reduce tasks for each partition when all of map tasks are done.
    4. Thread safe

```golang
type SafeCounter struct {
	Count int
	Cond  *sync.Cond
}
```
- Count tracks number of completed tasks
- Cond is used to wake up sleeping threads when condition is met.
    1. Before map tasks are done, threads on coordinator are put into sleep when reduce worker ask for a task.
    2. When too many workers compete for map/reduce tasks, put remaining worker into sleep. Signal sleeping worker when any of in progress task fails or is slow to respond.


```golang
type SafeMap struct {
	partitions map[int][]string
	mu         sync.RWMutex
}
```
- **partitions** stores partition number along with files
- **mu** provides synchronization when multiple map workers finish tasks at the same time and ask coordinator to update **partitions** concurrently.


```golang
type SafeHeap struct {
	h  *PriorityQueue
	mu sync.RWMutex
}

type PriorityQueue []*Task
```
- **PriorityQueue** is a binary heap. It allows push and pop operation in $logN$ time.
- Choose priority queue over slice. High priority task can be pushed and popped in logN time. A few examples:
1. A task that failed should be assigned soon
2. Remove stale task
3. Remove completed task
- **SafeHeap** provides thread safe abstraction on top of PriorityQueue
    - **mu** is a read write lock that provides synchronization in concurrent push and pop, while allowing many reads to be efficient.

```golang
func Worker(mapf func(string, string) []KeyValue, reducef func(string, []string) string)
```
- Worker function takes map/reduce function as input
- Internally, it makes RPC call to the coordinator to ask for task. Notify coordinator about task completion when task is done. This process continues until coordinator responds with **EXIT** task

Write files to temporary folder before moving them to target folder.
- Machine can fail in the middle of program execution, and we might see partial state of the results. This will complicate our analysis for correctness.

To make debug easier, format your log message with the following
- coordinator process id
- worker process id
- task number
- job type
- time
- event that change status

then you can find the event trace of task 1:
```
grep task_number=1 log-file*.txt
```

A daemon check expiry of the task in progress priority queue. Move tasks from progress queue to idle queue if tasks don't change status for more than 10 seconds.

To validate our map/reduce implementation, write out a sequential map/reduce and compare it with distributed map/reduce. Be aware of duration and correctness.

Test scenario
1. worker crashes.
    - os.Exit(1)
2. worker is slow to respond.
    - sleep
3. worker process map/reduce tasks successfully.
- Note that we have one coordinator. We have to restart everything without replication when coordinator machine crashes. Thus, there is not point testing this scenario.
