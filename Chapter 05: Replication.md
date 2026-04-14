Replication ka matlab hai ek hi data ki copy ko network ke through multiple machines par rakhna. Data replicate karne ke reasons:

- Data ko geographically users ke paas rakhna (latency kam karna)
- Agar kuch parts fail ho jaayein to system ko chalte rehna dena (availability badhana)
- Read queries serve karne wali machines ki sankhya badhana (read throughput badhana)

Replication ki saari mushkil is baat mein hai ki replicated data mein changes ko kaise handle kiya jaaye. Teen popular algorithms:

- Single-leader replication
- Multi-leader replication
- Leaderless replication

---

Section 1: Leaders and Followers  
Har node jo database ki copy store karta hai use **replica** kehte hain. Sabse common solution hai **leader-based replication** (active/passive ya master–slave replication bhi kehlata hai):

- Ek replica ko **leader** (master/primary) designate kiya jaata hai. Saare writes leader ke paas bheje jaate hain.
- Baaki replicas **followers** (read replicas/slaves/secondaries/hot standbys) hote hain. Leader data changes ko ek replication log ya change stream ke roop mein sabhi followers ko bhejta hai. Har follower writes ko same order mein apply karta hai.
- Reads leader ya kisi bhi follower ke paas ja sakti hain. Writes sirf leader par accept hoti hain.
<img width="2880" height="925" alt="image" src="https://github.com/user-attachments/assets/046948a2-803c-4367-8c46-103115f8162b" />

**Figure 5-1: Leader-based (master–slave) replication**  
(Aapki pehli image: User 1234 leader par picture update karta hai, leader replication streams ke through followers ko bhejta hai, User 2345 follower se read karta hai.)

Yeh approach use hoti hai: PostgreSQL (9.0+), MySQL, Oracle Data Guard, SQL Server AlwaysOn, MongoDB, RethinkDB, Espresso, Kafka, RabbitMQ highly available queues, DRBD mein.
<img width="2880" height="1128" alt="image" src="https://github.com/user-attachments/assets/4f70e967-ada1-475d-9a0a-1b105b18bcc3" />

### 1.1 Synchronous Versus Asynchronous Replication  
**Figure 5-2: Leader-based replication with one synchronous and one asynchronous follower**  
(Doosri image: Leader do followers ko data change bhejta hai, ek synchronous jo turant ok karta hai, doosra asynchronous.)

- **Synchronous** – Leader tab tak wait karta hai jab tak follower confirm na kar de ki write receive ho gayi, tabhi user ko success report karta hai.
  - Fayda: Follower guaranteed up-to-date copy rakhta hai jo leader ke consistent hai.
  - Nuksan: Agar synchronous follower respond nahi karta, to write process nahi ho sakta. Leader ko saare writes block karne padte hain.
- **Asynchronous** – Leader message bhejta hai lekin response ka wait nahi karta.

**Semi-Synchronous**  
Saare followers ka synchronous hona impractical hai (ek bhi node outage poora system rok deti hai). Practice mein: ek follower synchronous hota hai, baaki asynchronous. Agar synchronous follower unavailable ho jaaye, to kisi asynchronous follower ko synchronous bana diya jaata hai. Isse kam se kam do nodes (leader + ek synchronous follower) par up-to-date copy guarantee hoti hai.

**Fully Asynchronous**  
Agar leader fail ho jaaye aur recover na ho, to jo writes abhi followers tak replicate nahi hui, wo lost ho jaati hain (client ko confirm karne ke bawajood write durable nahi hai).  
Fayda: Leader writes process karta reh sakta hai chahe saare followers piche chhoot gaye hon.  
Asynchronous replication widely used hai, khaas kar jab bahut saare followers ho ya geographically distributed replicas ho.

**Research on Replication**  
Asynchronously replicated systems mein leader fail hone par data lose hona serious problem hai, isliye researchers replication methods investigate karte rahe hain jo data lose na karein lekin phir bhi acchi performance aur availability dein.  
**Chain replication** synchronous replication ka ek variant hai jise Microsoft Azure Storage jaise systems mein successfully implement kiya gaya hai. Ismein writes nodes ki chain ke through sequence mein propagate hoti hain (head → intermediate → tail), aur reads tail se serve hoti hain. Yeh strong consistency provide karta hai saath hi read load distribute karta hai.  
Replication ki consistency aur **consensus** (kai nodes ko ek value par agree karwana) ke beech strong connection hai, jise Chapter 9 mein detail mein explore kiya gaya hai.

### 1.2 Setting Up New Followers  
Naye follower ke paas bina downtime ke accurate copy kaise ensure karein:

1. Leader ki database ka ek consistent snapshot lo kisi point in time par (poori database lock kiye bina). MySQL ke liye `innobackupex` jaise third-party tools.
2. Snapshot ko naye follower node par copy karo.
3. Follower leader se connect karta hai aur snapshot ke baad ke saare data changes maangta hai. Snapshot ko leader ke replication log mein exact position se associated hona chahiye (PostgreSQL: log sequence number; MySQL: binlog coordinates).
4. Jab follower backlog process kar leta hai, to wo catch up kar leta hai aur aage changes aate hi process karta rehta hai.

### 1.3 Handling Node Outages  
**Follower Failure: Catch-up Recovery**  
- Har follower leader se receive hue data changes ka log rakhta hai.
- Crash/restart ya network interruption par, follower leader se connect karta hai aur wo saare changes maangta hai jo uske disconnected rahne ke dauran hue the.
- Apply karne ke baad, wo catch up kar leta hai aur stream receive karna continue karta hai.

**Leader Failure: Failover**  
Kisi ek follower ko naya leader promote karna padta hai. Is process ko **failover** kehte hain.

Automatic failover steps:

1. **Determining the leader has failed** – Zyadatar systems timeout use karte hain (e.g., 30 seconds bina response ke → assumed dead).
2. **Choosing a new leader** – Baaki replicas ki majority se election, ya pehle se elected controller node dwara appointment. Best candidate: replica jiske paas sabse up-to-date data ho (data loss minimize karne ke liye). Yeh ek **consensus problem** hai.
3. **Reconfiguring the system** – Clients naye leader ko writes bhejne lagte hain. Agar purana leader wapas aa jaaye, to use follower ban-na padta hai aur naye leader ko pehchan-na padta hai.

**Things That Can Go Wrong with Failover**  
- **Unreplicated writes lost** – Agar async replication, to naye leader ke paas purane leader ke saare writes nahi ho sakte. Sabse common solution: purane leader ke unreplicated writes discard kar diye jaate hain (durability expectations violate ho sakti hain).
- **Dangerous with external coordination** – GitHub incident: out-of-date MySQL follower ko leader promote kiya gaya, autoincrementing primary keys reuse hui jo Redis mein bhi use hoti thi → private data galat users ko dikha diya gaya.
- **Split brain** – Do nodes dono khud ko leader maan lete hain. Agar dono bina conflict resolution ke writes accept karein → data loss ya corruption. Safety catch: **STONITH** (Shoot The Other Node In The Head) — agar do leaders detect ho to ek node ko shutdown kar do. Agar design kharab ho to dono nodes shutdown ho sakti hain.
- **Timeout too short** – Temporary load spikes ya network glitches ke dauran unnecessary failovers, situation aur kharab kar dete hain.

In reasons ki wajah se, kuch operations teams automatic failover support hone ke bawajood manual failover prefer karte hain.

### 1.4 Implementation of Replication Logs  

**Statement-Based Replication**  
Leader har write statement (`INSERT`, `UPDATE`, `DELETE`) log karta hai aur followers ko bhejta hai.  
Problems: Nondeterministic functions (`NOW()`, `RAND()`) har replica par alag values generate karte hain; statements jo existing data par depend karte hain unhe exactly same order mein execute hona chahiye; side effects (triggers, stored procedures) alag ho sakte hain.  
MySQL mein version 5.1 se pehle use hota tha. VoltDB ise safely use karta hai deterministic transactions require karke.

**Write-Ahead Log (WAL) Shipping**  
Leader apna WAL (wohi log jo crash recovery ke liye use hota hai) network par followers ko bhejta hai.  
PostgreSQL aur Oracle mein use hota hai.  
Nuksan: WAL data ko bahut low level par describe karta hai (kaunse bytes kis disk block mein change hue) → storage engine se closely coupled. Leader aur followers par different software versions nahi chala sakte → zero-downtime upgrades mushkil hain.

**Logical (Row-Based) Log Replication**  
Storage engine se alag log format use karta hai (**logical log** vs physical representation).  
Records ka sequence jo row ke granularity par writes describe karta hai:
- Inserted row: saare columns ke naye values.
- Deleted row: row ko uniquely identify karne ke liye kaafi info (primary key, ya saare column values).
- Updated row: row identify karne ke liye info + changed columns ke naye values.

Storage engine se decoupled → backward compatible, different software versions ya even different storage engines leader aur followers par allow karta hai.  
External applications ke liye parse karna aasan → **change data capture** enable karta hai (database contents ko data warehouses, custom indexes, caches mein bhejna).  
MySQL ka binlog (row-based replication) yeh approach use karta hai.

**Trigger-Based Replication**  
Database triggers aur stored procedures use karta hai changes ko alag table mein log karne ke liye, jise external process padhta hai aur doosre system mein replicate karta hai.  
Examples: Oracle GoldenGate, Databus for Oracle, Bucardo for Postgres.  
Zyada overhead aur bugs ke liye prone, lekin flexibility ke liye useful (data ka subset replicate karna, different database types ke beech replicate karna, custom conflict resolution).

---

Section 2: Problems with Replication Lag  
Leader-based replication mein saare writes ek hi node se guzarte hain, lekin read-only queries kisi bhi replica ke paas ja sakti hain. Read-heavy workloads (web par common) ke liye aap bahut saare followers bana sakte ho aur reads unpar distribute kar sakte ho — ek **read-scaling architecture**.

Yeh realistically sirf asynchronous replication ke saath kaam karta hai (sabhi followers ko synchronous replication ka matlab kisi ek node ka fail hona poore system ko writing ke liye unavailable kar deta).

**Eventual Consistency**  
Agar application asynchronous follower se padhta hai, to use outdated information dikh sakti hai. Agar aap same query ek hi time par leader aur follower par run karein, to alag results mil sakte hain. Yeh inconsistency temporary hai — agar aap writes band kar dein aur wait karein, to followers eventually catch up kar lenge. Ise **eventual consistency** kehte hain.

**Replication lag** (leader par write aur follower par reflect hone ke beech ka delay) normal operation mein fraction of a second ho sakta hai, lekin load ya network problems mein kai seconds ya minutes tak badh sakta hai.

Teen examples of problems caused by replication lag:

### 2.1 Reading Your Own Writes  
User data submit karta hai (leader ke paas bheja), phir use view karta hai (follower se padha). Agar follower ne abhi tak catch up nahi kiya, to user ko apna hi data kho gaya lagta hai.
<img width="2880" height="1128" alt="image" src="https://github.com/user-attachments/assets/b8dab9d3-97d6-4212-bb84-0593c298c714" />

**Figure 5-3: A user makes a write, followed by a read from a stale replica**  
(Teesri image: User 1234 comment insert karta hai leader par, phir User 2345 follower se read karta hai, lekin follower ke paas abhi comment nahi hai.)

Humein **read-after-write consistency** (ya **read-your-writes consistency**) chahiye: agar user page reload kare, to wo hamesha apne dwara submit kiye gaye updates dekhega. Doosre users ke updates ke baare mein koi promise nahi.

**Techniques to Implement Read-After-Write Consistency**  
- Un cheezon ke liye leader se padho jinhe user modify kar sakta hai. Example: social network par, user ka apna profile hamesha leader se padho, doosre users ke profiles followers se.
- Last update ka time track karo — last update ke ek minute baad tak, saare reads leader se karo. Ya replication lag monitor karo aur un followers par queries prevent karo jo ek minute se zyada peeche ho.
- Client most recent write ka timestamp yaad rakhe — system ensure kare ki read serve karne wala replica kam se kam us timestamp tak ke updates reflect kare. Timestamp logical ho sakta hai (log sequence number) ya actual system clock (clock synchronization require karta hai).
- Multi-datacenter complexity — Koi bhi request jise leader ki zaroorat hai, use us datacenter mein route karna hoga jismein leader hai.

**Cross-Device Read-After-Write Consistency**  
Agar user ek device par information enter kare aur doosre par dekhe:
- Timestamp metadata centralized hona chahiye (ek device ka code doosre device ke updates ke baare mein nahi jaanta).
- Agar replicas different datacenters mein hain, to koi guarantee nahi ki alag devices ke connections same datacenter mein route ho — shayad user ke saare devices ko same datacenter mein route karna pade.

### 2.2 Monotonic Reads  
User alag-alag replicas se kai reads karta hai. Ho sakta hai pehle fresh replica se padhe, phir stale replica se — aisa lagega jaise time ulta chal gaya (e.g., ek comment aata hai, phir refresh par gayab ho jaata hai).
<img width="2880" height="1474" alt="image" src="https://github.com/user-attachments/assets/e46a4f30-a02d-4e08-b7d5-5a684b1443f2" />

**Figure 5-4: A user first reads from a fresh replica, then from a stale replica. Time appears to go backward**  
(Chauthi image: User 2345 do baar read karta hai, pehli baar comment dikhta hai, doosri baar nahi.)

**Monotonic reads** guarantee karta hai ki agar ek user sequence mein kai reads kare, to wo time ko ulta nahi dekhega — naye data ke baad purana data nahi padhega. Yeh strong consistency se kam guarantee hai, lekin eventual consistency se strong hai.

Implementation: Ensure karo ki har user hamesha same replica se padhe (e.g., user ID ke hash ke based par choose karo, randomly nahi). Agar wo replica fail ho jaaye, to doosre par reroute karo.

### 2.3 Consistent Prefix Reads  
Causality ka violation: observer answer ko question se pehle dekh leta hai.

Example: Mr. Poons question poochta hai, Mrs. Cake answer deti hai. Lekin agar Mrs. Cake ka message kam lag wale follower se jaaye aur Mr. Poons ka message zyada lag wale follower se, to teesra aadmi answer ko question se pehle dekh sakta hai.
<img width="2880" height="1843" alt="image" src="https://github.com/user-attachments/assets/ef878411-3cb4-4aec-9809-51f4dd974fd7" />

**Figure 5-5: If some partitions are replicated slower than others, an observer may see the answer before they see the question**  
(Paanchvi image: Mr. Poons ka question Partition 1 mein, Mrs. Cake ka answer Partition 2 mein, observer answer pehle padh leta hai.)

**Consistent prefix reads** guarantee karta hai: agar writes ka sequence ek particular order mein ho, to unhe padhne wala hamesha unhe usi order mein dekhega.

Yeh khaas kar partitioned (sharded) databases mein problem hai jahan alag partitions independently operate karte hain aur writes ka koi global ordering nahi hota.  
Solution: Causally related writes ko same partition mein likho. Ya algorithms use karo jo explicitly causal dependencies track karein.

### 2.4 Solutions for Replication Lag  
Agar application replication lag ko minutes ya hours tak badhne par "no problem" tolerate kar sakta hai, to bahut accha.  
Agar isse user experience kharab hota hai, to system ko stronger guarantees provide karne ke liye design karo (read-after-write, monotonic reads, etc.).  
In issues ko application code mein deal karna complex hai aur galti karna aasan hai.  
**Transactions** isliye exist karte hain taaki stronger guarantees provide karein aur application simple ho sake. Lekin bahut saari distributed databases ne transactions abandon kar diye hain, yeh kehkar ki wo bahut expensive hain — yeh overly simplistic hai.

---

Section 3: Multi-Leader Replication  
Leader-based replication ka ek bada nuksan: sirf ek leader hai, aur saare writes use se guzarne padte hain. Agar aap leader se connect nahi kar sakte, to write nahi kar sakte.

**Multi-leader configuration** (master–master / active/active): ek se zyada nodes ko writes accept karne deta hai. Har leader simultaneously doosre leaders ke liye follower ka kaam karta hai.

### 3.1 Use Cases  

**Multi-Datacenter Operation**  
Har datacenter mein ek leader ke saath, regular leader–follower replication har datacenter ke andar use hoti hai; datacenters ke beech, har leader doosre datacenters ke leaders ko changes replicate karta hai.
<img width="2880" height="1401" alt="image" src="https://github.com/user-attachments/assets/a36aef98-09a1-48f9-b35e-27a5751beb32" />

**Figure 5-6: Multi-leader replication across multiple datacenters**  
(Chhathi image: Do datacenters, dono mein leader, dono read-write queries accept karte hain, beech mein changes replicate hote hain, conflict resolution hota hai.)

| Single-Leader | Multi-Leader |
|---------------|--------------|
| **Performance** | Har write internet par leader ke datacenter jaata hai (significant latency) | Har write local datacenter mein process hota hai, asynchronously replicate hota hai (inter-datacenter delay users se hidden) |
| **Datacenter outage tolerance** | Doosre datacenter mein follower promote karne ke liye failover chahiye | Har datacenter independently operate karta hai; failed datacenter wapas aane par replication catch up karta hai |
| **Network problem tolerance** | Inter-datacenter link problems ke liye bahut sensitive (writes is link par synchronous hain) | Network problems better tolerate karta hai (async replication) |

Tools: Tungsten Replicator (MySQL), BDR (PostgreSQL), GoldenGate (Oracle).

Bada nuksan: Same data concurrently do different datacenters mein modify ho sakta hai → **write conflicts** resolve karne padte hain. Multi-leader replication aksar dangerous territory maana jaata hai jise avoid karna chahiye agar possible ho.

**Clients with Offline Operation**  
Applications jinhe disconnected rahkar kaam karna hai (e.g., mobile/laptop par calendar apps). Har device ke paas local database hota hai jo leader ka kaam karta hai; async multi-leader replication (sync) tab hoti hai jab online ho. Har device essentially ek "datacenter" hai jiska network connection extremely unreliable hai. CouchDB is mode ke liye design kiya gaya hai.

**Collaborative Editing**  
Real-time collaborative editing (Etherpad, Google Docs) essentially multi-leader replication hai. Fast collaboration ke liye, change ki unit bahut chhoti banao (e.g., single keystroke) aur locking avoid karo — lekin yeh multi-leader replication ke saare challenges laata hai including conflict resolution.

### 3.2 Handling Write Conflicts  
<img width="2880" height="1488" alt="image" src="https://github.com/user-attachments/assets/15fdf00a-da74-4441-b4ac-792a83d3733d" />

**Figure 5-7: A write conflict caused by two leaders concurrently updating the same record**  
(Saatvi image: User 1 Leader 1 par value = 1 set karta hai, User 2 Leader 2 par value = 2 set karta hai, conflict.)

**Conflict Avoidance**  
Sabse simple strategy: ensure karo ki particular record ke liye saare writes same leader ke through jaayein. Example: particular user ke requests ko same datacenter mein route karo. Yeh tab toot jaata hai jab designated leader change karna pade (datacenter failure, user relocation).

**Converging Toward a Consistent State**  
Multi-leader mein, writes ka koi defined ordering nahi hai. Database ko conflicts ko **convergent** tareeke se resolve karna hoga (saare replicas same final value par pahunchein):

- **Last Write Wins (LWW)** – Har write ko unique ID do (e.g., timestamp), highest ID ko winner pick karo, baaki discard. Popular lekin dangerously prone to data loss.
- **Replica priority** – Higher-numbered replica hamesha precedence leta hai. Iska bhi matlab data loss.
- **Merge values** – E.g., alphabetically order karo aur concatenate karo ("B/C").
- **Record the conflict** – Saari information ko explicit data structure mein preserve karo, baad mein resolve karo (shayad user se pooch kar).

**Custom Conflict Resolution Logic**  
- **On write** – Database conflict handler call karta hai jaise hi conflict detect ho (e.g., Bucardo Perl snippets allow karta hai). Background mein run hota hai, jaldi execute hona chahiye.
- **On read** – Saare conflicting writes store kiye jaate hain. Jab next time data padha jaaye, multiple versions application ko return kiye jaate hain, jo conflict resolve karta hai. CouchDB aise kaam karta hai.

Conflict resolution usually individual row ya document ke level par apply hota hai, poore transaction par nahi.

**Automatic Conflict Resolution**  
- **CRDTs (Conflict-free Replicated Datatypes)** – Data structures (sets, maps, ordered lists, counters) jo concurrently edit kiye ja sakte hain aur automatically conflicts resolve karte hain. Riak 2.0 mein implemented.
- **Mergeable persistent data structures** – History explicitly track karte hain (Git ki tarah), three-way merge function use karte hain.
- **Operational transformation** – Algorithm behind Etherpad aur Google Docs, ordered lists (e.g., text document mein characters) ke concurrent editing ke liye design kiya gaya.

### 3.3 Multi-Leader Replication Topologies  
<img width="2880" height="800" alt="image" src="https://github.com/user-attachments/assets/1c1e20b3-64e0-4982-9598-e233f0b77c9d" />

**Figure 5-8: Three example topologies in which multi-leader replication can be set up**  
( image: Circular, Star, All-to-all topologies dikhayi gayi hain.)

- **Circular topology** – Har node ek node se writes receive karta hai aur doosre ko forward karta hai. MySQL ka default.
- **Star topology** – Ek designated root node baaki sabko writes forward karta hai. Tree mein generalize kiya ja sakta hai.
- **All-to-all** – Har leader har doosre leader ko writes bhejta hai. Sabse general.

Circular aur star topologies mein, writes kai nodes se guzarti hain. Har write ko node identifiers se tag kiya jaata hai taaki infinite replication loops na ho. Problem: agar ek node fail ho jaaye, to flow interrupt ho jaata hai (manual reconfiguration chahiye).

All-to-all ki fault tolerance better hai (messages alag paths se travel karte hain), lekin ismein problem ho sakti hai ki writes galat order mein arrive karein different network link speeds ki wajah se:
<img width="2880" height="1602" alt="image" src="https://github.com/user-attachments/assets/43d5302a-cbb4-4a51-ba1e-c27921025783" />

**Figure 5-9: With multi-leader replication, writes may arrive in the wrong order at some replicas**  
( image: Client A Leader 1 par insert karta hai, Client B Leader 3 par update karta hai, lekin update insert se pehle Leader 2 par pahunch jaata hai.)

Yeh causality problem hai: update prior insert par depend karta hai, lekin replica update ko insert se pehle receive kar sakta hai. Timestamps kaafi nahi hain (clocks par bharosa nahi kar sakte). **Version vectors** use kiye ja sakte hain events ko sahi order mein karne ke liye, lekin conflict detection bahut saare multi-leader systems mein poorly implemented hai.

---

Section 4: Leaderless Replication  
Kuch systems leader ka concept hi chhod dete hain, kisi bhi replica ko directly clients se writes accept karne dete hain. Amazon ke in-house **Dynamo** system ne popularize kiya. Open source Dynamo-style datastores: Riak, Cassandra, Voldemort.

### 4.1 Writing When a Node Is Down  
Leaderless configuration mein, client write ko saare replicas ko parallel mein bhejta hai. Agar kuch unavailable hain, to write succeed ho jaata hai agar kaafi replicas acknowledge kar dein.
<img width="2880" height="1572" alt="image" src="https://github.com/user-attachments/assets/5feb7e6c-f4b4-47d7-a9c9-ad552aa63582" />

**Figure 5-10: A quorum write, quorum read, and read repair after a node outage**  
( image: Write 3 replicas mein se 2 par successful, read 3 mein se 2 par, stale replica ko read repair se update kiya jaata hai.)

Padhte waqt, client kai nodes ko parallel mein requests bhejta hai aur version numbers use karta hai determine karne ke liye ki kaunsi value newer hai.

**Read Repair**  
Jab client kai nodes se padhta hai aur stale response detect karta hai, to wo newer value wapas stale replica par likh deta hai. Frequently read values ke liye accha kaam karta hai.

**Anti-Entropy Process**  
Background process jo constantly replicas ke beech differences dhundhta hai aur missing data copy karta hai. Writes ko kisi particular order mein copy nahi karta; significant delay ho sakta hai.

### 4.2 Quorums for Reading and Writing  
Agar *n* replicas hain, to har write ko *w* nodes confirm karna chahiye, aur har read ke liye kam se kam *r* nodes query karne padenge. Jab tak **w + r > n**, hum expect karte hain ki padhte waqt up-to-date value milegi (kam se kam ek *r* node ke paas latest value hogi).
<img width="1012" height="452" alt="image" src="https://github.com/user-attachments/assets/4ca975fd-6e04-43ec-8262-23b58b7eaac4" />

**Figure 5-11: If w + r > n, at least one of the r replicas you read from must have seen the most recent successful write**  
( image: 5 replicas, w=3, r=3, overlap dikhaya gaya hai.)

Common choice: *n* odd (typically 3 ya 5), *w = r = (n + 1) / 2*.
- *w = n, r = 1* → fast reads ke liye optimized.
- *w = 1, r = n* → fast writes ke liye optimized.
- *n = 3, w = 2, r = 2* → ek unavailable node tolerate karta hai.
- *n = 5, w = 3, r = 3* → do unavailable nodes tolerate karta hai.

Reads aur writes hamesha saare *n* replicas ko parallel mein bheje jaate hain; *w* aur *r* determine karte hain ki kitne ke liye wait karna hai.

### 4.3 Limitations of Quorum Consistency  
*w + r > n* ke bawajood bhi stale values return ho sakti hain:

- **Sloppy quorums** – *w* writes alag nodes par end up ho sakte hain jahan *r* reads karein.
- **Concurrent writes** – Clear ordering nahi; agar LWW use ho raha hai to clock skew ki wajah se writes lost ho sakti hain.
- **Concurrent write and read** – Write sirf kuch replicas par reflect ho sakti hai.
- **Write succeeded on fewer than w replicas** – Jahan succeed hua wahan rollback nahi hota.
- **Node restored from stale replica** – Naye value wale replicas ki sankhya *w* se kam ho sakti hai.

Dynamo-style databases un use cases ke liye optimized hain jo eventual consistency tolerate kar sakte hain. Parameters *w* aur *r* stale reads ki probability adjust karte hain lekin absolute guarantees nahi hain.

### 4.4 Sloppy Quorums and Hinted Handoff  
Network interruption ke dauran, client shayad value ke *n* "home" nodes tak nahi pahunch paaye, lekin cluster ke doosre nodes tak pahunch sakta hai.

- **Sloppy quorum** – Writes un reachable nodes par accept kar lo jo designated *n* "home" nodes mein nahi hain. (Jaise ghar mein taala lagne par padosi ke yahan rehna.)
- **Hinted handoff** – Jab network interruption fix ho jaaye, to temporarily doosre nodes dwara accept ki gayi writes appropriate "home" nodes ko bhej di jaati hain. (Padosi aapse ghar jaane ko kehta hai jab aapko chaabi mil jaaye.)

Sloppy quorums write availability badhate hain lekin iska matlab hai ki *w + r > n* hone par bhi aap latest value padhne ke baare mein sure nahi ho sakte (wo *n* ke bahar wale node par ho sakti hai). Yeh sirf durability ka assurance hai, consistency ka nahi.

- Riak: sloppy quorums default mein enabled.
- Cassandra aur Voldemort: default mein disabled.

### 4.5 Detecting Concurrent Writes  
Dynamo-style databases kai clients ko same key par concurrently write karne dete hain → strict quorums ke saath bhi conflicts arise hote hain.
<img width="2880" height="1406" alt="image" src="https://github.com/user-attachments/assets/1b394d8b-d3d3-4b1f-866d-7f3fae0ac2d5" />

**Figure 5-12: Concurrent writes in a Dynamo-style datastore: there is no well-defined ordering**  
( image: Client A set X = A, Client B set X = B, Node 3 dono ko alag order mein dekhta hai.)

**Last Write Wins (LWW)**  
Har write ke saath timestamp attach karo, sabse bade ko winner pick karo, baaki discard karo.  
Cassandra mein sirf yahi supported conflict resolution hai; Riak mein optional.  
Eventual convergence achieve karta hai lekin durability ke cost par — concurrent writes mein se sirf ek bachti hai, baaki silently discard ho jaati hain.  
Sirf tab safe hai jab key ek baar likhi jaaye aur uske baad immutable treat ki jaaye (e.g., UUID as key).

**The "Happens-Before" Relationship and Concurrency**  
- Operation A **happens before** B agar B A ke baare mein jaanta hai, A par depend karta hai, ya A par build karta hai.
- Do operations **concurrent** hain agar na to ek doosre se pehle hua (na hi ek doosre ke baare mein jaanta hai).
- Teen possibilities: A happened before B, B happened before A, ya A aur B concurrent hain.

**Capturing Causal Dependencies (Version Numbers)**  
<img width="2880" height="1633" alt="image" src="https://github.com/user-attachments/assets/c5fac883-ca40-4932-98ab-545b45b59d38" />

**Figure 5-13: Capturing causal dependencies between two clients concurrently editing a shopping cart**  
( image: Shopping cart mein milk, flour, eggs, bacon ka sequence version numbers ke saath dikhaya gaya hai.)
<img width="2880" height="718" alt="image" src="https://github.com/user-attachments/assets/34bcc7c3-2a9a-4ee9-948b-6e1c06ec2a5f" />

**Figure 5-14: Graph of causal dependencies**  
( image: Shopping cart items ke beech causal dependencies ka graph.)

Algorithm:
- Server har key ke liye version number maintain karta hai, har write par increment hota hai.
- Jab client read karta hai, server saari values return karta hai jo abhi tak overwrite nahi hui + latest version number. Client ko write se pehle read karna zaroori hai.
- Jab client write karta hai, to wo prior read ka version number include karta hai aur saari received values merge karna padta hai.
- Server us version number ya usse kam wali saari values overwrite kar deta hai, lekin higher version number wali values rakhta hai (wo concurrent hain).

**Merging Concurrently Written Values (Siblings)**  
Riak concurrent values ko **siblings** kehta hai.  
Simple approach: sibling values ka union le lo (e.g., shopping carts merge karo).  
Problem with removals: hataya hua item union mein wapas aa jaata hai. Solution: **tombstone** (deletion marker with version number) use karo instead of actually deleting.  
**CRDTs** (e.g., Riak ka datatype support) automatically siblings merge kar sakte hain, including preserving deletions.

**Version Vectors**  
Multiple replicas (no leader) ke saath, har replica ke liye bhi version number use karo saath hi key ke liye.  
Har replica apna version number increment karta hai writes par aur doosre replicas ke version numbers track karta hai.  
Saare replicas ke version numbers ke collection ko **version vector** kehte hain.  
**Dotted version vectors** (Riak 2.0 mein use) ek refined variant hai.  
Version vectors database se clients ko reads par bheje jaate hain, aur writes par wapas database ko (Riak ise **causal context** kehta hai).  
Note: version vectors ≠ vector clocks (subtle difference — version vectors replica state compare karne ke liye sahi structure hai).

---

Summary  
**Key Takeaways**  

**Purposes of Replication:**  
- **High availability** – System chalte rehta hai jab machines/datacenters down ho jaayein.
- **Disconnected operation** – Application network interruptions ke dauran kaam karta hai.
- **Latency** – Data geographically users ke paas.
- **Scalability** – Replicas ke through higher read volume handle karna.

**Three Approaches:**  
- **Single-leader** – Saare writes ek node par. Reads kisi bhi replica se (stale ho sakti hai). Samajhna aasan, no conflict resolution.
- **Multi-leader** – Writes kisi bhi leader par. Faults/latency ke liye zyada robust, lekin reason karna mushkil, conflict resolution chahiye.
- **Leaderless** – Writes aur reads kai nodes ko parallel mein. Quorum-based. Sabse robust lekin weakest consistency guarantees.

**Consistency Models for Replication Lag:**  
- **Read-after-write consistency** – Users hamesha apna submit kiya hua data dekhein.
- **Monotonic reads** – Users time ko ulta nahi dekhte.
- **Consistent prefix reads** – Users data ko causal order mein dekhte hain.

**Concurrency and Conflict Resolution:**  
- Multi-leader aur leaderless concurrent writes allow karte hain → conflicts.
- **LWW** simple hai lekin data lose karta hai.
- **Version vectors** aur **happens-before relationship** concurrency determine karte hain.
- **Siblings merge karna** (union, CRDTs, tombstones) data preserve karta hai.
