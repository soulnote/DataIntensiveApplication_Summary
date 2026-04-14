Fault-tolerant systems banane ka best tareeka hai kuch general-purpose abstractions dhundhna jinmein useful guarantees ho, unhe ek baar implement karna, aur phir applications ko un guarantees par rely karne dena. Distributed systems ke liye sabse important abstractions mein se ek hai **consensus**: saare nodes ko kisi baat par agree karwana.

Yeh chapter cover karta hai:

- **Linearizability** — sabse strong commonly used consistency model
- **Ordering Guarantees** — causality, total ordering, total order broadcast
- **Distributed Transactions and Consensus** — 2PC, fault-tolerant consensus algorithms

---

Section 1: Consistency Guarantees  
Zyadatar replicated databases kam se kam **eventual consistency** (convergence) provide karte hain: agar aap write karna band kar dein aur wait karein, to eventually saare read requests same value return karte hain. Yeh bahut weak guarantee hai — yeh nahi batata ki replicas kab converge karenge.

Eventual consistency application developers ke liye mushkil hai kyunki yeh single-threaded program se bahut alag hai jahan aap value assign karte ho aur turant wapas padh lete ho. Stronger consistency models ki performance kharab hoti hai ya fault-tolerance kam hoti hai, lekin use karna aasan hota hai.

Transaction isolation (concurrent transactions se race conditions avoid karna) aur distributed consistency (delays aur faults ke bawajood replica state coordinate karna) mostly independent concerns hain, lekin kuch overlap hai.

---

Section 2: Linearizability  
Idea: system aisa lagne lage jaise data ki sirf ek copy ho, aur us par saare operations atomic ho. Ise **atomic consistency**, **strong consistency**, **immediate consistency**, ya **external consistency** bhi kehte hain.

Linearizability ek **recency guarantee** hai: jaise hi ek client successfully write complete kare, saare clients jo database padh rahe hain unhe abhi likhi gayi value dikhni chahiye.

**Alice and Bob Example**  
<img width="2880" height="2063" alt="image" src="https://github.com/user-attachments/assets/6aa0bafa-3544-4992-81e4-9a9895720eac" />


**Figure 9-1: This system is not linearizable, causing football fans to be confused**  
(Aapki pehli image: Referee final score insert karta hai, Alice leader se padh leti hai, Bob follower se padhta hai aur use purana score dikhta hai.)

Alice final score dekhti hai, Bob ko batati hai. Bob refresh karta hai lekin uski request lagging replica mein jaati hai — use game abhi bhi ongoing dikhta hai. Kyunki Bob ki query Alice ka result sunne ke baad shuru hui, yeh linearizability violation hai.

**What Makes a System Linearizable?**  
<img width="2880" height="829" alt="image" src="https://github.com/user-attachments/assets/0fc09a50-3caf-4465-b944-b72cb89400b1" />


**Figure 9-2: If a read request is concurrent with a write request, it may return either the old or the new value**  
(Doosri image: Client A read 0, Client C write 1, Client B read concurrent to write may get 0 or 1.)

Operations on a register: `read(x) ⇒ v`, `write(x, v) ⇒ r`, `cas(x, v_old, v_new) ⇒ r`  
Agar read write ke saath concurrent hai, to wo purani ya nayi value return kar sakta hai.  
Lekin ek baar koi read nayi value return kare, to **saare subsequent reads** bhi nayi value return karni chahiye — value write ke dauran atomically "flip" hoti hai.

<img width="2880" height="829" alt="image" src="https://github.com/user-attachments/assets/cb91851d-2031-4c2f-a571-2894934f4d42" />


**Figure 9-4: Visualizing the points in time at which reads and writes appear to have taken effect**  
(Operation markers ke beech lines hamesha time mein aage badhni chahiye, kabhi peeche nahi.)

Requirement: lines joining operation markers hamesha time mein aage badhein, kabhi peeche nahi.

**Linearizability vs Serializability**  
- **Serializability** — Transactions ka isolation property (multiple objects). Guarantee karta hai ki transactions aise behave karein jaise kisi serial order mein execute hue. Wo order actual execution order se alag ho sakta hai.  
- **Linearizability** — Single register (individual object) ke reads aur writes par recency guarantee. Operations ko transactions mein group nahi karta.  

**Strict serializability (strong-1SR)** = serializability + linearizability. 2PL ya actual serial execution achieve karte hain. SSI linearizable nahi hai (consistent snapshot se padhta hai).

**Relying on Linearizability**  
- **Locking and leader election** — Saare nodes ko agree karna hoga ki lock kis node ke paas hai / leader kaun hai. ZooKeeper aur etcd iske liye consensus algorithms use karte hain. Oracle RAC per disk page linearizable locks use karta hai.  
- **Constraints and uniqueness guarantees** — Unique usernames, non-negative bank balances, no double-booking. Ek single up-to-date value chahiye jis par saare nodes agree karein. (Kuch constraints loosely treat kiye ja sakte hain — e.g., flight overbooking.)  
- **Cross-channel timing dependencies** — Jab multiple communication channels ho (e.g., file storage + message queue), linearizability unke beech race conditions prevent karta hai.
<img width="2880" height="922" alt="image" src="https://github.com/user-attachments/assets/4f6d2f25-e783-404b-866d-42740a29cb76" />


**Figure 9-5: The web server and image resizer communicate both through file storage and a message queue**  
(Chauthi image: Web server file storage mein upload karta hai, message queue mein message bhejta hai, resizer dono se padhta hai.)

**Implementing Linearizable Systems**  

| Replication Method                | Linearizable?                                                                                                           |
|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| **Single-leader**                 | Potentially (agar reads leader ya synchronous followers se ho). Snapshot isolation ya concurrency bugs ho to nahi. Failover linearizability violate kar sakta hai. |
| **Consensus algorithms**          | Yes — ZooKeeper, etcd. Split brain aur stale replicas prevent karne ke measures hote hain.                              |
| **Multi-leader**                  | Not linearizable (multiple nodes par concurrent writes, async replication).                                              |
| **Leaderless (Dynamo-style)**     | Probably not. LWW with clocks almost certainly nonlinearizable. Sloppy quorums ruin karte hain. Strict quorums bhi nonlinearizable ho sakte hain. |

**Linearizability and Quorums**  
<img width="2880" height="1602" alt="image" src="https://github.com/user-attachments/assets/dab9545d-8bdf-4d3e-803b-e73b4acfee20" />


**Figure 9-6: A nonlinearizable execution, despite using a strict quorum**  
(Paanchvi image: Writer write karta hai, replica 3 update nahi hota, Reader A purani value padhta hai jabki Reader B nayi value padh chuka hai.)

*w + r > n* ke bawajood, race conditions strict quorums ko nonlinearizable bana sakti hain. Dynamo-style quorums ko linearizable banane ke liye **synchronous read repair** (results return karne se pehle) + writer ko likhne se pehle latest state padhna padta hai — performance reduce hoti hai. Linearizable compare-and-set aise implement nahi ho sakta — consensus algorithm chahiye.

**The Cost of Linearizability**  
<img width="2880" height="1294" alt="image" src="https://github.com/user-attachments/assets/9551f91b-1ca5-4ad0-9cdb-f09fad953330" />


**Figure 9-7: A network interruption forcing a choice between linearizability and availability**  
(Chhathi image: Do datacenters ke beech network partition, application consistency ya availability choose kare.)

Agar aapki application linearizability require kare aur kuch replicas disconnected ho → wo replicas unavailable ho jaate hain (wait karna hoga ya error return karna hoga).  
Agar aapki application linearizability require nahi kare → har replica independently requests process kar sakta hai → network problems ke dauran available rehta hai.

**The CAP Theorem**  
Popularly: Consistency, Availability, Partition tolerance — 3 mein se 2 pick karo. Lekin yeh misleading hai: network partitions ek **fault** hain, choice nahi. Wo honge hi.  
Better phrasing: **Consistent or Available when Partitioned**.  
CAP ka scope bahut narrow hai (sirf linearizability + network partitions). Isse zyada precise results ne supersede kar diya hai aur aaj yeh mostly historical interest ka hai. Best avoided.

**Linearizability and Network Delays**  
Modern multi-core CPU ki RAM bhi linearizable nahi hai (har core ka apna cache hota hai, asynchronously main memory update hoti hai). Reason: performance, not fault tolerance.  
Linearizability slow hai — response time kam se kam network delays ki uncertainty ke proportional hota hai. Faster algorithm for linearizability exist nahi karta. Weaker consistency models bahut fast ho sakte hain.

---

Section 3: Ordering Guarantees  

**Ordering and Causality**  
Ordering **causality** preserve karne mein help karta hai (cause effect se pehle aata hai). Poori book mein examples:

- Consistent prefix reads — answer question se pehle dekhna
- Replication — update to a row that doesn't exist yet
- Concurrent writes — happens-before relationship
- Snapshot isolation — consistent with causality (saare causally prior operations visible hain)
- Write skew — causally dependent on observation of current state
- Alice and Bob — Bob ko Alice ka exclamation sunne ke baad score dekhna chahiye

**Causal Order is Partial, Not Total**  
- **Total order** — Koi bhi do elements compare kiye ja sakte hain (e.g., natural numbers: 5 < 13).  
- **Partial order** — Kuch elements incomparable hain (e.g., mathematical sets: {a,b} vs {b,c}).

|                      | Linearizability                                               | Causality                                                     |
|----------------------|---------------------------------------------------------------|---------------------------------------------------------------|
| **Order type**       | Total order (single timeline, saare operations ordered)       | Partial order (concurrent operations incomparable hain)        |
| **Concurrency**      | No concurrent operations (single copy, atomic)                | Concurrent operations exist (timeline branches and merges, jaise Git) |

**Linearizability Implies Causality**  
Koi bhi linearizable system causality correctly preserve karta hai. Lekin linearizability performance aur availability harm karta hai. Acchi khabar: **causal consistency** strongest possible consistency model hai jo network delays ki wajah se slow nahi hota aur network failures ke dauran available rehta hai. Bahut saare systems jinhe linearizability chahiye lagta hai, actually sirf causal consistency chahiye hoti hai.

**Sequence Number Ordering**  
Saare causal dependencies track karna impractical hai. Instead, **sequence numbers** ya **timestamps** from a **logical clock** use karo (counter jo har operation ke liye increment ho, physical clock nahi).

- Single-leader replication mein: leader har operation ke liye counter increment karta hai → total order consistent with causality.  
- Bina single leader ke, noncausal generators (odd/even counters, physical timestamps, block allocation) causality ke saath consistent nahi hain.

**Lamport Timestamps**  
<img width="2880" height="1210" alt="image" src="https://github.com/user-attachments/assets/f35e094d-46ea-4089-a1c6-3b6b237d1786" />


**Figure 9-8: Lamport timestamps provide a total ordering consistent with causality**  
(Saatvi image: Nodes counters maintain karte hain, max seen include karte hain, (counter, nodeID) pairs.)

Ek pair `(counter, node ID)`. Pehle counter se compare karo, phir node ID tiebreaker ki tarah.  
**Key idea:** har node aur client ab tak dekha gaya maximum counter value track karta hai aur har request mein include karta hai. Jab node kisi request mein higher counter dekhta hai, to turant apna counter us maximum tak badha deta hai.  
Yeh ensure karta hai ki ordering causality ke saath consistent hai.

**Lamport timestamps vs version vectors:** Lamport timestamps total order enforce karte hain lekin concurrent aur causally dependent operations distinguish nahi kar sakte. Version vectors kar sakte hain.

**Timestamp Ordering Is Not Sufficient**  
Lamport timestamps causality ke saath consistent total order define karte hain, lekin bahut saari problems ke liye kaafi nahi hain (e.g., unique username registration). Total order tabhi emerge hota hai jab saare operations collect ho jaayein — ek node real-time mein nahi jaan sakta ki koi doosra node concurrently same username claim kar raha hai lower timestamp ke saath. Aapko jaanna hoga ki order kab **finalized** hua → **total order broadcast**.

**Total Order Broadcast**  
Ise **atomic broadcast** bhi kehte hain. Nodes ke beech messages exchange karne ka protocol with two safety properties:

1. **Reliable delivery** — Agar message ek node ko deliver hua, to saare nodes ko deliver hoga.  
2. **Totally ordered delivery** — Messages har node ko **same order** mein deliver hote hain.

**Uses:**  
- **Database replication (state machine replication)** — Har replica same writes same order mein process karta hai.  
- **Serializable transactions** — Har message ek deterministic transaction hai; saare nodes same order mein process karte hain.  
- **Lock service with fencing tokens** — Har lock request log mein append hoti hai; sequence numbers fencing tokens ka kaam karte hain (ZooKeeper ka `zxid`).

**Implementing Linearizable Storage Using Total Order Broadcast**  
Total order broadcast asynchronous hai (guarantee nahi ki message kab deliver hoga). Linearizability recency guarantee hai. Lekin aap TOB ke upar linearizable storage bana sakte ho:

1. Username claim karne ke liye log mein message append karo.  
2. Log padho, wait karo jab tak tumhara message wapas deliver na ho.  
3. Agar us username ke liye pehla message tumhara hai → success. Warna → abort.  

**Linearizable reads ke liye:** reads ko log ke through sequence karo, ya latest log position query karo aur wait karo, ya synchronously updated replica se padho (chain replication).

**Implementing Total Order Broadcast Using Linearizable Storage**  
Atomic increment-and-get ke saath linearizable register use karo. Har message ke liye, register increment karo aur sequence number attach karo. Lamport timestamps ke opposite, in numbers mein **koi gaps nahi** hote — agar aapke paas message 4 hai aur message 6 receive hota hai, to aap jaante ho ki message 5 ka wait karna hai.

**The Equivalence**  
Linearizable compare-and-set register, total order broadcast, aur consensus — teeno **equivalent** hain. Agar aap ek solve kar sakte ho, to use doosre ke solution mein transform kar sakte ho.

---

Section 4: Distributed Transactions and Consensus  
**Consensus:** kai nodes ko kisi baat par agree karwana. Important for:

- **Leader election** — Saare nodes ko agree karna hoga leader kaun hai (split brain avoid kare → data loss).
- **Atomic commit** — Saare nodes ko distributed transaction ke outcome par agree karna hoga (saare commit ya saare abort).

**The FLP Impossibility Result**  
Fischer, Lynch, aur Paterson ne prove kiya ki **koi algorithm nahi** jo hamesha consensus reach kare agar node crash ka risk ho — **asynchronous system model** mein (no clocks or timeouts). Lekin agar algorithm timeouts ya random numbers use kar sakta hai, to consensus solvable ho jaata hai. Distributed systems usually practice mein consensus achieve kar sakte hain.

### 4.1 Two-Phase Commit (2PC)  
Multiple nodes ke across atomic commit solve karne ka sabse common tareeka. (Note: 2PC ≠ 2PL — completely different things.)
<img width="1018" height="372" alt="image" src="https://github.com/user-attachments/assets/d81f18a8-bbbf-4288-bea6-af791e3b015a" />


**Figure 9-9: A successful execution of two-phase commit (2PC)**  
(Aathvi image: Coordinator prepare bhejta hai, participants yes vote karte hain, coordinator commit karta hai.)

Ek **coordinator** (transaction manager) use karta hai jo single-node transactions mein nahi hota.

**The Protocol**  
1. Application coordinator se globally unique transaction ID request karta hai.  
2. Application har participant par single-node transaction shuru karta hai, global transaction ID se attached.  
3. Jab commit karne ko ready ho, coordinator saare participants ko **prepare** request bhejta hai.  
4. Har participant ensure karta hai ki wo **definitely commit** kar sakta hai under all circumstances (data disk par likhta hai, constraints check karta hai). "Yes" reply karke, wo abort karne ka right surrender kar deta hai lekin abhi commit nahi karta.  
5. Coordinator definitive **commit** ya **abort** decision leta hai (sirf commit agar sab ne "yes" vote diya). Decision apne transaction log mein disk par likhta hai — yeh **commit point** hai.  
6. Coordinator sabhi participants ko commit/abort request bhejta hai. Agar fail ho jaaye, to forever retry karta hai — decision **irrevocable** hai.

Do **"points of no return"**: participant "yes" vote kare (commit karne ka promise), aur coordinator decide kare (irrevocable).

**Coordinator Failure**  
<img width="2880" height="1006" alt="image" src="https://github.com/user-attachments/assets/5d1cf330-c87d-4578-a33a-61009551efe9" />


**Figure 9-10: The coordinator crashes after participants vote 'yes.' Database 1 does not know whether to commit or abort**  
(Nauvi image: Coordinator crash ke baad participant stuck in doubt.)

Agar coordinator participants ke "yes" vote karne ke baad lekin commit/abort request bhejne se pehle crash ho jaaye, to participants **stuck in doubt** (uncertain) ho jaate hain. Wo unilaterally commit ya abort nahi kar sakte. Complete karne ka ek hi tareeka hai: coordinator ke recover hone ka wait karo aur uska transaction log padho.

**Three-Phase Commit (3PC)**  
Ek alternative jo theoretically nonblocking hai, lekin bounded network delay aur bounded response times assume karta hai — zyadatar practical systems mein kaam nahi karta. 2PC coordinator failure problem ke bawajood use hota rehta hai.

### 4.2 Distributed Transactions in Practice  
Do types often conflated:

- **Database-internal** — Same distributed database ke nodes (e.g., VoltDB, MySQL Cluster). Optimized internal protocols use kar sakte hain. Often quite well kaam karte hain.
- **Heterogeneous** — Different technologies (e.g., do alag databases, ya database + message broker). Much more challenging.

**Exactly-Once Message Processing**  
Queue se message ko processed acknowledge kiya ja sakta hai **if and only if** database transaction for processing it successfully committed ho. Message acknowledgment aur database writes dono atomically commit hote hain ek hi distributed transaction mein. Agar koi fail ho, to dono abort ho jaate hain → message safely redelivered.

**XA Transactions**  
**X/Open XA (eXtended Architecture)** — 1991 se ek standard for implementing 2PC across heterogeneous technologies. Supported by PostgreSQL, MySQL, DB2, SQL Server, Oracle, ActiveMQ, HornetQ, MSMQ, IBM MQ.

XA ek **C API** hai (network protocol nahi) transaction coordinator ke saath interface karne ke liye. Coordinator aksar same application process mein library hota hai. Participants track karta hai aur commit/abort decisions ke liye local disk log use karta hai.

**Holding Locks While In Doubt**  
Critical problem: database transactions modified rows par **row-level locks** leti hain. Ye locks tab tak release nahi ho sakte jab tak transaction commit ya abort na ho. Agar coordinator crash ho jaaye aur 20 minute restart hone mein lage, to locks 20 minute tak held rahenge. Agar coordinator ka log lost ho jaaye, to locks **forever** held rahenge (until manual resolution). Doosri transactions jo same data need karti hain **blocked** ho jaati hain.

**Recovering from Coordinator Failure**  
**Orphaned in-doubt transactions** tab hoti hain jab coordinator outcome determine nahi kar sakta (e.g., corrupted log). Ye forever baithi rehti hain, locks hold karti hain. Ek hi raasta: administrator manually decide kare commit karna hai ya roll back — production outage ke dauran high stress mein.

**Heuristic decisions** — Emergency escape hatch allowing participant to unilaterally decide. "Heuristic" is a euphemism for *probably breaking atomicity*.

**Limitations of Distributed Transactions**  
- Coordinator **single point of failure** hai (often highly available by default nahi hota).  
- Application servers **stateful** bana deta hai (coordinator logs crucial durable state hain).  
- XA **lowest common denominator** hai — deadlocks across systems detect nahi kar sakta, SSI ke saath kaam nahi karta.  
- 2PC requires **all** participants to respond → tendency to amplify failures (fault tolerance ke against jaata hai).

### 4.3 Fault-Tolerant Consensus  
Ek ya zyada nodes values propose karte hain, aur consensus algorithm unmein se ek value decide karta hai.

**Properties**  
- **Uniform agreement** — Koi do nodes alag decide nahi karte.  
- **Integrity** — Koi node do baar decide nahi karta.  
- **Validity** — Agar node value *v* decide kare, to *v* kisi node ne propose kiya tha.  
- **Termination** — Har node jo crash nahi hota eventually kuch value decide karta hai (fault-tolerance property — liveness property).

Termination property ka matlab hai ki **2PC consensus satisfy nahi karta** (wo coordinator ke wait karte stuck ho sakta hai). Consensus ke liye kam se kam **majority of nodes** functioning honi chahiye.

Safety properties (agreement, integrity, validity) hamesha meet hoti hain, chahe majority fail ho jaaye. Large-scale outage processing rok deta hai lekin consensus system corrupt nahi kar sakta.

**Consensus Algorithms and Total Order Broadcast**  
Best-known: **Viewstamped Replication (VSR)**, **Paxos**, **Raft**, **Zab**. Ye ek sequence of values decide karte hain (total order broadcast), sirf single value nahi.

- **VSR** (1988, Oki and Liskov) — Earliest fault-tolerant consensus algorithms mein se ek. Primary replica propose karta hai; agar primary fail ho, to view change protocol naya primary elect karta hai incremented view number ke saath. 2012 mein Liskov aur Cowling ne revisit kiya.
- **Paxos** (Lamport) — Most widely studied. Single-decree Paxos ek value decide karta hai; Multi-Paxos sequence of values decide karne ke liye optimize karta hai. Correctly implement karna notoriously difficult hai.
- **Raft** (2014, Ongaro and Ousterhout) — Explicitly understandability ke liye design kiya gaya. Leader epochs ke liye **term number** use karta hai. Widely adopted in practice (etcd, CockroachDB, TiKV).
- **Zab** (ZooKeeper Atomic Broadcast) — ZooKeeper use karta hai. Raft similar hai lekin independently developed.

In algorithms mein kaafi similarities hain, lekin same nahi hain. VSR, Raft, aur Zab directly total order broadcast implement karte hain. Paxos ke case mein, yeh optimization **Multi-Paxos** kehlata hai.

Total order broadcast = repeated rounds of consensus (har round next message decide karta hai deliver karne ke liye):

- Agreement → saare nodes same messages same order mein deliver karein
- Integrity → messages duplicated nahi
- Validity → messages corrupted ya fabricated nahi
- Termination → messages lost nahi

**Epoch Numbering and Quorums**  
Saare consensus protocols internally **leader** use karte hain, lekin guarantee nahi karte ki leader unique hai. Instead: **epoch number** (Paxos mein ballot number, VSR mein view number, Raft mein term number) guarantee karta hai ki leader **unique within each epoch** hai.

- Jab leader dead maana jaaye → vote karo naya leader elect karne ke liye incremented epoch number ke saath.  
- Kuch bhi decide karne se pehle, leader ko **quorum** (typically majority) se votes collect karne hote hain.  
- Do rounds of voting: ek leader choose karne ke liye, ek proposal par vote karne ke liye. Quorums **overlap** karna chahiye — kam se kam ek node dono votes mein participate kare.

**Key differences from 2PC:** coordinator **elected** hota hai (fixed nahi), aur sirf **majority** chahiye (har participant nahi).

**Limitations of Consensus**  
- Voting ek tarah ki synchronous replication hai → performance cost.  
- Strict majority require karta hai (1 failure tolerate karne ke liye 3 nodes, 2 failures ke liye 5 nodes).  
- Fixed set of nodes assume karta hai (dynamic membership extensions less well understood hain).  
- Failure detection ke liye timeouts par rely karta hai → variable network delays wale environments mein frequent false leader elections → terrible performance.  
- Network problems ke liye sensitive hai (e.g., Raft leadership bounce kar sakta hai do nodes ke beech agar ek network link unreliable ho).

### 4.4 Membership and Coordination Services  
**ZooKeeper** aur **etcd** small amounts of data (jo memory mein fit ho) hold karne ke liye design kiye gaye hain, fault-tolerant total order broadcast use karke replicated.

ZooKeeper Google ke **Chubby lock service** (2006, Mike Burrows) par modeled hai. Chubby sirf consensus nahi implement karta balki lock service with advisory locks, small-file storage, aur session/lease management bhi. Google internally use karta hai leader election, naming, aur configuration ke liye. ZooKeeper open-source equivalent hai, same ideas implement karta hai interesting set of additional features ke saath.

**ZooKeeper Features**  
- **Linearizable atomic operations** — Atomic compare-and-set distributed locks implement karne ke liye (leases with expiry time).  
- **Total ordering of operations** — Monotonically increasing transaction ID (`zxid`) aur version number (`cversion`) fencing tokens ka kaam karte hain.  
- **Failure detection** — Clients long-lived sessions maintain karte hain periodic heartbeats ke saath. Agar heartbeats session timeout ke beyond cease ho jaayein → session dead declared → locks automatically release (ephemeral nodes).  
- **Change notifications** — Clients changes ke liye watch kar sakte hain (naye nodes join, sessions time out). Frequent polling avoid karta hai.

**Use Cases**  
- **Allocating work to nodes** — Leader/primary election, partition assignment, rebalancing. Atomic operations, ephemeral nodes, aur notifications use karta hai.  
- **Service discovery** — Services apne network endpoints ZooKeeper mein register karti hain jab start up hoti hain. (DNS traditional approach hai aur consensus require nahi karta; lekin agar aapka consensus system already leader jaanta hai, to service discovery ke liye bhi use karna sense banata hai.)  
- **Membership services** — Determine karna ki cluster mein currently kaunse nodes active aur live members hain. Failure detection ko consensus ke saath couple karne se nodes agree kar sakte hain ki kaun alive hai aur kaun dead.

ZooKeeper fixed number of nodes (usually 3 ya 5) par run hota hai aur un nodes ke beech majority votes perform karta hai, while supporting potentially large number of clients — coordination work ko external service ko "outsource" karna.

Used by: HBase, Hadoop YARN, OpenStack Nova, Kafka. MongoDB apna khud ka config server use karta hai. LinkedIn ka Espresso Helix use karta hai (jo ZooKeeper par built hai).

---

Summary  

**Equivalent Problems (All Reducible to Consensus)**  

| Problem                                      | What is "decided"                                         |
|----------------------------------------------|-----------------------------------------------------------|
| Linearizable compare-and-set registers       | Whether to set the value based on current value           |
| Atomic transaction commit                    | Whether to commit or abort                                |
| Total order broadcast                        | The order in which to deliver messages                    |
| Locks and leases                             | Which client acquired the lock                            |
| Membership/coordination service              | Which nodes are alive or dead                             |
| Uniqueness constraint                        | Which conflicting write to allow                          |

**Three Ways to Handle Leader Failure**  
1. **Wait for recovery** — System blocked in the meantime (XA/JTA coordinators). Termination satisfy nahi karta.  
2. **Manual failover** — Humans choose a new leader. "Consensus by act of God." Human speed se limited.  
3. **Automatic leader election** — Requires a consensus algorithm (VSR, Paxos, Raft, Zab). **Recommended approach.**

**Key Insights**  
- **Linearizability** replicated data ko single copy with atomic operations jaise appear karata hai. Samajhna aasan lekin slow (khaas kar network delays ke saath). CAP theorem mostly historical interest ka hai.  
- **Causal consistency** strongest model hai jo network delays ki wajah se slow nahi hota aur network failures ke dauran available rehta hai. Bahut saare systems jinhe linearizability chahiye lagta hai, actually sirf causal consistency chahiye hoti hai.  
- **Lamport timestamps** causality ke saath consistent total order provide karte hain lekin real-time decisions ke liye kaafi nahi hain (order fact ke baad hi emerge hota hai).  
- **Total order broadcast** consensus ke equivalent hai. Database replication, serializable transactions, aur lock services ke liye foundation provide karta hai.  
- **2PC** atomic commit provide karta hai lekin **blocking protocol** hai (coordinator failure → participants stuck in doubt holding locks).  
- **Fault-tolerant consensus** (Raft, Paxos, Zab) epoch numbering aur quorum voting use karta hai. Majority of nodes require karta hai. Total order broadcast aur linearizable operations provide karta hai.  
- **ZooKeeper/etcd** consensus, failure detection, aur membership services ko dedicated coordination service ko outsource karte hain.
