---
title: "Raft 101"
date: 2023-11-18
tags: ['raft', 'distributed systems']
draft: false
summary: "Explore how Raft powers reliable distributed systems through efficient leader election and log replication, making consensus simpler, stronger, and easier to reason about"
---

## Intro


Raft is a **consensus** **algorithm** for managing a **replicated log** (we will explain what this mean later on) and it is widely used in building Distributed Systems. The core of distributed systems is just a set of operating computers that are communicating with each other through in order to do some coherent tasks.


In this article we are going to go through two of the core mechanism for Raft which are Leadership election and Log replication, this is going to be a long one so grab your favourite drink and enjoy the read.


## Motives


Before we talk about the motives I want to emphasize that if you're designing a system/solving a problem you should always opt for a **monolithic** design **if** you can, as they are far more simpler, easier to build and maintain.


Sometimes the problem you want to solve might need something from the following


* High Performance (Parallelism)


 You want lots of CPU, lots of memory, etc. A good example would be **Map-Reduce** model, which is used for processing and generating large data sets. It's one of the simplest distributed systems examples, you can read more about it from the official paper [here](https://research.google.com/archive/mapreduce-osdi04.pdf) and you can find a simple example implementation for it on my GitHub [repository](https://github.com/abdelrahmanIEl-Batal/mit-distributed-systems/tree/main/src/mr).


* Fault Tolerance


 Striving for a resilient system that can withstand software, hardware, or natural failures? By avoiding a single point of failure, you fortify your system against unexpected disruptions.


* Physical Reasons


 Some systems are inherently distributed due to physical constraints. Whether dealing with large data that defies storage in a single location or managing communication between geographically separated entities, such as bank branches in different locations, the need for distributed systems becomes evident.




These motives represent just a fraction of the considerations in system design. While there are additional factors at play, these are the most common ones.


## Raft


We introduced Raft earlier as an algorithm that manages **log replication** but what does this actually mean? This requires us to have a closer look at what **consensus** **algorithms** are.


### Consensus Mechanism


Consensus algorithms involves **multiple** machines/servers **agreeing** on some values **(state)** and once they reach a decision on some value, that decision is final.


Imagine having couple of servers running, a user sends a login request, the request get served by **one** of the servers and it generates some kind of auth token, stores it in the database and returns it to the user to be used in future requests, something unfortunate happens and our server (the one who served the login request) **fails** **before replicating** the auth token **(state)** to other servers, now whenever the user sends any request, the server would reply with some kind of "you need to login" message even though he has logged in before.


Consensus algorithms ensures that a system will continue working when the majority of machines/servers are **available** and agree on some **state**.


For example a majority for a system which has 5 servers would be able to make progress when **at most 2 servers fail.**


Consensus algorithms typically have the following properties


1. They must produce correct results under all non-[Byzantine](https://en.wikipedia.org/wiki/Byzantine_fault) conditions, including network latency, network failures (packet loss, packet duplication, etc.)


2. They must be time independent to ensure log consistency.


3. They must be fully functional, all servers can communicate with clients and each other, whenever a server fails, it can recover and has its state consistent with the rest of the cluster.


4. Usually the reply is sent back to a client, when majority of servers have applied the **request/command** to their **state,** this ensures that not only other servers has a **replica** of the requests/commands but that they also applied them to their state.




now we know what log replication means, it's just replicating whatever command a client sends amongst servers and applying it to the server **state**.


### Raft features


1. Strong Leader


 One of Raft's rules is that there has to be only **one** **Leader** at any given time, and this leader is responsible for replicating logs among servers and ensure that they have the latest log entries.


2. Leadership Election


 Leadership elections happens whenever **non-Leader** servers (we will call them **followers** from now on) don't hear from the **Leader** for a certain amount of time, they assume the leader has failed and start an election to elect a new Leader.


3. Membership change


 This refers to the scenario where you want to add or remove servers (nodes) in the Raft cluster. This might be necessary for various reasons, such as scaling the cluster up or down, replacing failed nodes, or reconfiguring the system.




### Raft Server


Each server represent a replicated state machine, in our Raft context the state machine refers to the data or the state that needs to be consistent across all servers in the cluster.


Each server stores a log containing a series of commands, which its state machine executes in order. Each log contains the same commands in the same order, so each state machine processes the same sequence of commands. Since the state machines are deterministic (hopefully), each computes the same state and the same sequence of outputs.


**Notice** that not all commands are **deterministic**, an example for **non** **deterministic** command is generating a random number, in this case the Leader can't replicate the command itself, as every other server is probably going to end up with a different result, a solution can be that the Leader execute this command and then try to replicate the result to other servers.


![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700084765235/eaa91a26-1667-45b1-8991-a0009769fc16.png)


The **consensus module** on the server receives requests from client, add it to its log and communicate with other modules on other servers ensuring that their logs contain the same requests in the same order and only once the log is consistent each **server's state machine** apply these commands in their order.


## Leadership Election


Servers inside a cluster can be elected to being a Leader through a certain process, but first let's explain the different **states** for the servers at any given **time**, and explain what we mean by **time** in the context of Raft


### Server States


1. **Follower**


 Followers are passive, they **don't issue** any requests on their own, they are only responding to requests from a **Leader** or a **Candidates.**


2. **Candidate**


 When a **Follower** doesn't receive any communication (either a **heartbeat** or request) from a Leader within a time period called **election timeout,** it **converts itself to a Candidate and starts and election,** during the election it request votes from other servers, and if a Candidate receive majority of votes it becomes the leader for that time.


3. **Leader**


 The **Leader** is the active server responsible for handling client calls (if a client reaches out to a server which happens to be a follower it redirects the request to the leader) and **managing log replication**.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700168463594/ad0ef035-b1da-427a-8929-f74bb7648fcd.jpeg)


### Time in Raft ?


Raft divides time into something called ***terms*** of arbitrary length, terms are numbered using **consecutive** **integers**, the following diagram illustrates this


![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700170721790/1d0c1ad6-0b12-4f98-88c1-3ee7316d92ff.jpeg)


Each term starts with an election, after an election only **ONE** leader manages the cluster until the end of the term. Sometimes a term can end with no leader being elected like **Term 3** in the figure.


Terms act as a **logical clock** for the servers, they allow server to detect obsolete information like **stale Leaders or stale requests.**


Imagine we have a cluster of 5 servers, and **server[1]** is the leader for **Term [1],** due to some network issues or message delays, other servers elected another leader at **Term [2], server [1]** continues to act as a **Leader** until it receives a request from the new Leader at **term [2],** it finds that its term is outdated and and reverts to being a **follower.**


The same happens to detect **stale requests,** whenever a server receives a request which has a **term less than the current term for the receiving server**, it ignores this request and replies back with it's term for the **calling server to update its state**.


Rule of thumb, whenever a server detected that it's term is outdated, it updates it's term (in case that server is a Candidate/Leader it converts to being a follower)


### Elections


Raft uses mechanism called **heartbeat** to trigger elections, a leader periodically sends **heartbeats** to the other servers to notify them that it is alive, these are called **AppendEntries RPC** calls, this RPC is not specific to sending heartbeats only, in fact whenever a leader receives a request/command and want to **replicate** it to other servers it sends them an **AppendEntries RPC** call which contains the **logs** to be replicated, in case leader hasn't received any new entry, then the **AppendEntries RPC** acts as a heartbeat with an **empty log.**


If a follower receives no AppendEntries RPC within some period of time called **election timeout** (these are random timeout specific for each server and we will know why in a second) it assumes the **Leader** **died** and starts a new election going through the following steps


1. Converting from a Follower state to Candidate


	* The server increments its **current term** (there can't be two leaders on the same term)
	
	
	* The server votes for itself
2. Sending **RequestVote RPCs** to other servers request their vote to become a leader.


3. Candidate continues in this process until one of the following happens


	* It wins the election by getting majority of votes (votesGranted > n / 2, where n = no. servers in the cluster) from other servers having the same **term** number.
	
	
	* Another server establishes itself as a Leader, this could happen if a Candidate receives an AppendEntries RPC from another server claiming to be the leader during an election and the **AppendEntries RPC has a term >= the Candidate's term**, then the Candidate recognizes that their is a leader and converts to a **Follower** **state and update its term to the Leader's**.
	
	
	* No one wins an election and the term ends with no **leader**
	
	
	 This can happen if multiple **Followers** become a **Candidate** at the same time and a **split vote** happens (no one gets majority of votes but instead the votes are split).
	
	
	 Raft solves the **split vote** problem by **randomizing the election timeout** for each server (notice this timeout is reset by certain actions that we will discuss later) because in case these timeouts are permanent to servers we can have in have multiple servers have close timeout that they become a Candidate at the same time.


### Log Replication


Once a Leader has been elected it is responsible for serving client requests, each request contains a command to be executed by the other state machines on each server. The leader operates through the following processes


1. Send Heartbeat


 As soon as a Leader has been elected it sends **heartbeats** to every other server to notify them that leader is alive and reset their **election timeouts,** then periodically sends heartbeat every `HEART_BEAT_INTERVAL`, the paper discusses what values should be suitable for this.


2. AppendEntries


 Whenever a leader receives a new command, it appends it to its log and sends AppendEntries RPC calls in parallel to other servers to replicate this new command, when the entry has been safe replicated (majority of servers) only then the Leader applies the command to its state machine and sends the result to the client.


 If some followers crash the Leader would continue sending them AppendEntry RPC indefinitely **(even after responding to a client)** until it has been replicated.




### Sample Code


Fortunately [MIT-Distributed-Systems](https://www.youtube.com/watch?v=cQP8WApzIQQ&list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB) course provides us with a skeleton to build our own Raft version, we will only be discussing leadership election and log replication, also make sure to check out Distributed System playlist by Eng. Ahmed Farghal [here](https://www.youtube.com/watch?v=s_p3I5CMGJw&list=PLald6EODoOJW3alE1oPAkGF0bHZkPIeTK). You can clone the skeleton from



```
git clone git://g.csail.mit.edu/6.5840-golabs-2023 6.5840

```

The skeleton provides us by means of communication between servers using **RPC calls through Unix sockets,** we will just need to write the logic.


Before jumping into the code let's talk about how are we going to make our code concurrent and avoid race conditions. We will be using [locks](https://go.dev/tour/concurrency/1), [rpcs](https://pkg.go.dev/net/rpc), [channels](https://go.dev/tour/concurrency/2) and [go routines](https://go.dev/tour/concurrency/1) packages.


* Concurrency


 Imagine a server is sending RequestVote RPC to other servers, sending one RPC call then waiting for the answer and then sending the next one and so on is not very efficient, instead we wan't to send them concurrently for performance, and that's what is actually happening inside any distributed system.


 Fortunately Go provides us with [go routines](https://go.dev/tour/concurrency/1) which are excellent to achieve this kind of behavior.


* Locks


 We need **locks** to avoid race conditions for go routines sharing the same **server shared data,** lets look at the example we just provided, sending a VoteRequest looks like the following



```
  rf.peers[server].Call("Raft.RequestVote", args, reply)

```

 Inside the RequestVote RPC we are actually going to access shared data held by multiple routines so we are going to use locks basically everywhere in out code.




You should familiarize yourself with the [extended Raft paper](https://raft.github.io/raft.pdf) and especially **Figure 2, and also how Raft guarantees safety** and correctness. Also refer to [this](https://raft.github.io/) for some visualization of Raft.


**Figure 2** from the paper condenses the algorithm and the messages exchanged between servers (excluding server membership changes & log compaction).


![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700229076079/8e010402-b93a-48ff-a328-d20b9ae6ffbd.jpeg)


1. **State**



```
 type Raft struct {
     mu        sync.Mutex          // Lock to protect shared access to this peer's state
     peers     []*labrpc.ClientEnd // RPC end points of all peers
     me        int                 // this peer's index into peers[]
     dead      int32               // set by Kill()

     currentTerm int
     votedFor    int
     log         []LogCommand
     // volatile all followers
     commitIndex int
     lastApplied int
     // volatile for leaders only
     // gets re-initialised after each election
     nextIndex  []int
     matchIndex []int

     state           state
     lastHeartBeat   time.Time
 }

```

 These are the states for each server, we will need a Mutex Lock to lock our shared data (**data shared for the same server among multiple Go routines**), a peers array which contains the all our servers (please check labrpc folder in the skeleton to get what really is going on), the rest are just states form Figure 2.

* State is just a type definition for an int
```
 type state int
	
 const (
    Follower state = iota
    Candidate
    Leader
 )
	
```
	
* LogCommand looks like this
```
 type LogCommand struct {
	Term    int
	Command interface{}
 }
```
	
just a struct holding the Command's **term** number and the command itself (since we are not going to do any actually commands its just an empty interface)
	
	
* Creating a Raft server and initializing its state
```
 type ApplyMsg struct {
	CommandValid bool
	Command      interface{}
	CommandIndex int
 }
	
 func Make(peers []*labrpc.ClientEnd, me int,
    persister *Persister, applyCh chan ApplyMsg) *Raft {
        DebugPrint("[%v] starting   ", me)
	    rf := &Raft{}
	    rf.peers = peers
	    rf.persister = persister
	    rf.me = me
	    rf.mu = sync.Mutex{}
	    rf.state = Follower
	    rf.lastHeartBeat = time.Now()
	    rf.currentTerm = 0
	    rf.votedFor = -1
	    rf.commitIndex = 0
	    rf.lastApplied = 0
	    rf.log = make([]LogCommand, 1)
	    rf.nextIndex = make([]int, len(rf.peers))
	    rf.matchIndex = make([]int, len(rf.peers))
	    for i := 0; i < len(rf.peers); i++ {
	        rf.matchIndex[i] = 0
	        rf.nextIndex[i] = len(rf.log)
	    }
	    // this will be used by the tester to test our output
	    // whenever a log entry is commited it should send an
	    // applyMsg to a channel
	    rf.applyChannel = applyCh
	    DebugPrint("[%v] initialised", rf.me)
	    // initialize from state persisted before a crash
	    rf.readPersist(persister.ReadRaftState())
	    // start ticker goroutine to start elections
	    go rf.LeaderElection()
	    return rf
    }
	
```
The make function is called whenever we want to create a new server (refer to the config.go which basically initializes servers and run tests), and fires up another go routine which calls **rf.LeaderElection()**.
	
	
The variable **lastHeartBeat** is store the last time this server received a **AppendEntry RPC or a heartbeat if the log is empty**, when we are creating a new server we initialize this with current time.

2. **Election Process**

```
 const (
     ELECTION_INTERVAL = 150
     HEART_BEAT_INTERVAL = 100
 )

 // The ticker go routine starts a new election if this peer hasn't received
 // heartsbeats recently.
 func (rf *Raft) LeaderElection() {
     for rf.killed() == false {
         electionTimeout := ELECTION_INTERVAL + rand.Intn(200)
         startTime := time.Now()
         time.Sleep(time.Duration(electionTimeout) * time.Millisecond)
         rf.mu.Lock() // locking to avoid data races
         if rf.killed() {
             rf.mu.Unlock()
             return
         }
         if rf.lastHeartBeat.Before(startTime) {
             if rf.state != Leader {
                 DebugPrint("[%v] peer : starting election", rf.me)
                 go rf.StartElection()
             }
         }
         // rf.ApplyLog() we don't need this for now
         rf.mu.Unlock()
     }
 }

```

 Whenever a server is created, it should be up and listening for requests (either an AppendEntry/RequestVote RPC). So what we are doing here is basically checking if our server is not dead, **sleep for a random duration of time (timeout)** and check if we haven't heard anything since **lastHeartBeat** we fire another go routine to start an election.


3. **Starting an Election**

```
 type RequestVoteArgs struct {
     Term         int
     CandidateId  int
     LastLogIndex int
     LastLogTerm  int
 }
 // field names must start with capital letters!
 type RequestVoteReply struct {
     Term        int // for candidate to get updated if his term is lower
     VoteGranted bool
 }

 func (rf *Raft) StartElection() {
     rf.mu.Lock()
     rf.ConvertToCandidate()
     DebugPrint("[%v] peer: attempting election at term %v", rf.me, rf.currentTerm)
     term := rf.currentTerm
     candidateId := rf.me
     lastLogIndex := len(rf.log) - 1
     lastLogTerm := rf.log[len(rf.log)-1].Term
     votes := 1
     finished := false
     rf.mu.Unlock()
     // now we need to send concurrent RPCs
     for i := range rf.peers {
         if i == rf.me {
             continue
         }
         go func(server int) {
             rf.mu.Lock()
             request := RequestVoteArgs{
                 Term:         term,
                 CandidateId:  candidateId,
                 LastLogIndex: lastLogIndex,
                 LastLogTerm:  lastLogTerm,
             }
             reply := RequestVoteReply{}
             rf.mu.Unlock()
             ok := rf.sendRequestVote(server, &request, &reply)
             DebugPrint("[%v] voted: %v for candidate[%v]", server, reply.VoteGranted, request.CandidateId)
             rf.mu.Lock()
             defer rf.mu.Unlock()
             if !ok {     // connection issue we don't get a reply
                 return
             }
             if reply.Term > rf.currentTerm {
                 rf.ConvertToFollower(reply.Term)
                 return
             } else if reply.VoteGranted {
                 votes++
             }
             if votes <= len(rf.peers)/2 || finished {
                 return
             }
             finished = true
             if rf.currentTerm != term || rf.state != Candidate {
                 return
             }
             DebugPrint("[%v] is elected leader at term[%v] with votes[%v]", candidateId, term, votes)
             rf.ConvertToLeader()
             for j := 0; j < len(rf.peers); j++ {
                 if j != rf.me {
                     go rf.LeaderOperation(j)
                 }
             }
         }(i)
     }
 }

 func (rf *Raft) ConvertToCandidate() {
     DebugPrint("[%v] converting to candidate", rf.me)
     rf.currentTerm += 1
     rf.votedFor = rf.me
     rf.lastHeartBeat = time.Now() // resetting election timer
     rf.state = Candidate
     // rf.persist() // we don't need this now
 }

 func (rf *Raft) ConvertToFollower(correctTerm int) {
     DebugPrint("[%v] converting to follower", rf.me)
     rf.state = Follower
     rf.votedFor = -1
     rf.lastHeartBeat = time.Now() // reset timeout
     rf.currentTerm = correctTerm
     // rf.persist()
 }

 func (rf *Raft) ConvertToLeader() {
     DebugPrint("[%v] converting to leader", rf.me)
     rf.state = Leader
     rf.lastHeartBeat = time.Now()
     rf.votedFor = -1
     rf.InitialiseLeaderState()
     // rf.persist()
 }
 func (rf *Raft) InitialiseLeaderState() {
     if rf.state != Leader {
         return
     }
     for i := 0; i < len(rf.peers); i++ {
         rf.nextIndex[i] = len(rf.log)
         rf.matchIndex[i] = 0
     }
 }

```

 As we discussed, to start an election the server needs to convert its state to a **Candidate,** from **Figure 2** we can see what args should be sent to the server we are requesting a vote from. We then fire a go routines for each server we are requesting a vote from, the server can reply with one of the following


* The server grants us the vote (we will check the logic for RequestVote RPC in a second). Candidate increments the number of votes granted and check if it received majority of votes, if yes it converts to a leader.
	
	
* The server replies with a higher term than the Candidate's in this case the Candidate converts to being a follower and the election ends.


 There is actually a very important check we have to do before converting to a leader, **during the election process the term for the Candidate might have changed from the term it originally started the election with (there might be another Leader in a higher term that sent an AppendEntries RPC to the Candidate to it converted to a Follow with the new term)**, so even before converting to a leader we have to check that the **current term for the Candidate** is the **same term it started the election** with.


1. **RequestVote RPC**

 From **Figure 2** we can see that the logic for granting votes is through the following


* If the **Candidate's term (args.Term) < server term,** that means that the Candidate is outdated and should convert to a follower and update its term, this happens inside the **StartElection** function we saw earlier, because **reply.Term which is the term for the server that didn't grant the vote is > the Candidate's.**
	
	
* If the **Candidate's term > server term** then it converts to a **follower** (you would think this is redundant but it's actually not, the server receiving the request vote might itself be in election at a lower term or even the leader)
	
	
	+ Check if the Candidate log is at least up to date as the server's
		
		
		 A server should first check their last log **index and term** and compare them to the Candidate's, the server's log is better if it has a **higher term than the Candidate's last log term** **OR they have the same last log term but the server has more log entries for that term (i.e last log index > Candidate's last log index)**, if the server log is better it doesn't grant the vote.

```
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
    rf.mu.Lock()
    defer rf.mu.Unlock()
    DebugPrint("[%v] receives request to vote from [%v] at term[%v]", rf.me, args.CandidateId, args.Term)
    DebugPrint("current term for voter [%v]", rf.currentTerm)
    if args.Term < rf.currentTerm {
        reply.Term = rf.currentTerm
        reply.VoteGranted = false
        return
    }
    if args.Term > rf.currentTerm {
        rf.votedFor = -1
        rf.ConvertToFollower(args.Term)
    } else {
        if rf.votedFor == -1 {
            rf.votedFor = args.CandidateId
            // rf.persist()
        }
    }
    currentLastLogTerm := rf.log[len(rf.log)-1].Term
    currentLastLogIndex := len(rf.log) - 1
    currentLogIsBetter := (currentLastLogTerm > args.LastLogTerm) ||
        (currentLastLogTerm == args.LastLogTerm && currentLastLogIndex > args.LastLogIndex)
    grantVote := (rf.votedFor == -1 || rf.votedFor == args.CandidateId) && !currentLogIsBetter
    if grantVote {
        reply.VoteGranted = true
        reply.Term = args.Term
        return
    }
    reply.VoteGranted = false
}

```

1. **Leader Operation**


 Right after a server becomes the leader we fire a go routine which does the following for each server, it sends an **AppendEntries RPC** every `HEART_BEAT_INTERVAL` which we defined earlier to be `100 ms` , the paper explores suitable values for heartbeats and timeouts, if at some point the server is not a leader anymore we exit from this function.

```
 func (rf *Raft) LeaderOperation(pid int) {
     for rf.killed() == false {
        rf.mu.Lock()
        if rf.state != Leader {
           DebugPrint("[%v] not leader anymore", rf.me)
           rf.mu.Unlock()
           return
        }
        DebugPrint("[%v] leader attempting to send heartbeat every %v milliseconds", rf.me, HEART_BEAT_INTERVAL)
        rf.mu.Unlock()
        go rf.sendAppendEntries(pid)
        time.Sleep(time.Duration(HEART_BEAT_INTERVAL) * time.Millisecond)
     }
 }

```

2. Send AppendEntries RPC


 Following **Figure 2** rules for AppendEntries, we construct our arguments for the RPC call and notice we are only **sending the logs starting from the next index that should be sent to a server (if any)** `Entries: rf.log[rf.nextIndex[server]:]` .


 Based on the reply from the server we do the following


* We get a successful reply (i.e the logs has been replicated successfully) we update the **nextIndex and matchIndex** for that server and **check if we can actually update the Leader's commitIndex**, we can **only update it once the logs has been replicated to majority of servers** an then we apply the log (the first point in Figure 2 in Rules for servers section).
	
	
* The reply has a **higher term** then we convert the current server to a **follower**.
	
	
* There was a mismatch between the Leader's log and the server's, so we **decrement the nextIndex[server] (so we will send older logs) and retry this RPC** until we get a success reply.

```
func (rf *Raft) sendAppendEntries(server int) {
    rf.mu.Lock()
    if rf.state != Leader {
       rf.mu.Unlock()
       return
    }
    previousLogIndex := rf.nextIndex[server] - 1
    DebugPrint("current log: %v", rf.log)
    DebugPrint("prevLogIndex: %v, rf.nextIndex[%v]: %v", previousLogIndex, server, rf.nextIndex)
    args := AppendEntriesArgs{
       LeaderTerm:        rf.currentTerm,
       LeaderId:          rf.me,
       PrevLogIndex:      previousLogIndex,
       PrevLogTerm:       rf.log[previousLogIndex].Term,
       LeaderCommitIndex: rf.commitIndex,
       Entries:           rf.log[rf.nextIndex[server]:],
    }
    reply := AppendEntriesReply{}
    rf.mu.Unlock()
    DebugPrint("[%v] sending heartbeat to [%v]", rf.me, server)
    // remember don't lock while doing RPC call
    ok := rf.peers[server].Call("Raft.AppendEntries", &args, &reply)
    rf.mu.Lock()
    if !ok {
       rf.mu.Unlock()
       return
    }
    if reply.Success {
       rf.nextIndex[server] += len(args.Entries)
       rf.matchIndex[server] = rf.nextIndex[server] - 1
       // update commitIndex
       go func() {
          rf.mu.Lock()
          defer rf.mu.Unlock()
          for i := len(rf.log) - 1; i > rf.commitIndex; i-- {
             count := 0
             for _, matchIndex := range rf.matchIndex {
                if matchIndex >= i {
                   count++
                }
             }
             if count > len(rf.peers)/2 {
                rf.commitIndex = i
                rf.ApplyLog()
                break
             }
          }
       }()
    } else {
       if reply.Term > rf.currentTerm {
          rf.ConvertToFollower(reply.Term)
       } else {
          rf.nextIndex[server]--
       }
    }
    rf.mu.Unlock()
}
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
    rf.mu.Lock()
    defer rf.mu.Unlock()
    DebugPrint("[%v] received heartbeat from [%v]", rf.me, args.LeaderId)
    rf.lastHeartBeat = time.Now()
    reply.Term = rf.currentTerm
    if rf.currentTerm > args.LeaderTerm {
        reply.Success = false
        reply.Term = rf.currentTerm
        return
    }
    if args.LeaderTerm > rf.currentTerm {
        rf.ConvertToFollower(args.LeaderTerm)
    }
    if len(rf.log)-1 < args.PrevLogIndex || rf.log[args.PrevLogIndex].Term != args.PrevLogTerm {
        reply.Success = false
        reply.Term = rf.currentTerm
        return
    }
    if len(rf.log) == args.PrevLogIndex+1 || (len(args.Entries) > 0 && rf.log[args.PrevLogIndex+1].Term != args.Entries[0]Term) {
        rf.log = rf.log[:args.PrevLogIndex+1]
        rf.log = append(rf.log, args.Entries...)
        // rf.persist()
    }
    if args.LeaderCommitIndex > rf.commitIndex {
        rf.commitIndex = min(args.LeaderCommitIndex, len(rf.log)-1)
        rf.ApplyLog()
    }
    reply.Term = rf.currentTerm
    reply.Success = true
}
func (rf *Raft) ApplyLog() {
    if rf.commitIndex > rf.lastApplied {
        go func() {
            rf.mu.Lock()
            defer rf.mu.Unlock()
            for i := rf.lastApplied + 1; i <= rf.commitIndex; i++ {
                var applyMsg ApplyMsg
                // command valid just indicated that this is a new entry
                applyMsg.CommandValid = true
                applyMsg.Command = rf.log[i].Command
                applyMsg.CommandIndex = i
                // sending the applyMsg to the tester in config.go
                rf.applyChannel <- applyMsg
                rf.lastApplied = applyMsg.CommandIndex
            }
        }()
    }
}
```

 Now let's jump into the AppendEntries RPC and see what it do


* **Reset the server's** `rf.lastHeartBeat` to avoid an election


* Check if the server's term is higher than the Leader's, if yes it **reply with false and its term for the Leader to convert to a follower**.


* If the **previousLogIndex's term** doesn't match for both Leader and server, we reply with a false. **Notice this is the case when we are decrementing the nextIndex[server] and retry the RPC call again**.


* If we have an **entry conflicting with a new one** (same index but different term) we should **delete it and every entry after it** `rf.log = rf.log[:args.PrevLogIndex+1]`, then append the new entries to the log and update the commitIndex.




### Testing


The code from MIT acutally consists of tests for multiple labs, leadership election and log replication are labs A and B respectively and you can run the tests by running the following in your terminal, but first `cd to the raft package`


1. Leadership Election


	* `go test -run 2A`
2. Log replication


	* `go test -run 2B`


please check out `test_test.go` file to see what the tests are actually doing, they will help understand why your code is failing. Its a good idea to run your program multiple times and check that all test cases all pass.


## Pitfalls


This might seem easy enough right? just simple loops and if conditions, that's what I thought at first too, I thought its as simple as reading the paper and watchin some lectures and you're good to go, unfortunately it was the opposite.


1. Figure 2


 If you wan't to save yourself long hours of debugging and shedding tears, you **must follow Figure 2 instructions religiously**, every tiny detail matter and will affect the correctness of your program.


2. Dead Locks


 Make sure you're handling your locks correctly, **maybe your trying to lock a lock that hasn't been unlocked or you are not unlocking before doing an RPC call**. You probably should run your program using the race flag `go test -race -run 2A` it will tell you if you have a data race in your program.


3. Debugging


 This can get pretty annoying, because there is so much going on that you actually don't know whats the issue, is it the locks? the logic? heartbeats? timeouts? failures? delayed messages? you will have to think of every single thing there is, and the more you look at your buggy code the more you think it makes sense. My advice is to think of possible corner cases, build each component and see if it works first before advancing to another (you can manipulate the test\_test.go according to your needs).


 I found printing out logs everywhere super helpful, so you should do that it.




## Conclusion


Raft is created to be easy to understand and also performant. It was designed for a large audience to understand the algorithm comfortably and its getting more popular by day, many technologies people use today actually use Raft as an underlying replication layer like Kafka, RabbitMQ, MongoDb and Kubernetes (etcd).


I have learnt a lot from the [raft paper](https://raft.github.io/raft.pdf) and from the exercises provided by [MIT-Course](https://www.youtube.com/watch?v=cQP8WApzIQQ&list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB), despite all the hurdles it was the most interesting programming tasks I have every worked on.


## References


* <http://nil.csail.mit.edu/6.824/2017/papers/raft-extended.pdf>


* <https://www.youtube.com/watch?v=cQP8WApzIQQ&list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB>


* <https://raft.github.io/>


* <https://www.youtube.com/watch?v=s_p3I5CMGJw&list=PLald6EODoOJW3alE1oPAkGF0bHZkPIeTK>




