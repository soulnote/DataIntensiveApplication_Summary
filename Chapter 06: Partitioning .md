Bahut bade datasets ya bahut high query throughput ke liye, sirf replication kaafi nahi hoti — humein data ko **partitions** (ya **sharding**) mein todna padta hai.

**Terminology Across Databases**  

| Term      | Used By                                 |
|-----------|-----------------------------------------|
| Partition | General / established term              |
| Shard     | MongoDB, Elasticsearch, SolrCloud       |
| Region    | HBase                                   |
| Tablet    | Bigtable                                |
| vnode     | Cassandra, Riak                         |
| vBucket   | Couchbase                               |

Har piece of data exactly ek partition mein belong karta hai. Effectively, har partition apne aap mein ek chhota database hai. Partitioning ka main reason **scalability** hai: alag-alag partitions ko alag-alag nodes par **shared-nothing cluster** mein rakh sakte hain, jisse data bahut saari disks par distribute ho jaata hai aur query load bahut saare processors par.

---

Section 1: Partitioning and Replication  
Partitioning ko usually replication ke saath combine kiya jaata hai taaki har partition ki copies multiple nodes par stored ho sakein **fault tolerance** ke liye. Ek node ek se zyada partition store kar sakta hai. Leader–follower replication ke saath, har node kuch partitions ke liye leader ho sakta hai aur kuch ke liye follower.
<img width="2880" height="1496" alt="image" src="https://github.com/user-attachments/assets/5ca40537-1ccc-4f96-92e4-ad5621c6b36d" />


**Figure 6-1: Combining replication and partitioning: each node acts as leader for some partitions and follower for other partitions**  
(Aapki pehli image: Node 1, 2, 3, 4 mein alag partitions ke leaders aur followers dikhaye gaye hain, replication streams per partition.)

---

Section 2: Partitioning of Key-Value Data  
Goal hai data aur query load ko evenly nodes par failana. Agar partitioning unfair hai, to use **skewed** kehte hain. Jis partition par disproportionately high load ho use **hot spot** kehte hain.

Records ko randomly nodes ko assign karne se data evenly distribute ho jaata hai lekin reads impossible ho jaati hain bina saare nodes query kiye. Hum isse better kar sakte hain.

### 2.1 Partitioning by Key Range  
Har partition ko keys ki ek continuous range assign karo (minimum se maximum tak), jaise paper encyclopedia ke volumes.

<img width="2880" height="1023" alt="image" src="https://github.com/user-attachments/assets/bf4c6fef-75f2-4071-a7a5-c92d1471da46" />

**Figure 6-2: A print encyclopedia is partitioned by key range**  
(Doosri image: A-ak se Zywiec tak alag volumes mein range-wise partition.)

Ranges zaroori nahi evenly spaced hon — partition boundaries data distribution ke hisaab se adapt hone chahiye.  
Boundaries manually administrator choose kar sakta hai ya automatically database.  
Use hota hai: Bigtable, HBase, RethinkDB, MongoDB (version 2.4 se pehle).

**Advantages**  
- Har partition ke andar, keys sorted order mein rakhi ja sakti hain → efficient **range scans**.
- Key ko concatenated index ki tarah treat kar sakte hain taaki related records ek hi query mein fetch ho sakein.
- Example: sensor data with key = timestamp (year-month-day-hour-minute-second) → easily fetch all readings from a particular month.

**Disadvantage: Hot Spots**  
Kuch access patterns hot spots lead karte hain. Agar key timestamp hai, to saare writes aaj wale partition mein jaate hain jabki baaki idle baithe hain.  
Solution: Har timestamp ke aage sensor name lagao taaki partitioning pehle sensor name se ho, phir time se. Write load partitions mein spread ho jaata hai. Trade-off: multiple sensors ki values ek time range mein fetch karne ke liye har sensor name ke liye alag range query karni padegi.

### 2.2 Partitioning by Hash of Key  
Ek **hash function** use karo yeh determine karne ke liye ki given key kis partition mein jaayegi. Accha hash function skewed data ko uniformly distributed bana deta hai.
<img width="2880" height="901" alt="image" src="https://github.com/user-attachments/assets/98dd8317-8a1c-4178-ba66-c469ae5b0ddd" />


**Figure 6-3: Partitioning by hash of key**  
(Teesri image: Timestamp values aur unke hashes dikhaye gaye hain, partition assignment hash range ke hisaab se.)

Har partition ko hashes ki ek range assign karo (keys ki range nahi). Jis key ka hash partition ki range mein aata hai, wo us partition mein store hota hai.  
Hash function cryptographically strong hona zaroori nahi. Cassandra aur MongoDB MD5 use karte hain; Voldemort Fowler–Noll–Vo use karta hai.  
**Caution:** Java ka `Object.hashCode()` aur Ruby ka `Object#hash` alag processes mein alag hash values return kar sakte hain — partitioning ke liye suitable nahi.

**Consistent Hashing**  
Karger et al. ne define kiya — randomly chosen partition boundaries se bina central control ke load evenly distribute karne ka tareeka.  
Yeh term confusing hai ("consistent" ka replica consistency ya ACID consistency se koi lena-dena nahi). Practice mein yeh approach databases ke liye bahut acchi tarah kaam nahi karti, isliye best hai ise simply **hash partitioning** kehna.

**Disadvantage: No Efficient Range Queries**  
Key ka hash use karke hum efficient range queries karne ki ability kho dete hain. Jo keys pehle adjacent thi, ab saare partitions mein scatter ho jaati hain.  
MongoDB (hash-based sharding): range queries saare partitions ko bhejni padti hain.  
Riak, Couchbase, ya Voldemort primary key par range queries support nahi karte.

**Cassandra's Compromise: Compound Primary Key**  
Table **compound primary key** ke saath declare ki ja sakti hai jismein kai columns ho.  
Sirf key ka pehla hissa partition determine karne ke liye hash kiya jaata hai; baaki columns SSTables mein data sort karne ke liye concatenated index ki tarah use hote hain.  
Query pehle column mein values ki range search nahi kar sakti, lekin agar pehle column ki fixed value specify ki jaaye, to baaki columns par efficient range scan kar sakti hai.  
Example: primary key `(user_id, update_timestamp)` → kisi particular user ke saare updates efficiently time interval mein retrieve karo, timestamp se sorted. Alag users alag partitions par ho sakte hain, lekin ek user ke andar updates single partition par ordered hain.

### 2.3 Skewed Workloads and Relieving Hot Spots  
Hashing hot spots reduce karta hai lekin poora avoid nahi kar sakta. Agar saare reads aur writes ek hi key ke liye ho (e.g., social media par celebrity user jiske millions of followers hain), to saari requests same partition par jaati hain.

**Technique: Random Prefix**  
Hot key ke beginning ya end mein ek random number (e.g., do-digit decimal) add kar do → writes 100 alag keys par alag partitions mein evenly split ho jaati hain.  
Trade-off: Reads ab saari 100 keys se padhni padengi aur results combine karne padenge. Sirf chhoti number of hot keys ke liye sense banata hai — zyadatar keys ke liye unnecessary overhead hai. Bookkeeping ki zaroorat hoti hai track karne ke liye ki kaunsi keys split ho rahi hain.  
Zyadatar data systems currently automatically highly skewed workloads compensate nahi kar sakte — application ki responsibility hai.

---

Section 3: Partitioning and Secondary Indexes  
**Secondary indexes** kisi record ko uniquely identify nahi karte — yeh ek tareeka hai particular value ke occurrences search karne ka (e.g., saari red cars dhundho). Yeh relational databases ki bread and butter hain aur document databases mein bhi common hain. Problem: secondary indexes neatly partitions mein map nahi hote.

Do main approaches:

### 3.1 Document-Partitioned Indexes (Local Indexes)  
Har partition apne secondary indexes maintain karta hai, jo sirf us partition ke documents cover karte hain. Jab aap document likhte ho, aap sirf us partition se deal karte ho jismein wo document ID hai.
<img width="2880" height="1365" alt="image" src="https://github.com/user-attachments/assets/a4cf1f35-8856-4542-8492-5e5f6ceb7bb0" />


**Figure 6-4: Partitioning secondary indexes by document**  
(Chauthi image: Partition 0 aur Partition 1 mein alag primary key index aur local secondary indexes dikhaye gaye hain, query "red car" ke liye dono partitions se scatter/gather.)

- **Writing:** Simple — sirf wo partition update karo jismein document hai.
- **Reading:** Query ko saare partitions mein bhejna padta hai aur results combine karne padte hain. Ise **scatter/gather** kehte hain — expensive ho sakta hai aur tail latency amplification ke liye prone.
- Use hota hai: MongoDB, Riak, Cassandra, Elasticsearch, SolrCloud, VoltDB.
- Database vendors recommend karte hain partitioning aise structure karo ki secondary index queries single partition se serve ho sakein, lekin yeh hamesha possible nahi (khaas kar jab ek query mein multiple secondary indexes ho).

### 3.2 Term-Partitioned Indexes (Global Indexes)  
**Global index** saare partitions ke data ko cover karta hai, lekin khud partitioned hota hai (primary key index se alag tareeke se).
<img width="2880" height="1223" alt="image" src="https://github.com/user-attachments/assets/16f2e9b6-59a6-4007-a538-7020813acf25" />


**Figure 6-5: Partitioning secondary indexes by term**  
(Paanchvi image: Partition 0 aur Partition 1 mein global secondary index by term (color, make) dikhaya gaya hai, jismein red cars ke references dono partitions mein stored hain.)

Jis term ko hum dhundh rahe hain, wo index ka partition determine karta hai (e.g., `color:red` → partition 0; `color:yellow` → partition 1).  
Term khud se partition kar sakte ho (range scans ke liye useful) ya term ke hash se (zyada even load distribution).

|                       | Document-Partitioned (Local)                  | Term-Partitioned (Global)                               |
|-----------------------|-----------------------------------------------|---------------------------------------------------------|
| **Reads**             | Saare partitions par scatter/gather (expensive) | Single partition lookup (efficient)                     |
| **Writes**            | Sirf ek partition update (fast)               | Index ke multiple partitions update karne pad sakte hain (slower, more complicated) |

Practice mein, global secondary indexes ke updates aksar **asynchronous** hote hain (change immediately index mein reflect nahi hota).  
Amazon DynamoDB: global secondary indexes normally fraction of a second mein update hote hain, lekin infrastructure faults ke dauran longer delays experience kar sakte hain.  
Aur use hota hai: Riak's search feature, Oracle data warehouse.

---

Section 4: Rebalancing Partitions  
Time ke saath, query throughput badhta hai, dataset size grow karta hai, ya machines fail hoti hain — data aur requests ko ek node se doosre node move karna padta hai. Ise **rebalancing** kehte hain.

**Requirements**  
- Rebalancing ke baad, load nodes ke beech fairly shared hona chahiye.
- Database ko rebalancing ke dauran reads aur writes accept karte rehna chahiye.
- Sirf utna hi data move ho jitna zaroori ho (network aur disk I/O minimize karo).

### 4.1 How NOT to Do It: hash mod N  
`hash(key) mod N` aasan lagta hai, lekin agar `N` badle, to zyadatar keys move karni padti hain. Example: `hash(key) = 123456` → 10 nodes par node 6, 11 nodes par node 3, 12 nodes par node 0. Excessively expensive.

### 4.2 Fixed Number of Partitions  
Nodes se bahut zyada partitions create karo aur har node ko kai partitions assign karo. Example: 10 nodes, 1,000 partitions → ~100 partitions per node.
<img width="2880" height="1557" alt="image" src="https://github.com/user-attachments/assets/be3e1039-ff57-4351-9737-3f13b2467d72" />


**Figure 6-6: Adding a new node to a database cluster with multiple partitions per node**  
(Chhathi image: 4 nodes se 5 nodes rebalancing mein partitions kuch same node par rahe, kuch migrate hue.)

Jab naya node add hota hai, to wo har existing node se kuch partitions chura leta hai. Jab node remove hota hai, to reverse hota hai.  
Sirf poori partitions move hoti hain. Partitions ki sankhya nahi badalti, na hi keys ka assignment partitions mein — sirf partitions ka assignment nodes mein badalta hai.  
Mismatched hardware ke liye account kar sakte ho — zyada powerful nodes ko zyada partitions assign karke.  
Use hota hai: Riak, Elasticsearch, Couchbase, Voldemort.  

Partitions ki sankhya usually setup par fix hoti hai aur baad mein change nahi ki jaati. Future growth ke liye kaafi high chuni jaani chahiye, lekin itni bhi nahi ki management overhead excessive ho jaaye.  
Difficulty: Sahi number choose karna agar dataset size highly variable ho (partitions bahut badi → expensive rebalancing; bahut chhoti → too much overhead).

### 4.3 Dynamic Partitioning  
Key range–partitioned databases ke liye, fixed boundaries inconvenient hoti (saara data ek partition mein end up ho sakta hai).

Jab partition configured size se zyada badh jaata hai (HBase default: 10 GB), to wo **split** ho jaata hai do parts mein. Jab partition shrink ho jaata hai threshold se neeche, to adjacent partition ke saath **merge** ho jaata hai. B-tree top-level splitting jaisa.  
Split ke baad, ek half doosre node par transfer kiya ja sakta hai load balance karne ke liye.  
**Fayda:** Partitions ki sankhya total data volume ke hisaab se adapt hoti hai — chhota data = few partitions (low overhead); bada data = bounded partition size.  
**Caveat:** Empty database single partition se start hota hai (saare writes ek node par jaate hain). Mitigate karne ke liye: **pre-splitting** — empty database par initial set of partitions configure karo (HBase, MongoDB). Key distribution pehle se jaanna zaroori hai.  
Key-range aur hash-partitioned data dono ke saath kaam karta hai (MongoDB 2.4+ dono support karta hai).

### 4.4 Partitioning Proportionally to Nodes  
Teesra option (Cassandra aur Ketama use karte hain): har node par partitions ki fixed sankhya (Cassandra default: 256 per node).

- Partition size dataset size ke proportionally grow karta hai jab tak node count unchanged ho; jab nodes add hote hain, partitions chhoti ho jaati hain.
- Jab naya node join karta hai, to wo randomly existing partitions choose karta hai split karne ke liye aur har ek ka half ownership leta hai.
- Randomization unfair splits produce kar sakti hai, lekin bahut saare partitions par average out ho jaata hai. Cassandra 3.0 ne improved algorithm introduce kiya unfair splits avoid karne ke liye.
- Requires hash-based partitioning (boundaries hash function range se pick ki jaati hain). Original definition of consistent hashing se most closely corresponds karta hai.

### 4.5 Automatic or Manual Rebalancing  
- **Fully automatic** — Convenient lekin unpredictable ho sakta hai. Rebalancing expensive hai (requests reroute karna, bada amount of data move karna). Network overload kar sakta hai aur performance harm kar sakta hai.
- **Dangerous with automatic failure detection** — Overloaded node temporarily slow ho → doosre nodes conclude karte hain ki wo dead hai → automatically rebalance → overloaded node aur network par additional load → cascading failure.
- **Fully manual** — Administrator explicitly partition assignment configure karta hai.
- **Middle ground** (Couchbase, Riak, Voldemort) — System suggested partition assignment generate karta hai, lekin administrator ko commit karna padta hai effect hone se pehle.

Human in the loop slower hai lekin operational surprises prevent karne mein help karta hai.

---

Section 5: Request Routing  
Jab client request karna chahta hai, to use kaise pata chalta hai ki kis node se connect kare? Yeh **service discovery** ka instance hai.

**Three Approaches**  
<img width="2880" height="1281" alt="image" src="https://github.com/user-attachments/assets/c6d2ac9b-a6b8-4eda-9fa6-db255c2aa04c" />


**Figure 6-7: Three different ways of routing a request to the right node**  
(Saatvi image: Client routing tier se contact kare, ya random node se jo forward kare, ya directly sahi node se.)

1. **Kisi bhi node ko contact karo** (e.g., round-robin load balancer ke through). Agar wo node partition own karta hai, to request handle karta hai; warna appropriate node ko forward kar deta hai.
2. **Routing tier** — Saari requests pehle ek partition-aware load balancer ke paas jaati hain, jo sahi node ko forward karta hai. Routing tier khud requests handle nahi karta.
3. **Client-aware** — Client partitioning jaanta hai aur directly appropriate node se connect karta hai.

**ZooKeeper**  
<img width="2880" height="1260" alt="image" src="https://github.com/user-attachments/assets/fda1ec3c-5aa7-40f5-81d0-8913f3d44378" />


**Figure 6-8: Using ZooKeeper to keep track of assignment of partitions to nodes**  
(Aathvi image: Client routing tier se key range ke hisaab se node IP address lookup karta hai, mapping ZooKeeper se aati hai.)

Bahut saare distributed data systems cluster metadata track karne ke liye separate coordination service jaise **ZooKeeper** par rely karte hain:

- Har node ZooKeeper mein register hota hai.
- ZooKeeper partitions to nodes ka authoritative mapping maintain karta hai.
- Routing tier ya partitioning-aware clients is information ko subscribe karte hain.
- Jab partition ownership change hoti hai ya node add/remove hota hai, ZooKeeper routing tier ko notify karta hai.
- Use hota hai: LinkedIn's Espresso (Helix ke through), HBase, SolrCloud, Kafka.

MongoDB apna khud ka **config server** implementation aur **mongos** daemons as routing tier use karta hai.

**Gossip Protocol**  
Cassandra aur Riak nodes ke beech cluster state changes disseminate karne ke liye **gossip protocol** use karte hain. Requests kisi bhi node ko bheji ja sakti hain, jo appropriate node ko forward kar deta hai (approach 1). External coordination service par dependency avoid karta hai lekin database nodes mein zyada complexity daalta hai.

**Parallel Query Execution**  
**MPP (Massively Parallel Processing)** relational database products jo analytics ke liye use hote hain, bahut zyada sophisticated queries (joins, filtering, grouping, aggregation) support karte hain. MPP query optimizer complex query ko execution stages aur partitions mein todta hai, jinmein se kai alag nodes par parallel execute hote hain.

---

Summary  
**Key Takeaways**  

**Two Main Partitioning Approaches:**  
- **Key range partitioning** — Keys sorted hain; partition kisi minimum se maximum tak range own karta hai. Efficient range queries, lekin hot spots ka risk. Typically rebalanced by dynamic splitting.
- **Hash partitioning** — Hash function har key par apply; partition hashes ki range own karta hai. Key ordering destroy karta hai (range queries inefficient), lekin load zyada evenly distribute karta hai. Commonly fixed number of partitions use hota hai.
- **Hybrid** — Compound key: ek hissa partition identify karta hai (hashed), doosra sort order ke liye (Cassandra).

**Secondary Index Partitioning:**  
- **Document-partitioned (local)** — Index data ke saath same partition mein stored. Write par single partition updated; read par scatter/gather.
- **Term-partitioned (global)** — Index separately partitioned by indexed values. Efficient reads from single partition; writes may update multiple partitions (often async).

**Rebalancing Strategies:**  
- `hash mod N` mat karo.
- Fixed number of partitions (Riak, Elasticsearch, Couchbase).
- Dynamic partitioning with splitting/merging (HBase, MongoDB).
- Proportional to nodes (Cassandra).
- Cascading failures se bachne ke liye human in the loop prefer karo.

**Request Routing:**  
- Kisi bhi node ko contact karo (gossip protocol — Cassandra, Riak).
- Routing tier (ZooKeeper — HBase, Kafka, Espresso).
- Client-aware partitioning.
