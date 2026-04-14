Distributed systems ke saath kaam karna fundamentally alag hai single computer par software likhne se — cheezein galat hone ke bahut saare naye aur exciting tareeke hain. Yeh chapter un problems ka thoroughly pessimistic overview hai jo occur ho sakte hain: unreliable networks, unreliable clocks, aur process pauses. Distributed systems mein, suspicion, pessimism, aur paranoia pay off karte hain.

---

Section 1: Faults and Partial Failures  
Single computer par, cheezein fairly predictable hain: ya to kaam karta hai ya nahi. Agar internal fault ho, to hum prefer karte hain total crash rather than returning a wrong result. Computers fuzzy physical reality chhupakar ek idealized system model present karte hain mathematical perfection ke saath.

Distributed system mein, kuch parts broken ho sakte hain jabki baaki fine kaam karein — yeh **partial failure** hai. Partial failures **nondeterministic** hain: same operation kabhi kaam kare aur kabhi fail ho. Aapko pata bhi nahi chal sakta ki kuch succeed hua ya nahi.

**Cloud Computing vs Supercomputing**

|                           | Supercomputing (HPC)                                           | Cloud Computing                                               |
|---------------------------|----------------------------------------------------------------|---------------------------------------------------------------|
| **Fault handling**        | Checkpoint to storage; failure par poori cluster stop; restart from checkpoint | Faults tolerate karo; keep running; rolling upgrades         |
| **Availability**          | Offline batch jobs (stop/restart kar sakte hain)               | Online services (users ko low latency mein serve karna hai)   |
| **Hardware**              | Specialized, reliable hardware; RDMA                           | Commodity machines; higher failure rates                      |
| **Network**               | Specialized topologies (meshes, toruses)                       | IP and Ethernet (Clos topologies)                             |
| **Scale**                 | Agar kuch hamesha broken ho, to system recover karne mein time waste karta hai | Kharab VMs kill karo, naye request karo                       |

Hazaaron nodes wale system mein, kuch na kuch hamesha broken hota hai. Fault handling software design ka hissa hona chahiye.

**Building a Reliable System from Unreliable Components**  
- Error-correcting codes noisy channel par accurate transmission allow karte hain.  
- TCP unreliable IP ke upar reliable transport provide karta hai (lost packets retransmit, duplicates eliminate, reorder).  
- System apne parts se zyada reliable ho sakta hai, lekin hamesha ek limit hoti hai.

---

Section 2: Unreliable Networks  
Zyadatar distributed systems **shared-nothing architecture** with **asynchronous packet networks** use karte hain. Jab aap request bhejte ho aur response expect karte ho, bahut kuch galat ho sakta hai:
<img width="2880" height="898" alt="image" src="https://github.com/user-attachments/assets/979f20a0-2ab9-4cfe-a5f7-da50a54d86e6" />


**Figure 8-1: If you send a request and don't get a response, it's not possible to distinguish whether (a) the request was lost, (b) the remote node is down, or (c) the response was lost**  
(Aapki pehli image: Client network ke across service ko request bhejta hai, node unresponsive ho jaata hai, client ko pata nahi kya hua.)

- Aapki request lost ho gayi ho (kisi ne cable unplug kar di).  
- Aapki request queue mein wait kar rahi ho (network ya recipient overloaded).  
- Remote node fail ho gaya ho (crashed ya powered down).  
- Remote node temporarily stop ho gaya ho (GC pause) lekin baad mein respond karega.  
- Remote node ne request process kar li, lekin response lost ho gaya.  
- Remote node ne request process kar li, lekin response delayed hai.  

Sender nahi bata sakta ki packet deliver hua ya nahi. Ek hi option hai: **timeout** — wait karna chhod do aur assume karo response nahi aayega. Lekin tab bhi aap nahi jaante ki request pahunchi ya nahi.

**Network Faults in Practice**  
- Ek study ne paaya ki medium-sized datacenter mein ~12 network faults per month hote hain.  
- Redundant networking gear utna help nahi karta jitni umeed hoti hai (human error — outages ka bada cause — se guard nahi karta).  
- EC2 frequent transient network glitches ke liye notorious hai. Sharks undersea cables kaat dete hain.  
- Chahe faults rare ho, aapke software ko unhe handle karna hoga. Testing mein deliberately network problems trigger karo (Chaos Monkey).

**Detecting Faults**  
- Load balancers ko dead nodes ko requests bhejna band karna hota hai.  
- Single-leader replication mein, followers ko failover ke liye leader failure detect karna hota hai.  
- Kuch feedback possible hai: RST/FIN packets agar process crash ho, notification scripts agar process crash ho lekin OS chal raha ho, routers se ICMP Destination Unreachable.  
- Lekin aap rapid feedback par count nahi kar sakte. Agar aap sure hona chahte ho ki request succeed hui, to aapko application se hi positive response chahiye.

**Timeouts and Unbounded Delays**  
- Lamba timeout → node ko dead declare karne mein der.  
- Chhota timeout → galat tareeke se node dead declare karne ka risk (temporary slowdown ke dauran premature failover situation aur kharab kar deta hai).  
- Agar network maximum delay *d* guarantee kare aur nodes maximum handling time *r* guarantee karein, to *2d + r* reasonable timeout hota. Lekin asynchronous networks mein **unbounded delays** hote hain aur servers maximum handling time guarantee nahi kar sakte.

**Network Congestion and Queueing**  
<img width="2880" height="1081" alt="image" src="https://github.com/user-attachments/assets/b7f73624-a21a-4ab3-9bad-24abb9bd7521" />


**Figure 8-2: If several machines send network traffic to the same destination, its switch queue can fill up**  
(Doosri image: Network switch mein input links, switch fabric, output queues dikhaye gaye hain.)

Packet delays mein variability mostly queueing ki wajah se hoti hai:  
- **Network switch queue** — Multiple nodes same destination ko bhej rahe hain. Agar queue bhar jaaye, packets drop ho jaate hain.  
- **OS queue** — Saare CPU cores busy hain; incoming requests queue mein wait karti hain jab tak application handle na kare.  
- **TCP flow control** — Network overload avoid karne ke liye sending rate limit karta hai.  
- **TCP retransmission** — Lost packets timeout ke baad retransmit hote hain, delay add karte hain.

**TCP vs UDP**  
- UDP flow control aur retransmission avoid karta hai → lower aur more predictable latency, lekin unreliable.  
- Latency-sensitive applications jahan delayed data worthless ho (VoIP, videoconferencing) ke liye accha hai.

**Phi Accrual Failure Detector**  
Fixed timeouts ki jagah, systems continuously response times aur jitter measure kar sakte hain, aur automatically timeouts adjust kar sakte hain. Akka aur Cassandra use karte hain.

**Synchronous vs Asynchronous Networks**  
- Telephone networks **circuit switching** use karte hain: har call ke liye fixed bandwidth reserve hoti hai. No queueing → bounded delay. Lekin idle hone par bandwidth waste hoti hai.  
- Datacenter networks aur internet **packet switching** use karte hain: bursty traffic ke liye optimized. Bandwidth dynamically shared → queueing → unbounded delays. Lekin utilization maximize hoti hai.  
- **Quality of Service (QoS)** aur admission control packet networks par circuit switching emulate kar sakte hain, lekin yeh currently multi-tenant datacenters ya public internet mein enabled nahi hai.

---

Section 3: Unreliable Clocks  
Har machine ka apna clock hota hai (quartz crystal oscillator). Clocks **NTP (Network Time Protocol)** ke through synchronize kiye ja sakte hain, lekin synchronization network delay se limited hoti hai.

**Time-of-Day Clocks vs Monotonic Clocks**

|                          | Time-of-Day Clock                                           | Monotonic Clock                                              |
|--------------------------|-------------------------------------------------------------|--------------------------------------------------------------|
| **Returns**              | Current date and time (wall-clock time, seconds since epoch) | Arbitrary value (e.g., nanoseconds since boot)               |
| **Synchronized**         | Yes, via NTP. Adjust hone par backward jump kar sakta hai.   | No synchronization needed. NTP rate slew kar sakta hai lekin kabhi jump nahi karta. |
| **Use for**              | Points in time (yeh kab hua?)                               | Durations (ismein kitna time laga?)                          |
| **Cross-machine comparison** | Meaningful (agar synchronized ho)                         | Meaningless                                                  |
| **Suitable for measuring elapsed time** | No (jump kar sakta hai)                           | Yes                                                          |
| **API**                  | `clock_gettime(CLOCK_REALTIME)`, `System.currentTimeMillis()` | `clock_gettime(CLOCK_MONOTONIC)`, `System.nanoTime()`        |

**Clock Synchronization and Accuracy**  
NTP aur hardware clocks fickle hain:  
- Quartz clocks drift karte hain (up to 200 ppm = 6 ms drift agar din mein ek baar sync ho, 17 seconds agar ek baar sync ho).  
- NTP ko bina kisi ke notice kiye firewall block kar sakta hai.  
- NTP accuracy network delay se limited hai (internet par minimum ~35 ms error).  
- Kuch NTP servers galat ya misconfigured hote hain.  
- **Leap seconds** (59 ya 61 seconds in a minute) ne bahut saare bade systems crash kar diye hain. Workaround: **smearing** (din bhar gradual adjustment).  
- **VM pauses** — Jab CPU core shared ho, har VM dusre ke run hone ke dauran tens of milliseconds ke liye paused hota hai.  
- User-controlled devices — Hardware clocks deliberately galat time set kiye ja sakte hain.

**Relying on Synchronized Clocks**  
Galat clocks easily unnoticed reh jaate hain — yeh dramatic crashes nahi karte, bas silent aur subtle data loss karte hain. Saari machines ke beech clock offsets monitor karo; nodes with excessive drift ko dead declare kar do.

**Timestamps for Ordering Events (Dangerous!)**  
<img width="2880" height="1325" alt="image" src="https://github.com/user-attachments/assets/c92ad878-c794-472e-a047-5f0c31a70748" />


**Figure 8-3: The write by client B is causally later than the write by client A, but B's write has an earlier timestamp**  
(Teesri image: Client A set x=1 karta hai timestamp 42.004 ke saath, Client B increment karta hai lekin timestamp 42.003 ke saath, galat order.)

- **Last Write Wins (LWW)** time-of-day clocks ke saath: <3 ms clock skew ke bawajood, writes galat order ho sakti hain. Lagging clock wala node silently writes drop kar deta hai.  
- LWW sequential writes ko truly concurrent writes se distinguish nahi kar sakta.  
- **Logical clocks** (incrementing counters) events order karne ke liye safer alternative hain.

**Clock Confidence Intervals**  
- Clock reading precise point in time nahi hai — iski ek uncertainty range hoti hai.  
- Zyadatar systems yeh uncertainty expose nahi karte.  
- Google ka **TrueTime API** (Spanner) explicitly `[earliest, latest]` report karta hai — actual time is interval mein kahin hai. Har datacenter mein GPS receivers aur atomic clocks use karta hai (~7 ms uncertainty).

**Synchronized Clocks for Global Snapshots (Spanner)**  
- Spanner TrueTime confidence intervals use karta hai snapshot isolation ke liye across datacenters.  
- Agar do intervals overlap nahi karte, to ordering definite hai. Agar overlap karte hain, to ordering uncertain hai.  
- Spanner deliberately confidence interval ki length ka wait karta hai commit karne se pehle, taaki causal ordering ensure ho.

**Process Pauses**  
Ek thread kisi bhi point par significant time ke liye paused ho sakta hai:  
- **Garbage collection (GC)** — "Stop-the-world" pauses seconds ya minutes tak ho sakte hain.  
- **VM suspension** — VM suspend aur resume kiya ja sakta hai (live migration). Pause arbitrary length ka ho sakta hai.  
- **Context switching** — OS ya hypervisor doosre thread/VM par switch karta hai. Steal time.  
- **Synchronous disk I/O** — Thread slow disk ka wait karte hue paused (khaas kar network filesystems jaise EBS).  
- **Swapping to disk (paging)** — Memory access page fault trigger karta hai → slow disk I/O.  
- **Unix signal handling** (e.g., SIGSTOP/SIGCONT).

Ek node ko assume karna chahiye ki uska execution significant length ke liye paused ho sakta hai. Pause ke dauran, baaki duniya chalti rehti hai aur paused node ko dead declare kar sakti hai.

**Response Time Guarantees**  
- **Hard real-time systems** (aircraft, rockets, robots) ke specified deadlines hote hain. Restricted programming languages, no dynamic memory allocation, enormous testing chahiye. Bahut expensive. Zyadatar server-side systems ke liye economical nahi.  
- **Limiting GC Impact** — GC pauses ko brief planned outages ki tarah treat karo: node ko requests bhejna band karo, outstanding requests finish hone do, phir GC karo jab koi requests progress mein na ho. Kuch financial trading systems yeh approach use karte hain.

---

Section 4: Knowledge, Truth, and Lies  
Network mein ek node kuch bhi sure nahi jaan sakta — wo sirf un messages ke based par guesses bana sakta hai jo use receive hote hain (ya nahi hote). Agar remote node respond nahi karta, to yeh jaanna impossible hai ki wo kis state mein hai, kyunki network ki problems reliably node ki problems se distinguish nahi ki ja sakti.

### 4.1 The Truth Is Defined by the Majority  
Ek node apne judgment par bharosa nahi kar sakta. Distributed system single node par rely nahi kar sakta. Iske bajay, bahut saare algorithms **quorum** par rely karte hain — nodes ke beech voting. Decisions ke liye minimum number of votes chahiye taaki kisi ek node par dependency kam ho.

Agar ek quorum of nodes kisi doosre node ko dead declare kare, to use dead maanna hoga, chahe wo node khud ko alive feel kare. Individual node ko quorum decision abide karna hoga aur step down hona hoga. Most commonly, quorum **absolute majority** hota hai (aadhe se zyada nodes).

### 4.2 The Leader and the Lock  
Situations jahan sirf ek node ko kuch karne allow hai:  
- Sirf ek node database partition ka leader ho (split brain avoid karne ke liye).  
- Sirf ek transaction ko particular resource ke liye lock hold karne allow hai (concurrent writes aur corruption prevent karne ke liye).  
- Sirf ek user ko particular username register karne allow hai.

Chahe ek node believe kare ki wo "the chosen one" hai, doosre nodes ne use dead declare kiya ho sakta hai aur replacement elect kiya ho sakta hai. Agar purana node apni self-appointed capacity mein kaam karta rahe, to data corruption cause kar sakta hai.

**Fencing Tokens**  
<img width="2880" height="1057" alt="image" src="https://github.com/user-attachments/assets/39f17b36-6144-4f5d-a7d9-be0ec7b97e51" />


**Figure 8-4: Incorrect implementation of a distributed lock: client 1 believes it still has a valid lease, even though it has expired, and thus corrupts a file in storage**  
(Chauthi image: Client 1 GC pause ke baad bhi lock held maanta hai, Client 2 lock le leta hai, Client 1 write karta hai aur data corrupt ho jaata hai.)

Ek request-handling loop consider karo jo lease use karta hai:  
```java
while (true) {
    request = getIncomingRequest();
    // Ensure that the lease always has at least 10 seconds remaining
    if (lease.expiryTimeMillis - System.currentTimeMillis() < 10000) {
        lease = lease.renew();
    }
    if (lease.isValid()) {
        process(request);
    }
}
```
Do problems: (1) yeh synchronized clocks par rely karta hai (expiry time different machine ne set kiya tha), aur (2) local monotonic clock ke saath bhi, code assume karta hai ki time check karne aur request process karne ke beech bahut kam time guzarta hai. `lease.isValid()` ke around 15-second GC pause ka matlab ho sakta hai ki lease request process hone se pehle expire ho jaaye — lekin thread ko agle loop iteration tak pata nahi chalega, tab tak wo kuch unsafe kar chuka ho sakta hai.
<img width="2880" height="1057" alt="image" src="https://github.com/user-attachments/assets/9aff25db-d00c-4a04-9fff-cf43a563a05a" />


**Figure 8-5: Making access to storage safe by allowing writes only in the order of increasing fencing tokens**  
(Paanchvi image: Lock server fencing token deta hai, storage sirf increasing tokens accept karta hai, purana token reject ho jaata hai.)

- Har baar lock server lock ya lease grant karta hai, wo ek **fencing token** return karta hai — ek number jo har baar lock grant hone par badhta hai.  
- Storage service ko har write request ke saath current fencing token include karna hota hai.  
- Storage server kisi bhi write ko reject kar deta hai agar uska token already processed token se kam ho.  
- Example: Client 1 ko token 33 milta hai, pause ho jaata hai (lease expire). Client 2 ko token 34 milta hai, successfully write karta hai. Client 1 jaagta hai, token 33 ke saath write bhejta hai → storage server reject kar deta hai.  
- ZooKeeper ka `zxid` ya `cversion` fencing tokens ki tarah use ho sakta hai (guaranteed monotonically increasing).  
- **Resource ko khud tokens check karne chahiye** — clients ke liye apna lock status check karna kaafi nahi hai.

### 4.3 Byzantine Faults  
Fencing tokens un nodes ko handle karte hain jo inadvertently galat kaam kar rahe hain. Lekin agar ek node deliberately system subvert karna chahe, to wo fake fencing token ke saath messages bhej sakta hai.

Is book mein hum assume karte hain ki nodes **unreliable but honest** hain: wo slow ya unresponsive ho sakte hain, lekin agar respond karein, to apni best knowledge ke hisaab se sach bolte hain.

**Byzantine fault** tab hota hai jab node arbitrary faulty ya corrupted responses bheje (e.g., claim kare ki message receive hua jabki nahi hua). **Byzantine Generals Problem**: consensus reach karna jab kuch participants traitor ho sakte hain jo contradictory messages bhejein.

**The Byzantine Generals Problem**  
- **Two Generals Problem** ka generalization hai, jo imagine karta hai ki do army generals ko battle plan par agree karna hai. Wo sirf messenger se communicate kar sakte hain, aur messengers kabhi delayed ya lost ho jaate hain (network packets ki tarah).  
- Byzantine version mein, *n* generals hain jinhe agree karna hai, lekin unmein se kuch traitor hain. Zyadatar generals loyal hain aur truthful messages bhejte hain, lekin traitors fake ya untrue messages bhejkar doosron ko deceive aur confuse karne ki koshish kar sakte hain (while trying to remain undiscovered). Pehle se pata nahi hota ki traitor kaun hai.  
- Naam Byzantium se aaya hai (ancient Greek city jo Constantinople bana, ab Istanbul). Iska koi historic evidence nahi ki Byzantine generals khaas taur par intrigue-prone the — naam "Byzantine" se derive karta hai jiska matlab excessively complicated, bureaucratic, devious hota hai. Lamport ne nationality choose ki readers ko offend na karne ke liye — unhe salah di gayi thi ki "The Albanian Generals Problem" kehna accha idea nahi hoga.

**When Byzantine Fault Tolerance Matters**  
- **Aerospace** — Radiation memory/CPU registers corrupt kar sakti hai → unpredictable responses. Flight control systems ko Byzantine faults tolerate karne hote hain.  
- **Multiple organizations** — Participants cheat karne ki koshish kar sakte hain (e.g., peer-to-peer networks jaise Bitcoin/blockchains).

**When It Doesn't**  
- Apne datacenter mein, saare nodes aapki organization control karti hai. Byzantine fault-tolerant protocols zyadatar server-side systems ke liye too complex aur expensive hain.  
- Web applications ko malicious client behavior expect karna chahiye, lekin use input validation, sanitization, aur output escaping se handle karna chahiye — Byzantine fault-tolerant protocols se nahi.

### 4.4 Weak Forms of Lying  
Bina full Byzantine fault tolerance ke bhi, weak forms of "lying" ke khilaf guard karna worth hai — invalid messages due to hardware issues, software bugs, aur misconfiguration. Yeh protection mechanisms full-blown Byzantine fault tolerance nahi hain (determined adversary ko withstand nahi karenge), lekin yeh simple aur pragmatic steps hain better reliability ki taraf:

- **Network packet corruption** — Usually TCP/UDP checksums se catch ho jaata hai, lekin kabhi kabhi detection evade kar jaata hai. Simple measures usually kaafi hain, jaise application-level checksums.  
- **Input sanitization** — Publicly accessible application ko users ke inputs carefully sanitize karne chahiye: values reasonable ranges mein hain check karo, string sizes limit karo taaki large memory allocations se denial of service na ho. Firewall ke peeche internal service kam strict checks se kaam chala sakta hai, lekin basic sanity-checking (e.g., protocol parsing mein) accha idea hai.  
- **NTP with multiple servers** — Client saare servers contact kare, unki errors estimate kare, aur check kare ki majority of servers kuch time range par agree karte hain. Jab tak zyadatar servers theek hain, ek misconfigured NTP server jo galat time report kare, outlier detect hoga aur synchronization se exclude ho jaayega. Multiple servers NTP ko zyada robust banate hain single server par rely karne se.

### 4.5 System Models  
Distributed algorithms ke baare mein reason karne ke liye, hum ek **system model** define karte hain — ek abstraction jo describe karta hai ki algorithm kaunse faults assume kar sakta hai.

**Timing Models**

| Model                   | Assumption                                                                                         |
|-------------------------|----------------------------------------------------------------------------------------------------|
| **Synchronous**         | Bounded network delay, bounded process pauses, bounded clock error. Zyadatar practical systems ke liye realistic nahi. |
| **Partially synchronous** | Zyadatar time synchronous jaisa behave karta hai, lekin kabhi kabhi bounds exceed kar deta hai. **Most realistic model.** |
| **Asynchronous**        | No timing assumptions at all. No clocks, no timeouts. Very restrictive.                             |

**Node Failure Models**

| Model               | Assumption                                                                                     |
|---------------------|------------------------------------------------------------------------------------------------|
| **Crash-stop**      | Node sirf crash karke fail ho sakta hai. Ek baar crash hua, forever gone.                      |
| **Crash-recovery**  | Nodes crash ho sakte hain aur baad mein restart. Stable storage (disk) crashes survive karta hai; in-memory state lost ho jaati hai. **Most useful model.** |
| **Byzantine**       | Nodes kuch bhi kar sakte hain, including sending contradictory messages.                       |

Real systems ke liye **most useful combination**: **partially synchronous model with crash-recovery faults**.

### 4.6 Correctness of Algorithms  
Properties define karo jo ek correct algorithm ko satisfy karni chahiye. Example for fencing tokens:  
- **Uniqueness** — Koi do requests same value return nahi karein.  
- **Monotonic sequence** — Agar request *x* complete hua *y* ke shuru hone se pehle, to `token_x < token_y`.  
- **Availability** — Jo node token request kare aur crash na ho, wo eventually response receive kare.

### 4.7 Safety and Liveness Properties  

|                       | Safety                                                        | Liveness                                                      |
|-----------------------|---------------------------------------------------------------|---------------------------------------------------------------|
| **Informal definition** | Kuch bura nahi hota                                           | Kuch accha eventually hota hai                                 |
| **If violated**       | Specific point in time point out kar sakte ho jab yeh toota. Violation undo nahi ho sakti. | Abhi hold nahi kar raha ho sakta hai, lekin future mein satisfy hone ki umeed hamesha hai. |
| **Examples**          | Uniqueness, monotonic sequence                                | Availability, eventual consistency                             |
| **Requirement**       | Hamesha hold karna chahiye, chahe saare nodes crash ho jaayein ya poori network fail ho jaaye | Caveats allow kiye ja sakte hain (e.g., sirf agar majority of nodes working ho) |

Safety aur liveness distinguish karna difficult system models deal karne mein help karta hai: safety properties saari situations mein hold karni chahiye; liveness properties conditions ke saath ho sakti hain.

### 4.8 Mapping System Models to the Real World  
- Crash-recovery model mein algorithms assume karte hain ki stable storage crashes survive karta hai. Lekin agar disk corrupt ho jaaye to?  
- Theoretical models simplified abstractions hain. Real implementation ko "impossible" situations handle karni pad sakti hain (chahe handling sirf `printf("Sucks to be you")` aur `exit(666)` hi kyun na ho).  
- Algorithm ko system model mein correct prove karna guarantee nahi karta ki implementation hamesha correctly behave karega — lekin yeh bahut accha first step hai.  
- **Theoretical analysis aur empirical testing equally important hain.**

---

Summary  
**Key Takeaways**  

**Problems in Distributed Systems:**  
- **Unreliable Networks** — Packets lost, delayed, duplicated, ya reordered ho sakte hain. Reply bhi lost ya delayed ho sakta hai. Agar reply nahi aata, to aap nahi jaante kyun. Ek hi option hai: timeout.  
- **Unreliable Clocks** — Clocks significantly out of sync ho sakte hain (NTP ke bawajood). Forward ya backward jump kar sakte hain. Events order karne ke liye in par rely mat karo. Logical clocks use karo.  
- **Process Pauses** — Process substantial time ke liye pause ho sakta hai (GC, VM suspension, context switching, disk I/O, swapping) aur doosre nodes use dead declare kar sakte hain, phir wo resume ho sakta hai bina jaane ki wo paused tha.

**Consequences:**  
- Ek node kuch bhi sure nahi jaan sakta — sirf received messages ke based par guesses bana sakta hai.  
- Faults detect karna mushkil hai — zyadatar algorithms timeouts par rely karte hain, jo network aur node failures distinguish nahi kar sakte.  
- No global variable, no shared memory, no common knowledge. Information flow ka ek hi tareeka hai: unreliable network.  
- Major decisions single node nahi le sakta — quorum protocols chahiye.

**Key Concepts:**  
- **Fencing tokens** — Monotonically increasing numbers jo stale lock holders ko data corrupt karne se rokta hai.  
- **Byzantine faults** — Nodes jo jhooth bolte hain. Aerospace aur blockchains ke liye relevant, lekin zyadatar server-side systems ke liye too expensive.  
- **System models** — Partially synchronous + crash-recovery sabse realistic hai.  
- **Safety properties** hamesha hold karni chahiye; **liveness properties** conditions ke saath ho sakti hain.

Agar aap cheezein single machine par rakh sakte ho, to generally worth hai. Lekin scalability, fault tolerance, aur low latency ke liye distributed systems zaroori hain — aur in problems ka saamna karna hi padta hai.
