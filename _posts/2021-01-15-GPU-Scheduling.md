---
title: GPU Scheduling for ML workload
author: hetong07
date: 2021-01-15 15:00:00 -0700
categories: [Blogging, Technical]
tags: [GPU, ML Sys, distributed system]
---
  

# GPU Scheduling for ML workload

  

GPU scheduling topic is very popular in both academia and industry. Previous focus is on distributed ML system, and key of those researches are the fact in ML training, the parameters do not require strong consistency. The solution is the Parameter server. Therefore, the key for scheduling multiple ML tasks within one cluster is also the ML properties.

## Workload properties

The ML workflow can be divided into two categories: 1) serving tasks and 2) training tasks. For serving tasks, there usually has a SLO, while for training, the bandwidth is more critical (batch processing), and can be terminated abruptly. Due to the different requirements, people usually dedicate cluster for training and serving, respectively.

## GPU properties

Compared to CPU, the memory size of GPU is very limited, tens of GB compared to hundreds of GB. This may incur problem of placing models in GPU: either the model cannot fit, or frequent memory exchange hurts the performance. Moreover, creating a GPU context is also time-consuming.


## [Pipeswitch](https://www.usenix.org/conference/osdi20/presentation/bai): scheduling in one machine 

It focuses on how to scheduling ML workload inside one machine.  As mentioned above, the memory limitation, the increasing model size and the strict SLO pose tough requirement on scheduling. The solution consists of 
1) Exploit the layer computation feature to divide module into different subgroup to create a computation pipeline to hide memory transfer;
2) Use double buffer (actually, triple buffers) to amortize the GPU initialization;
3) Customized memory management: 1) memory usage is invariant. 2) training tasks use memory in a FILO order (due to BP);
4) Pin GPU memory and allocate in customized manager;
5) Also store models in above manager to reduce module storage in memory.
6) Use CPU IPC to notify GPU that data is ready for next pipeline stage since GPU IPC is slow;
7) Reduce memory usage by separating model and its parameters.

My understanding of 6) is : the worker is a CPU side program which responsible for call GPU API to send data to GPU and start job, and the daemon is in charge of allocating memory. So to avoid using GPU IPC, the daemon only needs to notify worker instead of GPU.

Frankly speaking, the approach used in the paper is not novel, and the major drawback is very obvious: isolation. It also lack of considering for tasks that need multiple GPU. There would be low-utilization issue because the GPU is not spatially shared by different tasks.

## [Antman](https://www.usenix.org/conference/osdi20/presentation/xiao)

Different from Pipeswitch, Antman considering for scheduling GPU workload usually needs multiple GPUs.
The motivations are 

- Cluster utilization is low
- Jobs wait time is proportional to the number of GPUs it requires. 4 GPUs job may wait longer than 2 GPU jobs; simply reserve GPUs cannot solve this and it may further decrease utilization;
-  Memory demands varies during the lifetime of a task ( I think **Pipeswitch** would not agree :) ) and may cause interference for tasks co-located in the same GPU;

### memory management
The key property this paper leverage is the mini-batch: the memory utilization would drop significantly on the boundary of different mini-batches. At the starting point of each mini-batch, it adjust the GPU memory limit of each job based on its demands ( the number of tensors placed in CPU memory). As I can see, this approach could not totally solve the interference, and if one job continuously expands its memory usage, it would starve/hurts the co-located job. The authors seem to argue that mini-batch guarantees even if the starvation happens, it lasts very shortly due to the mini-batch properties.

### scheduling
To guard the SLO, the authors provides assign priority for different jobs and create a manager to scheduler to scheduling jobs. The scheduler also use the mini-batch info to predict which GPU is about to idle. To schedule a LE job it does:

- Allocate job in idle GPU, (optional) and guarantee GPUs are all "close";
- Reserve resource if No. of GPUs cannot be satisfied;

For BE job, what the scheduler does are:

- If the job has been detained for a long time, find a low utilized GPU (not idle) to allocate job
- Local coordinator limit BE job's memory and SM to reduce interference and gradually increase them it does not hurts the LE job's performance (mini-batch processing time)
- Also consider the interference severity between different jobs.
- If possible, promote BE to LE.

### Some thoughts
- I guess the reason for BE job upgrade is the possibility of starvation of BE job if LE requests are too many; 
- I am curious that if it is possible to co-locate multiple (>=2) LE jobs in one GPU because LE job may have more predicable memory usage; 
- It seems that all the control signals are gathered from application (framework) level, it may not be a problem since mini-batch processing time is usually hundreds of milliseconds;
- Mini-batch and the co-location properties are manually tuned, so if there is an new model emerges, this method may not idle in a short time;
- Still cannot solve the resource reservation caused low utilization issue. Maybe we can schedule some BE job to the reserved resources?
- More system level resource limitation is needed in GPU world, e.g. cgroup?
- Could we utilize the properties of training job (BW critical, more memory, stop abruptly) and serving job (latency critical, less memory, long running)?


## [Heterogeneous scheduling](https://www.usenix.org/conference/osdi20/presentation/narayanan-deepak)
This paper's novelty is that, as far as I know, it is the first paper to address the interconnection issue (Network, PCIe, QPI) as a whole for ML workload. It also proposes the advantage of CPU in ML workload.

### Background
There are two types of distributed ML systems:

1. All-reduce 
> In a task that uses all-reduce, only GPU machines are involved. In an iteration, GPUs compute the gradients of the model parameters independently, and then aggregate gradients using the all-reduce primitive.
2. Parameter Server.
> In PS tasks, both GPU machines and CPU machines can be used. Different from all-reduce, the gradients are sent to PS, which typically runs on CPU machines and aggregates the received gradients. PS then runs certain DNN training optimizer, e.g., SGD or Adam and sends back the updated model

The core difference is All-reduce is a P2P structure (more specific, **ring** structure), while the PS is a master-slave architecture.

The authors argue that the CPU usage is relatively low in GPU clusters.
### Solution
The authors prove the theoretical optimal configuration under different situations, such as, with and without extra CPU available (omit here), and create a scheduler based on the analysis. 
More interesting part in this paper is to consider the intra-machine communication part. It take the PCI, NVlink etc. into account. The basic idea here is to constrain the data exchange within the **bottleneck region** to mitigate the bandwidth contention, especially:

- PCIe only
The bottleneck here is the communication across the PCIe switch. So the remedy it. the authors let the GPU within the same PCIe switch to do reduce first, and then send data to CPU, and in turn, to NIC. 

- NVlink
The bottleneck is the NIC to CPU. So it use the NVLink to exchange data, and let the final aggregator to send data to NIC to reduce contention.

Some other analysis in this paper involves 1) why the divide the SS and CS and how to support async in this dividend; 2) Use pipeline to overlap processing time; 3) Amortize  RDMA WRITE round trip; 4) RDMA related issues (**interesting, you would not know in other places!!!**

### Some thoughts
I think the novelty in this paper is that it provides the theoretical analysis of the communication pattern in GPU scheduling, and it is the first, as far as I know, to consider the network and connection as whole. It also provides some very interesting knowledge related to RDMA. Such knowledge can be found in a NSDI paper from Microsoft two years ago.Beyond that, it also contributes to the PS architecture by splitting the SS and CS and place them on CPU and GPU, respectively. I am glad to learn so much internal knowledge of ML algorithms.

## [HiveD: GPU affinity scheduling](https://www.usenix.org/conference/osdi20/presentation/zhao-hanyu)

### Background
The authors found that current scheduling design only consider the number of GPU required by a job, however, due to many reasons, mostly the network and interconnection I think, if the GPU affinity is not considered, the performance will degrade. This observation is a little bit similar to the Bytedance paper (in terms of interconnection), but the authors focus on another aspect.

In current GPU cluster, users usually have the ability toe assign the affinity to a job, but due to the fragment, the affinity may not be fulfill, so a job can be starved. The authors further argue that reducing the fragment through scheduler is not easy to achieve since existing scheduler is already complicated. So they build a separate scheduler and use multi-level scheduling to solve this problem.

### Solution
> HiveD proposes to guarantee sharing safety (i.e., eliminating sharing anomaly as described in §2) as a prerequisite of sharing a GPU cluster. Specifically, if a sequence of GPU requests with affinity requirements can be satisfied in a private cluster, it should be satisfied in the corresponding virtual private cluster and the shared physical cluster.

**The goal of this paper is to guarantee that job runs on cluster will not perform worse (in terms of queuing delay) compared to on the private cluster.** The solution is very simple: it divides the GPU into 4 different layers: GPU (level-1), PCIe switch (level-2), CPU socket (level-3), and node levels (level-4), and exposes the structure of the GPU cluster to the schedulers. The scheduler is then only need to consult the structure assigned to users and try to allocate GPUs on different layers. Spiting and merging GPUs at different layer is exploit to reduce resource fragment.

It also has a preempt based priority support to improve GPU utilization, but to reduce the preempt chance, it prefers to allocate low priority job to cluster and is farthest away high priority jobs and place high priority jobs with less low priority jobs as possible. 

For failure recover, it employed the decentralized way to store the status and reconstruct the status during recover ( related to k8s knowledge).


### Some thoughts
- Exposing structure of GPU to upper layer is a usually a good way to solve complicated problem, but it requires the users or administrators to manually specify the GPU structure, which I think, would pose burden to them;
- Exposing the structure fails to consider the interconnection interference within the same machine (as discussed in the *Heterogeneous scheduling*);
- The allocation strategy (place low and high jobs as separately as possible) seems to cause low utilization, so multi-tenet on one GPU may be needed;
- The benefit of this paper is that it reduce/restrict the GPU fragment, no more;
- The performance of this solution highly depends on the jobs and the VC assignment.


## [Heterogeneity-Aware Scheduling](https://www.usenix.org/conference/osdi20/presentation/narayanan-deepak)

### Problems
This paper identifies three problems in state-of-the-art GPU scheduling framework:

- **Performance Heterogeneity** There are various type of accelerators, even GPU has different generations;
- **Generality across Policies** Different users has different optimization goal, but existing framework embedded with specific policy, making them hard to extend;
- **Co-location and Placement Optimizations** Co-location can improve utilization, but interference should be considered.

### Solution
This paper argues that we should explicitly schedule the tasks on various resources, e.g. GPU, TPU, FPGA. It needs to model the job allocation on different accelerators as an allocation matrix, X (the variant to solve), and the performance (in term of throughput) on different accelerators as T. It is noted that, T also contains the throughput if different jobs are spatial sharing an accelerator. The throughput matrix can be get from a pre-profiled database or profiled on the fly, and it also proposes a fingerprint profile matching solution using matrix completion.

After solving the optimization problem in above model, the scheduler is in charge to move the allocation as close to the optimal solution as possible. This is achieved through a priority score matrix computed by the time spent on specific job on specific machine (see &5).

The scheduling happens in stages (rounds), where the stages are divided by mini-batch (similar to that of Gandiva). to remedy the starvation problem. Considering scheduling with space sharing (SS) enabled is a NP-hard problem, the authors develop a greedy solution: 1) find the highest priority job and check whether its requirement can be fulfilled. if no, find the the next priority job, else 2) schedule this job and remove all jobs assignment contains this job to ensure the same job will not be scheduled more than once in one round.

### Some thoughts
- Section is of high value, need read more carefully;
- What is the matrix completion discussed in section 6?
- In experiments, it seems this solution is slightly better than Gandiva, I think it is reasonable, since its solution is similar to Gandiva, but the allocation is proved to be better than that of Gandiva;
- Due to the current ML workload properties, this paper only consider space sharing of two jobs. This is OK now, but may be in the future it would encounter a scalability problem if multiple sharing is permitted, and as far as I know, solving optimal problem is time-consuming;
- Compared to the **HiveD**, it does not pose burden to cluster assignment, so it would be a better solution;
- Need revisit this paper in the future.

## [Gandiva](https://www.usenix.org/conference/osdi18/presentation/xiao)

### Background
Gandiva is one the early solution to solve the ML training workload scheduling problem. And it also identifies some ML training workload properties as

- Training workload often needs early feedback;
- GPU affinity is important;
- Memory usage reduced greatly on the boundary of two mini-batches;

### Solution
This paper uses multiple to solve this problem:
- Time-slicing: multiple job that fit in one node shares the GPU resources through suspend-resuming techniques;
- Packing: Spatial sharing GPU resources; To remedy interference problem, it measures the mini-batch processing time and memory usage to know whether two jobs can be co-located, i.e., if after scheduling, the processing increase, then this assignment should be revoked.
- Grow-Shrink: Speculatively execute jobs on low utilization machines.
- Migration: migration to remedy the fragment issue. migration can be done in < 4 seconds.

When assign a job, the scheduler first tries to schedule it on minimal load nodes (prefer nodes with no load GPUs). If it cannot fulfill, the algorithm then gradually relax the affinity requirement. If there is no GPUs available, it migrates existing jobs to create usable resources. So migration is the least thing to do in scheduling. This approach make the job to run as fast as possible to fulfill the feature of training jobs.

### Some thoughts
- The reason in section 6.4, the Ganvida can outperforms the YARN is due to the fact the the GPU requirement cannot be fulfill. It seems the advance of Ganvida cannot be totally attributed to migration.
- Migration even on the basis of mini-batch is still time consuming, and considering the priority issue, it may causes it infeasible to fulfill next high priority jobs. Moreover, frequently migration may cause oscillation, which may hurts the performance even further.

## Others
- I remember that Prof. Arvind team has a [paper](https://homes.cs.washington.edu/~arvind/papers/nexus.pdf) talks about providing SLO for serving ML workload. It is also interesting since it points out that bin bucket is not sufficient for serving ML workflow with SLO on GPU cluster. I have forgotten its main idea, but I will read it again and add my understanding here later.
- I haven't read [Tiresias](https://www.usenix.org/conference/nsdi19/presentation/gu).