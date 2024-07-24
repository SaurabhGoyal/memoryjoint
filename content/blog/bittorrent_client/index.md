+++
title = "Writing a Bittorrent Client"
date = 2024-07-05
description = "Writing a Bittorrent client."

[taxonomies]
tags = ["torrent", "networking", "io"]

[extra]
toc = true
quick_navigation_buttons = true
+++

# Bittorrent
## What is it?
- It is a protocol to exchange data chunks with other people on a network, generally internet.

##How is it different from simple HTTP?
- HTTP requires client server model where you (client) download data from a data owner (server) that you know about.
- Bittorrent enables a lot of people in the internet to collaboratively make the file available to a lot of other people without one server being a bottleneck or owner. Essentially one file can have many servers instead of one.
- This results in higher availability and download speed of the file.

##How does it work?
- This is the best part - this is one of the simplest protocols I have ever read. This is essentially a simple logical solution to how to achieve multiple people in a network requesting and serving a file.
- The whole specification is listed [here officially](http://bittorrent.org/beps/bep_0003.html) and [here unofficially (but more verbose)](https://wiki.theory.org/BitTorrentSpecification) but I would give a high level view here -
    - You are a peer, looking for a file.
    - You need a unique-id for the file and someone with whom you can ask about servers of that file.
    - These two pieces of information is available in Torrent file / magnet link. You parse that file / link to fetch above two, i.e. unique-id (info-hash) and someone to enquire sources (tracker).
    - You ask tracker to provide a list of sources (IP addresses of other people in the network) who can give file.
    - You ping these peers and say that you need data for this file, if peer has it, they ping bank, i.e. handshake completed.
    - You ask for file data in smaller chunks from peers.
    - Peers send thbe requested smaller chunks if they have.
    - Once all chunks have been downloaded, you combine them into the file. !!! You have the file now. !!!
    - All this meanwhile, peer keeps telling you, which specific chunks it has and whether it can currently serve you those or not.
    - And the most important part, YOU ARE ALSO A PEER, serving file chunks that you have to other people requesting them.

## Why build a client?
Building a robust client requires dealing with following concepts in depth -
- **Reading protocol specifications and following them -** As insignificant as it sounds, most of my wasted time during implementation was result of me not reading the specification thoroughly enough and missing understanding the right sequence of messages to be sent to peer and having wrong expectations from the peer without a clear way to debug.
- **TCP/UDP and sockets -** It requires a lot of data transfer and packet exchanges which need to be raw TCP/UDP messages. For them to be correctly perceived by peer, you need to form them correctly. While you may be interacting with the networking APIs of the kernel via the networking APIs provided by your programming language's libraries, you would have to understand how sockets work, how they are made into blocking vs non-blocking to be able to correctly send and receive data at the same time from the peer.
- **IO -** You would have to write a lot of data into disk, it will be made available in chunks and then verified for integrity and merged into the output file. This can happen for multiple files concurrently. This would also require you to be able to continue from where you left in case of failures or client closures. This requires understanding IO APIs and FS APIs in detail.
- **Concurrency -** The application is going to be IO heavy since most of time would be spent in either downloading data from network (peers) or writing data to disk. This makes it a good and almost indespensable case for adding concurrency to application to get any practical performance. There would be atleast two aspects that need to be concurrent - 1) Requesting and downloading chunks from peers 2) Managing multiple peers at the same time.

# Design
## Entities
Let's first understand what entities we are dealing with. By entities I mean objects that are alive, i.e. require a lifecycle / state management.

### Block
This is the smallest chunk of data that you request from peers and combine them into the files one needs. A block of data may go through following two sets of lifecycle stages at any point of time -
- Data -
    - Pending for download
    - Requested from peer (enqueued)
    - Persisted temporarily (in a dedicated file of its own, pending merge)
    - Persisted in file (in the desired output file)
- Integrity check -
    - Not verified
    - Verified

![State diagram](https://www.plantuml.com/plantuml/svg/PP4nQ_Gm38Pt_mgHwViA7Rg68nmTEdLewT6bT71b-I8kSPnO2To_hnmdb5u7Wz3p1BqlEIQnaynzPuIb8wWUkm4lyCoUy8eTLSQe8GfUA3WEvmfiWXZsxUjCCxbrEVwOK-8avE14VImV9FbBdpZORiD-n-yqiMUqmaC0R0algx7W9Y0S3jWEZDGqnfYFkq-uRtAW6F8mch664_UaTX_Xtvpqa1ycSFGrp06rmNypghb6qgSsgUXPoqShRTuLx8s-kgIyvIeicFa-BhXEUQZXhRKF9NivRSK21w7pn78rnOoXXbzTlTKJRgDQwmP7cvBY8mMdV-iR)

### Peer
This is the TCP/UDP connection you maintain with a potential peer for fetching the blocks from. Other than the connection, interest status, choking status and a map of which blocks peer has is maintained in the state as well. A peer connection goes through following stages -
- Initiate a connection
- Complete handshake for a specific torrent.
- Express interest
- Update their block availability
- Request piece
- Save piece
- Connection ended

While a connection is not torrent specific, handshake between peers is torrent specific and creates a virtual session where every subsequent message would belong for the torrent on which peers handshook on and the messages themselves won't have any information of the torrent. This effectively means one connection per torrent per peer.

![State diagram](https://www.plantuml.com/plantuml/svg/NL2nJWCn3Dtp5LOdG5Ji7L2bBbHYgA1Cb26Nk2IwEmV5ZghxUvnSZqei5_lPxztpsxBOB6KSZ4GP45O7n0olyOnkSWEkZD45KVnk__C8XvJbVWMM8IxuSNU0NI929p5HcxbbzcB9Sx0zDZZmWh-7T84z2MPacUL8bk47kP1wzD1DKCsqUVdJlFqBZfZ7I8hwjYFQ6cC-7xvWlNvMXn7qSSPje9faoMX7AApIvvIXIn908G_g4Yuv2XhNc4t8LR9Q3bmBzQVL1jvqVm99e6TbX16PxJVoYQgY1DHHpjYMOL5IRgsBgzMctGI4wBdgzAHHIB029vfIWQeQhycWNxXDfxZ_v3971KfKUeTgOcwS9R3Sjpkz5QlSssNrev744LCHUDAR6Ekx6nBFiRiX8abRl4QvHV9b77u1)

### Torrent
This is the overall envelope for the torrent. In terms of work, it should provide -
- Scheduling of blocks
- Handling verification and persistence of blocks
- Merging blocks into the file output

It can also provide features such as state persistence across application launches as to continue the download from where it was left.
 
### Client
This is just an interface to add, start, delete and continue torrents using the underlying torrent API. It can also provides a monitoring interface such as a rendered view for all the updates happening under the torrents.  

## Actors
We need following components to perform the end to end download operation -
- Peer Controller - For given peer ip, port and torrent, this will perform following actions -
    - Initiate and maintain connection with peer.
    - Maintain and update state of interest, choking and piece-availability-map (bitfield).
    - Receive commands from other controllers to request piece from peer.
    - Receive and download piece from peer and send the piece data to other controllers for persistence and verification. 
- Torrent Controller - For given torrent metainfo source (torrent file or magnet url), this will perform following actions - 
    - Parse metainfo from the source using bencode parser.
    - Initiate and sync torrent state with the data that is already present on the disk.
    - Fetch updated peer list from tracker.
    - Schedule pending blocks for download by sending commands to peer controller.
    - Initiate and maintain peer controllers as part of scheduling (this, instead of peer connection pooling, is done as to make a peer connection only if needed for a block, saves a lot of socket count).
    - Persist downloaded blocks when recieved from peer controllers.
    - Verify persisted blocks using Sha1. 
    - Write files as and when all blocks for the file have been persisted.
- Client Controller - This will perform following actions - 
    - Initialise and sync client state with the data on disk for info of already added torrents and their status.
    - Provide APIs to add, pause, resume and remove torrents.
    - Render the state into UI.

## Key aspects
### Concurrency
- First read [Concurrency Primer](#concurrency-primer). 
- Let's now understand all our IO bound tasks and what we know about their timing -
    - Terminal -
        - Reading data from peer's connection - we don't know when peer may write to us
        - Writing data to peer's connection - we don't know when we may have to write to peer to request a block since that directly depends on block's scheduling and availability from all peers
    - Dependent -
        - Scheduling blocks - dependent on writing to peer
        - Writing block on disk - dependent on data from peer
        - Reading block on disk - dependent on data from peer - we have to read data to perform Sha1 verification
- For brevity, here are the main CPU bound tasks -
    - State management and decision making based on status
    - Sha1 verification of available blocks
- If we check IO bound tasks for a sample state of 5 active torrents with 5 peer connection each, we would have 25 peer connection at any given time and 5 torrent controllers to manage each torrent's state. This results in conecurreny factor of 55 (2 per peer for reader and writer + 1 per torrent controller = 2 * 25 + 5). Practically a torrent may make connections with ~10 peers and a client should be able to support ~10 torrents. That would make the concurrency factor to 210. We can clearly see that a thread based concurrency model would not be scalable for us. Because of clear unscalability of thread model and the IO-heavy nature of the application, async model is the right way to go.
- We will see later if we need to customise the thread pool to dedicatedly handle the CPU bound tasks for better performance.

### Data Sharing
Sharing data among concurrent tasks is tricky. I tried following three options -
- **Shared memory with exclusive locks (Mutex) -** I shared the torrent state among all peers and the torrent controller. It meant that anytime there was an update for torrent state (ex.- a block has been received from peer), the whole torrent state was locked and thus all other peers as well as torrent controller was blocked on it and waiting and could not perform any operation. While it worked with small number of peers and a waiting time of seconds among execution of each task, this eventually started resulting in too much waiting on the locks and bad performance when I increased number of peers and made refresh rate faster.
- **Shared memory with read write (RWLock) locks on individual blocks -** I then tried making locks granular by having lock per block instead of on whole torrent. While theoretically it should reduce blocked time on locks, it actually reduced the overall performance by a significant margin. The reason was that the torrent controller still needed to read lots of blocks to decide which ones to schedule (requires reading all blocks), which ones to write (requires reading all blocks) and which ones to verify when a block is received (requires reading all sibling blocks of a parent concept called piece). This meant that even now the locks were being held and waited upon for almost all of blocks except with the overhead of having a lock for each individual block which can be huge in number.
- **Message passing (using channels) -** I checked possible optimisations and realised two things - 1) the cost of updating the data that every task needs is quite low, it's in memory and finite (Do note that blocks actual data is always written to disk asap and not retained in memory for long) 2) the data consistency among tasks can be eventual and it would not cause a huge performance issue, i.e. even if a peer has old information about a block and downloads it again, it does not impact the system as the torrent controller would discard the data when sent by peer controller. So I switched to **message-passing model (just a way of sharing a copy of the data instead of the shared memory reference)** where every task maintains its own state and on update sends a message to other layers via a concept called **channels**. This makes the system completely lock free and enhances the performance by a lot, essentially no task is blocked on any other task for performing its functions.
    - Do note that channels themselves are built using a shared-memory concept but they still provide a better performance because they follow a relaxed consistency model by implementing it as a shared-queue with no guarantee on delivery timing and ordering of messages thus giving much better performance.
    - Another reason for performance is that generally a task's execution's involves reading data, doing some calculation and then updating the data with these 3 steps making the biggest part of task's time. In shared memory model, the data needs to be locked in all these three steps making the whole task a bottleneck compared to just the message passing instruction being locked.

## Implementation


<!-- Managing the life of things such as connection with peers, block-scheduler and torrent-state handler. Specifically below given parts -
    - Cardinality - Which entity are they mapped to, ex.- client / torrent / peer / block. Ex.- One peer controller per torrent<->peer combination. 
    - State management - How they would hold the state they are working on and how they would sync with state of other components. Ex.- Shared state vs message passing.
    - Shutdown - Finding the right set of conditions to shutdown a component gracefully. Ex.- Closing the torrent controller if all files have been downloaded, closing a peer controller if TCP timed out.   -->

# Next Steps
I am still working to add followings -
- DHT protocol support
- Bad peer blocking
- Performance measurement
- Performance improvement - memory footprint
- UDP protocol support
- Serving data

# Appendix
## Concurrency Primer
Ideally, we want everything to run all the time parallelly. We can just run every task some amount, switch to another and keep going like this untill all tasks are complete. However, a blanket strategy of making every task concurrent is not efficient because of two factors -
1. actual execution resource - This is always the processor, you are always bounded by the number of processors / cores you have in the system which is always limited and constant compared to the number of tasks you would want to parallelise. 
2. dependency of each task - we need to see where we can save time in execution and that is where we need to check those dependencies of the task which are outside the processors' capability such as time, devide drivers, network etc.  Simple examples
    - if two tasks say wait 10 second, the processor doesn't have to wait for 10 + 10 seconds, processor can trigger the tasks and parallelise the waiting part because the waiting (time increasing) is not happending inside the processor, the phenomenon lives outside the processor.
    - if two tasks say to write a file each to a disk, the processor doesn't have to write them one by one, because that ,means processor has given data for first file to the disk driver and then waiting for the driver to finish. Processor can trigger the tasks, send the writing request to disk and then parallelise the waiting part because the waiting (for disk driver to complete writing) is not happending inside the processor, the phenomenon lives outside the processor.

This means that if we have tasks which actually use the **processor's internal capability**, i.e. executing instructions, then we are limited to the concurrency we can support, or more accurately, we are limited to the benefit we can get from concurrency because end time taken by all tasks would be same as running them one by one. These are called **CPU bound** tasks and the processing is called **blocking request handling** because processor is blocked on performing the task. On the other hand, for tasks which don't need processor's internal capability and use **phenomenon that lives outside processor** (such as time and IO) would benefit a lot by concurrency if processor and the interface it is talking to provides an async model, i.e. the interface can take the request from processor and detach the proecssor from it and allow it to check the results at a later stage. Such tasks are called **IO bound** tasks and such processing is called **non-blocking request handling**.

With that clatrity, we can discuss the two concurrency models - multi-threading and async. **Multi-threading** is built on threads API provided by OS and is geared for CPU bound tasks. Since there can be limited concurrency among such tasks, the thread is a heavy resource and the scheduler is geared to handle small number of them. **Async** on the other hand, is geared towards large number of IO bound tasks and can have (specifically in Rust) any kind of third party scheduler implementation. Interesting point is that the tasks themselves can be conurrently handled within a single thread as well as can be distributed among a finite number of threads.
