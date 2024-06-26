### [Exploiting Cloud Object Storage for High-Performance Analytics (2023)](https://dl.acm.org/doi/abs/10.14778/3611479.3611486)

The paper presents the AnyBlob; A cloud based object store download manager based on io-uring for query engines that
optimizes throughput while minimizing CPU usage.

Due the recently improvements on network bandwidth on cloud providers it is more viable to use remote storages for
high-performance query analytics. In 2018 AWS introduce instances with 100 Gbit/s networking, which improves severely
the latency and close the gap between local file systems and remote storages.

Previously researches focus mostly on OLTP databases and using cache to avoid fetch data from remote storage, Anyblob on
the other hand demonstrate that even without caching it achieves similar performance to state-of-the-art cloud data
warehouses that cache data on local SSDs while improving resource elasticity.

Anyblob use io_uring to manage multiple object store downloads per thread asynchronously. With this model, the system
does not have to spawn too many threads to download these objects in parallel which would increase the CPU usage due to
kernel thread scheduling.

### io_uring
io_uring is a Linux specific asynchronous I/O API that allows the user to submit one or more I/O requests, which are
processed asynchronously without blocking the calling process. 

io_uring use a pair of lock-free ring buffers in shared memory that are used as queues between user and kernel space.
- Submission queue entry (SQE): The user process use the submission queue to send asynchronous I/O requests to the kernel.
- Completion queue entry (CQE): Once the I/O request is finished the Kernel use the completion queue to send asynchronous
      I/O results. 

Using io_uring the query engine need to spawn fewer threads to download objects from the remote storage in parallel,
which can dramatically reduce the CPU usage.

The basic flow is, the user inserts new SQE's, such as `recv` and `send` operations into the submission queue and then
to check if a request was successful, the user periodically peeks for available CQE's.  Inserting into the queue does
not require any system call, so the user can batch multiple operations into the queue and then execute all of them in a
single system call asynchronously.


### Object storage cost
The paper also discuss about object storage costs. A interesting detail is about the optional request size. Download
requests can either be full objects or byte ranges within objects. Cloud providers usually charge by the number of
requests, large requests result in lower cost for the same overall data size, on the other hand, the size should be as
small as possible so that small tables can benefit from parallel downloads. The experiments shows that request sizes of
8 - 16 MiB are cost-throughput optional for OLAP.

It's important to keep in mind that all experiments developed on this paper was using unsecured connection to S3 using
HTTP. It's because HTTPS requires more than 2X CPU resources of HTTP, and at AWS, all traffic be- tween regions and even
all traffic between AZs is automatically encrypted by the network infrastructure.  Within a location, no other user is
able to intercept traffic between an EC2 instance and the S3 gateway due to the isolation of VPCs, making HTTPS
superfluous. However, encryption-at-rest is required to ensure full data encryption outside the instance (e.g., at S3).


### Anyblob Design
Anyblob use a state machine for each request when downloading an object from a remote storage. An object download
request is represented by a `Message Task` which is associated with the state machine. The message task is divided into
multiple phases. On successfully completing a phase the state machine go to the next phase until the object is fully
download from the remote storage. The state machine enables asynchronous and multiplexed messages with a single thread.
Several `send` and `recv` system calls are required during transfer until the object is downloaded. After each system
call, Anyblob suspend the execution of the message until it is notified about the successful syscall. Each SQE has a
user-defined member that allows passing information to later identify its origin Message Task. The uring is periodically
checked for available completion queue entries (CQE). When a CQE is available, a system call has been processed. With
the retrieved information, Anyblob can evaluate the next Message Task step

### Database Engine Design
Anyblob was integrated with the DBMS [Umbra](https://umbra-db.com/). Like most database systems, Umbra uses a pool of
worker threads to process queries in parallel, the difference here is that Umbra use these worker threads to process
queries and also to download objects from remote storages.To overcome issues with long-running queries that block
resources, many database systems use tasks to process queries. These tasks can either be suspended or run only for a
small amount of time. Umbra tasks consist of morsels which describe a small chuck of data of the task. Worker threads
are assigned to tasks and process morsels until the task is finished. The main difference of this approach is that
workers do not only need to process data but also retrieve new blocks from the remote storage to processing. After
receiving a morsel for processing, the thread scans and filters the data according to the semantics of the table scan.
When all morsels of an active block are taken, the thread picks the morsel from a new, already retrieved block.

Umbra use a scheduler to download objects from the remote storage to process the data. The main goal of the object
scheduler is to strike a balance between pro- cessing and retrieval performance. It assigns different jobs to the
available worker threads to achieve this balance. If the retrieval performance is lower than the scan performance, it
increases the amount of retrieval and preparation threads. On the other hand, re- ducing the number of retrieval threads
results in higher processing throughput. 


### Performance
By simply swapping the retrieval library from the asynchronous AWS SDK to AnyBlob, Umbra achieves up to a factor of 1.2×
better performance and an improvement of up to 40% on computationally expensive queries. Additionally, AnyBlob reduces
the mean CPU usage by up to 25%. Umbra also achieves an average CPU utilization of ∼75% with asynchronous networking.
Networking uses a large share of CPU time that accounts for up to 25% of the total utilization, significantly reduced by
AnyBlob.
