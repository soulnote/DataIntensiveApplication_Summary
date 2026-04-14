Batch processing (Chapter 10) mein, input **bounded** hota hai — uska size fixed aur pata hota hai. Reality mein, bahut saara data **unbounded** hota hai — yeh gradually time ke saath aata hai aur dataset kabhi "complete" nahi hota. Batch processors artificially data ko fixed duration ke chunks mein todte hain (jaise har din ke end mein poore din ka data process karna).

**Stream processing** fixed time slices ko poori tarah abandon kar deta hai aur har event ko aate hi process karta hai. "Stream" ka matlab hai wo data jo incrementally time ke saath available hota hai.

---

Section 1: Transmitting Event Streams  
Batch processing mein, inputs aur outputs files hain. Stream processing mein, ek record ko **event** kehte hain — ek chhota, self-contained, immutable object jismein details hoti hain ki kisi point in time par kya hua (timestamp ke saath). Related events ko **topic** ya **stream** mein group kiya jaata hai.

### 1.1 Messaging Systems  
Consumers ko naye events ke baare mein notify karne ka common approach. Do important sawaal:

1. **Agar producers consumers se tez send karein to kya hoga?** Teen options: messages drop kar do, queue mein buffer karo, ya **backpressure** apply karo (flow control — sender ko block karo jab tak recipient catch up na kare).
2. **Agar nodes crash ho jaayein to kya hoga?** Durability ke liye disk par likhna aur/ya replication chahiye (throughput aur latency ke cost par).

**Direct Messaging (Producer to Consumer)**  
- **UDP multicast** — Financial industry mein stock market feeds ke liye use hota hai (low latency). Application-level protocols lost packets recover karte hain.  
- **Brokerless messaging** — ZeroMQ, nanomsg (TCP ya IP multicast par publish/subscribe).  
- **StatsD, Brubeck** — Metrics collection ke liye unreliable UDP messaging.  
- **Webhooks** — Doosre service ke saath callback URL register karna; event par HTTP/RPC request.  

Limitation: generally assume karte hain ki producers aur consumers constantly online hain. Agar consumer offline ho, to messages miss ho sakte hain.

**Message Brokers**  
**Message broker** (message queue) ek tarah ka database hai jo message streams handle karne ke liye optimized hai. Producers broker ko likhte hain; consumers us se padhte hain.

- Clients jo aate-jaate hain (connect, disconnect, crash) tolerate karta hai.  
- Consumers generally **asynchronous** hote hain — producer message process hone ka wait nahi karta.

**Message Brokers vs Databases**  

|                          | Message Brokers                                              | Databases                                                    |
|--------------------------|--------------------------------------------------------------|--------------------------------------------------------------|
| **Data retention**       | Successful delivery ke baad auto-delete                       | Jab tak explicitly delete na karo tab tak rakhta hai          |
| **Working set**          | Assume short queues                                          | Large datasets handle karte hain                              |
| **Querying**             | Topics subscribe karo jo pattern match karein                 | Secondary indexes, arbitrary queries                          |
| **Notification**         | Clients ko notify karo jab data change ho                     | Point-in-time snapshots; changes ka notification nahi         |

Traditional brokers: RabbitMQ, ActiveMQ, HornetQ, TIBCO, IBM MQ, Azure Service Bus, Google Cloud Pub/Sub (JMS aur AMQP standards).

### 1.2 Multiple Consumers  

<img width="2880" height="1379" alt="image" src="https://github.com/user-attachments/assets/fb683360-1984-4102-805e-4674658dfce5" />

**Figure 11-1: (a) Load balancing: sharing the work of consuming a topic among consumers; (b) fan-out: delivering each message to multiple consumers**  
(Aapki pehli image: Producer do messages bhejta hai, load balancing mein ek consumer ko ek message, fan-out mein dono consumers ko dono messages.)

- **Load balancing** — Har message ek consumer ko deliver hota hai (kaam share karna). AMQP: multiple clients on same queue. JMS: shared subscription.  
- **Fan-out** — Har message saare consumers ko deliver hota hai (independent broadcast). JMS: topic subscriptions. AMQP: exchange bindings.  

Dono patterns combine kiye ja sakte hain: do consumer groups saare messages receive karein, lekin har group ke andar sirf ek node ko har message mile.

### 1.3 Acknowledgments and Redelivery  
<img width="2880" height="1457" alt="image" src="https://github.com/user-attachments/assets/b489376e-cba9-4fdd-a2e2-8d86073dde49" />


**Figure 11-2: Consumer 2 crashes while processing m3, so it is redelivered to consumer 1 at a later time**  
(Doosri image: Producer messages bhejta hai, Consumer 2 m3 process karte waqt crash ho jaata hai, m3 baad mein Consumer 1 ko redeliver hota hai.)

Consumers ko **explicitly acknowledge** karna padta hai jab wo message process kar chuke ho. Agar acknowledgment nahi aata (connection close/timeout), to broker message ko doosre consumer ko redeliver kar deta hai.  
**Ordering problem:** Load balancing ke saath, redelivery messages ko out of order process karwa sakti hai (e.g., m4 m3 se pehle). Avoid karne ke liye: har consumer ke liye alag queue use karo (no load balancing). Agar messages ek doosre se independent hain to problem nahi.

### 1.4 Partitioned Logs  
Traditional message brokers delivery ke baad messages delete kar dete hain — **destructive consumption**. Aap same consumer dobara nahi chala sakte aur same result nahi pa sakte. Naya consumer sirf registration ke baad aaye messages receive karta hai.

**Log-based message brokers** databases ki durable storage ko messaging ki low-latency notification ke saath combine karte hain.

<img width="2880" height="1528" alt="image" src="https://github.com/user-attachments/assets/9de44358-ac7f-46ab-903f-d880d866982a" />

**Figure 11-3: Producers send messages by appending them to a topic-partition file, and consumers read these files sequentially**  
(Teesri image: Topic A do partitions mein divided, producers append karte hain, consumer group har partition se sequentially padhta hai, offsets maintain kiye jaate hain.)

**Log** disk par records ka append-only sequence hai. Producer end mein append karta hai; consumer sequentially padhta hai.  
Throughput ke liye log **partitioned** hota hai. Topic = partitions ka group. Har partition ke andar, broker har message ko monotonically increasing **sequence number (offset)** assign karta hai.  
Implementations: Apache Kafka, Amazon Kinesis Streams, Twitter's DistributedLog. Partitioning aur replication ke through millions of messages per second achieve karte hain.

**Log-Based vs Traditional Messaging**  
- **Fan-out:** trivial — multiple consumers independently log padhte hain (padhne se delete nahi hota).  
- **Load balancing:** poori partitions consumer nodes ko assign karo (individual messages nahi). Har client apne assigned partitions ke saare messages sequentially consume karta hai.  
- **Downside:** max parallelism = number of partitions. Ek slow message poori partition ko hold up kar deta hai (**head-of-line blocking**).

**Consumer Offsets**  
Offset < current offset wale saare messages processed maane jaate hain. Broker ko sirf periodically **consumer offset** record karna hota hai (individual acknowledgments track nahi karne padte).  
Bahut similar hai database replication ke **log sequence number** se — message broker leader database ki tarah hai, consumer follower ki tarah.

**Disk Space**  
Log segments mein divided hota hai; purane segments delete ya archive ho jaate hain → bounded-size buffer (**circular buffer / ring buffer**).  
Ek typical 6 TB hard drive 150 MB/s sequential write throughput par → ~11 hours ka buffer. Practice mein, deployments days ya weeks ke messages retain karte hain.  
Throughput constant rehta hai retention ke bawajood (har message waise bhi disk par likha jaata hai) — unlike in-memory brokers jo disk par spill hone par slow ho jaate hain.

**Replaying Old Messages**  
Consuming ek **read-only** operation hai (log change nahi hota). Consumer offset consumer ke control mein hai.  
Aap kal ke offsets ke saath consumer ki copy chala sakte ho, output alag location par likho, aur reprocess karo. Repeatable any number of times.  
Yeh log-based messaging ko batch processing ke aur kareeb laata hai — derived data input data se clearly separated hota hai through a repeatable transformation.

---

Section 2: Databases and Streams  
Replication log database write events ka stream hai. **State machine replication** principle: agar har replica same events same order mein process kare, to sab same final state par pahunchte hain.

### 2.1 Keeping Systems in Sync  
Zyadatar applications kai technologies combine karte hain (OLTP database, cache, full-text index, data warehouse). Inhe sync mein rakhna padta hai.

**Dual writes** — Application directly har system mein likhta hai. Problems:

- **Race conditions** — Do clients concurrently likhte hain, alag systems mein alag order mein arrive ho sakte hain → permanent inconsistency.  
- **Partial failure** — Ek write succeed, doosra fail → systems out of sync (atomic commit problem).

<img width="2880" height="1132" alt="image" src="https://github.com/user-attachments/assets/b8d52fb0-d50f-408c-81f8-9dec999088b0" />

**Figure 11-4: In the database, X is first set to A and then to B, while at the search index the writes arrive in the opposite order**  
(Chauthi image: Database mein X = A phir X = B, search index mein X = B phir X = A, final value alag.)

**Solution:** Ek database ko **leader** (single source of truth) banao aur doosre systems ko uske change stream ke through **followers** banao.

### 2.2 Change Data Capture (CDC)  
Database mein likhe gaye saare data changes observe karna aur unhe ek stream ke roop mein extract karna jo doosre systems mein replicate kiya ja sake.

<img width="2880" height="1206" alt="image" src="https://github.com/user-attachments/assets/0ae27fa1-3da4-4170-a8ca-046afa6f90e9" />

**Figure 11-5: Taking data in the order it was written to one database, and applying the changes to other systems in the same order**  
(Paanchvi image: System of record database change data capture ke through log of changes produce karta hai, derived systems us log ko consume karte hain.)

Ek database ko leader banata hai; doosron ko followers mein badalta hai. Log-based message broker ordering preserve karta hai.  
Implementations: LinkedIn's Databus, Facebook's Wormhole, Yahoo!'s Sherpa, Bottled Water (PostgreSQL), Maxwell aur Debezium (MySQL), Mongoriver (MongoDB), GoldenGate (Oracle).  
Usually **asynchronous** — slow consumer add karne se source database affect nahi hota, lekin replication lag issues apply hote hain.

**Initial Snapshot**  
Agar aapke paas poori log history nahi hai, to changes apply karne se pehle ek **consistent snapshot** chahiye (jo change log mein known offset se correspond kare).

**Log Compaction**  
Storage engine periodically same key wale log records dhundhta hai, duplicates hata deta hai, har key ke liye sirf most recent update rakhta hai. **Tombstone** (null value) deletion indicate karta hai.

Log compaction ke saath, aap derived data system rebuild kar sakte ho naya consumer offset 0 se chala kar — log mein database ki har key ke liye most recent value hoti hai (full copy bina snapshot ke). Apache Kafka supported hai.

**API Support for Change Streams**  
Databases increasingly change streams ko first-class interface ki tarah support kar rahe hain: RethinkDB (query notifications), Firebase aur CouchDB (data sync), Meteor (MongoDB oplog), VoltDB (stream of committed tuples), Kafka Connect (CDC tools ko Kafka ke saath integrate karta hai).

### 2.3 Event Sourcing  
Similar to CDC lekin abstraction ke different level par:

- **CDC:** Application database mutably use karta hai. Changes low level par extract kiye jaate hain (replication log parsing). Application ko CDC ke baare mein aware hone ki zaroorat nahi.
- **Event sourcing:** Application logic explicitly **immutable events** par built hota hai jo event log mein likhe jaate hain. Events user actions ko application level par reflect karte hain, low-level state changes nahi. Updates/deletes discouraged hain.

Event sourcing applications evolve karna aasan banata hai, debugging mein help karta hai, aur application bugs se guard karta hai.

**Commands vs Events**  
- **Command** user request hai jo fail ho sakti hai (e.g., validation reject kar sakti hai).  
- Jab validation success ho jaaye aur command accept ho jaaye, to wo **event** ban jaata hai — durable aur immutable.  

Event stream ka consumer event reject nahi kar sakta. Validation synchronously event likhe jaane se pehle honi chahiye.

### 2.4 State, Streams, and Immutability  
Mutable state aur append-only log of immutable events ek hi sikke ke do pehlu hain:

- **Application state** = integral of the event stream over time.  
- **Change stream** = derivative of the state by time.

<img width="2880" height="522" alt="image" src="https://github.com/user-attachments/assets/e39a8ee7-95b7-4867-b00b-5b39a92920da" />

**Figure 11-6: The relationship between the current application state and an event stream**  
(Chhathi image: State = stream ka integral, stream = state ka derivative — formula ke saath.)

Pat Helland: *"The truth is the log. The database is a cache of a subset of the log."*

**Advantages of Immutable Events**  
- Financial bookkeeping ki tarah — galtiyan compensating transactions add karke theek ki jaati hain, original ko mitaya nahi jaata.  
- Agar buggy code galat data likhe, to append-only log ke saath recovery destructive overwrites se kahin aasan hai.  
- Zyada information capture karta hai (e.g., customer ne cart mein item add kiya phir remove kiya — analytics ke liye useful hai chahe net effect zero ho).

**Deriving Several Views from the Same Event Log (CQRS)**  
Data jis form mein likha jaata hai (event log) use us form se alag karo jis form mein padha jaata hai (application state).  
**Command Query Responsibility Segregation (CQRS)** — Ek hi write-optimized event log se multiple read-optimized views derive karna.  
Normalization vs denormalization ki debates largely irrelevant ho jaati hain — aap read views mein denormalize kar sakte ho jabki event log source of truth bana rehta hai.

**Concurrency Control**  
Event log ke consumers usually asynchronous hote hain → user log mein likh sakta hai phir derived view se padhe jo abhi update nahi hua (read-your-writes problem).  
Agar event log aur application state same tareeke se partitioned hain, to single-threaded log consumer ko writes ke liye koi concurrency control nahi chahiye.

**Limitations of Immutability**  
- Workloads with high update/delete rates → immutable history prohibitively large ho sakta hai.  
- Privacy regulations actually data delete karne ko require kar sakti hain (sirf "deleted" event append karna kaafi nahi). Immutable logs ke saath yeh genuinely hard hai.

---

Section 3: Processing Streams  
Stream ke saath kya karna hai iske teen options:

1. Storage system mein likho (database, cache, search index) — batch workflow output ka streaming equivalent.  
2. Users ko push karo (email alerts, push notifications, real-time dashboard).  
3. Ek ya zyada input streams process karke output streams produce karo — processing stages ki pipeline.

Code ka piece jo streams process karta hai use **operator** ya **job** kehte hain. Batch se crucial difference: stream kabhi end nahi hoti.

### 3.1 Uses of Stream Processing  

**Complex Event Processing (CEP)**  
Stream mein events ke kuch patterns search karne ke liye rules specify karna (jaise events ke liye regex).  
High-level declarative query language (jaise SQL) ya graphical UI use karta hai.  
Queries aur data ke beech ka relationship databases ke comparison mein reversed hai: queries long-term stored hoti hain, data unke through flow hota hai.  
Implementations: Esper, IBM InfoSphere Streams, Apama, TIBCO StreamBase, SQLstream.

**Stream Analytics**  
Large numbers of events par aggregations aur statistical metrics ki taraf oriented (specific event sequences nahi).  
Examples: events ki rate per time interval, rolling averages, current statistics ko previous time intervals se compare karna.  
Fixed time intervals (**windows**) par compute hota hai.  
Probabilistic algorithms use kar sakte hain: Bloom filters (set membership), HyperLogLog (cardinality estimation), percentile estimation — kam memory mein approximate results.  
Frameworks: Apache Storm, Spark Streaming, Flink, Concord, Samza, Kafka Streams. Hosted: Google Cloud Dataflow, Azure Stream Analytics.

**Maintaining Materialized Views**  
Derived data systems (caches, search indexes, data warehouses) ko change stream consume karke up to date rakhna.  
Stream analytics ke unlike, isme arbitrary time period ke saare events chahiye ho sakte hain (sirf time window nahi) — ek window jo "beginning of time" tak stretch karta hai.  
In principle, koi bhi stream processor materialized view maintenance ke liye use ho sakta hai, lekin events forever maintain karne ki zaroorat kuch analytics-oriented frameworks ki assumptions ke khilaf jaati hai jo mostly limited duration ki windows par operate karte hain.  
Samza aur Kafka Streams is type ke usage support karte hain, Kafka ke log compaction par build karte hue. Samza khaas taur par suitable hai kyunki yeh Kafka ke saath tightly integrate hota hai: yeh Kafka topics input streams ke liye aur local state changes replicate karne ke liye use karta hai, jisse fault-tolerant local state enable hota hai jo log-compacted changelog se rebuild kiya ja sakta hai.

**Search on Streams**  
Queries long-term store karo, documents unke past run karo (conventional search ka reverse).  
Elasticsearch ka **percolator** feature yeh implement karta hai.

**Message Passing and RPC**  
- **Actor frameworks** primarily concurrency management ke liye hain; stream processing primarily data management ke liye.  
- Actors: ephemeral, one-to-one communication, arbitrary patterns (including cyclic). Streams: durable, multi-subscriber, acyclic pipelines.

### 3.2 Reasoning About Time  

**Event Time vs Processing Time**  

<img width="2880" height="1619" alt="image" src="https://github.com/user-attachments/assets/879492f0-881b-4de6-9a18-0cbbbb238338" />

**Figure 11-7: Windowing by processing time introduces artifacts due to variations in processing rate**  
(Saatvi image: Actual request rate constant 10 hai, lekin processing rate ke hisaab se measured rate vary karta hai — 10, 11, 12, ... 20.)

- **Event time** — Jab event actually hua (timestamp embedded in the event).  
- **Processing time** — Jab stream processor event process kare (local system clock).  

Processing delayed ho sakti hai (queueing, network faults, restarts, reprocessing). Messages out of order arrive ho sakte hain.  
Event time aur processing time confuse karne se galat data milta hai (e.g., jab backlog reprocess hota hai to event rate mein spike dikhta hai, jabki original events time par spread the).

**Straggler Events**  
Aap kabhi sure nahi ho sakte ki particular window ke saare events receive ho gaye.  
Do options: stragglers ignore karo (dropped events ko metric ki tarah track karo), ya **correction** publish karo (updated value with stragglers included).

**Whose Clock Are You Using?**  
Mobile apps events offline buffer kar sakte hain aur hours ya days baad bhej sakte hain.  
Device clocks par bharosa nahi kar sakte (galat set ho sakte hain).  
**Three timestamps approach:**  
1. Event occur hone ka time (device clock)  
2. Event server ko bheje jaane ka time (device clock)  
3. Event server dwara receive hone ka time (server clock)  

Timestamp 3 mein se timestamp 2 subtract karke device clock offset estimate karo, phir timestamp 1 par apply karo.

### 3.3 Types of Windows  

| Window Type | Description |
|-------------|-------------|
| **Tumbling** | Fixed length, non-overlapping. Har event exactly ek window mein belong karta hai. E.g., 1-minute windows: 10:03:00–10:03:59, 10:04:00–10:04:59. |
| **Hopping** | Fixed length, overlapping (smoothing provide karta hai). E.g., 5-minute window with 1-minute hop: 10:03:00–10:07:59, 10:04:00–10:08:59. Implemented as tumbling windows + adjacent windows par aggregation. |
| **Sliding** | Contains all events within some interval of each other. E.g., 5-minute sliding window includes events at 10:03:39 and 10:08:12 (< 5 min apart). Sorted buffer se implement hota hai, expired events remove karta hai. |
| **Session** | No fixed duration. Same user ke saare events jo time mein closely together occur karein group karta hai. User inactive hone ke kuch period (e.g., 30 minutes) ke baad end hota hai. Website analytics ke liye common. |

### 3.4 Stream Joins  
Stream processing mein teen types ke joins:

**Stream-Stream Join (Window Join)**  
Example: search events ko click events ke saath correlate karna (same session ID) click-through rate calculate karne ke liye.  
Stream processor state maintain karta hai (e.g., last hour ke saare events, session ID se indexed). Jab naya event aaye, doosre index mein matching event check karo.  
Agar search bina matching click ke expire ho jaaye, to event emit karo ki search click nahi hua.

**Stream-Table Join (Stream Enrichment)**  
Example: activity events ko user profile information se enrich karna (user ID → profile data).  
Database ki copy stream processor mein load karo (in-memory hash table ya local disk index) local lookups ke liye bina network round-trips ke.  
Local copy ko change data capture ke through up to date rakho — user profile database ke changelog ko subscribe karo.  
Stream-stream join jaisa, lekin table changelog "beginning of time" tak pahunchne wali window use karta hai (newer records older ko overwrite karte hain).

**Table-Table Join (Materialized View Maintenance)**  
Example: Twitter timeline cache. Jab naya tweet bheja jaaye, use sabhi followers ki timelines mein add karo. Jab user kisi ko unfollow kare, unke tweets remove karo.  
Stream processor har user ke liye followers ka database maintain karta hai.  
Equivalent to maintaining a materialized view of a SQL join between tweets aur follows tables.

**Time-Dependence of Joins**  
Teeno join types require karte hain ki stream processor ek join input ke based par state maintain kare aur doosre input ke messages par use query kare.  
Agar state time ke saath change ho, to join ke liye kaunsa version use karein? Example: agar aap cheezein bechte ho, to invoices par sahi tax rate apply karna hai, jo country, product type, aur sale date par depend karta hai (kyunki tax rates change hote hain). Sales ko tax rates ki table se join karte waqt, aapko sale ke time ka rate chahiye, jo current rate se alag ho sakta hai agar historical data reprocess kar rahe ho.  
Agar streams ke across ordering undetermined hai, to join **nondeterministic** ho jaata hai — aap same job same input par dobara chala kar necessarily same result nahi pa sakte (input streams ke events differently interleaved ho sakte hain).  
Data warehouses mein, ise **slowly changing dimension (SCD)** kehte hain. Aksar ise address karne ke liye joined record ke har version ke liye unique identifier use kiya jaata hai: jab bhi tax rate change ho, use naya identifier diya jaata hai, aur invoice mein sale ke time ke tax rate ka identifier include hota hai. Yeh join deterministic banata hai, lekin iska consequence hai ki log compaction possible nahi hai (records ke saare versions retain karne padte hain).

### 3.5 Fault Tolerance  
- **Batch processing:** agar task fail ho, to use same immutable input par restart karo. Output tabhi visible hota hai jab task successfully complete ho → aisa lagta hai jaise har record **exactly once** process hua (better described as **effectively-once**).  
- **Stream processing:** stream kabhi end nahi hoti, isliye aap simply beginning se restart nahi kar sakte.

**Microbatching**  
Stream ko chhote blocks (~1 second) mein todo, har block ko miniature batch process ki tarah treat karo. Spark Streaming use karta hai.  
Implicitly ek tumbling window provide karta hai jo batch size ke equal hoti hai.  
Trade-off: smaller batches = zyada scheduling overhead; larger batches = zyada delay.

**Checkpointing**  
Periodically state ke rolling checkpoints generate karo, durable storage mein likho. Crash par, most recent checkpoint se restart karo. Apache Flink use karta hai.  
Checkpoints message stream mein **barriers** se trigger hote hain (microbatch boundaries jaisa lekin bina fixed window size force kiye).

**Limitation**  
Microbatching aur checkpointing dono stream processing framework ke andar **exactly-once** semantics provide karte hain. Lekin ek baar output framework se bahar chala jaaye (database mein likhna, emails bhejna), failed task restart hone par external side effect do baar ho sakta hai.

**Atomic Commit**  
Ensure karo ki saare outputs aur side effects tabhi effect lein agar processing successful ho (messages to downstream operators, database writes, state changes, input acknowledgments).  
Google Cloud Dataflow, VoltDB mein use hota hai, Apache Kafka ke liye planned hai.  
XA ke unlike, yeh transactions stream processing framework ke andar internal rakhte hain (heterogeneous technologies ke across nahi). Overhead amortized hota hai kyunki har transaction mein kai messages process hote hain.

**Idempotence**  
Ek **idempotent** operation ka same effect hota hai chahe ek baar karo ya kai baar (e.g., key ko fixed value set karna idempotent hai; counter increment karna nahi).  
Non-idempotent operations ko bhi extra metadata ke saath idempotent banaya ja sakta hai (e.g., har database write ke saath Kafka message offset include karo duplicate updates detect karne ke liye).  
Requires: same messages same order mein replay ho (log-based broker), deterministic processing, same value par concurrent updates nahi.

**Rebuilding State After a Failure**  
- State ko remote storage mein replicate karo (Flink: periodic snapshots to HDFS; Samza/Kafka Streams: state changes to a dedicated Kafka topic with log compaction; VoltDB: redundant processing on multiple nodes).  
- Input streams se rebuild karo — Agar state short window ke aggregations hai, to input events replay karo. Agar state local database replica hai, to log-compacted change stream se rebuild karo.

---

Summary  
**Key Takeaways**  

**Two Types of Message Brokers:**  

|                          | AMQP/JMS-style                                              | Log-based (Kafka, Kinesis)                                   |
|--------------------------|-------------------------------------------------------------|--------------------------------------------------------------|
| **Delivery**             | Individual messages to consumers                            | All messages in a partition to one consumer                  |
| **Acknowledgment**       | Per-message                                                 | Consumer offset checkpoint                                   |
| **After delivery**       | Message deleted                                             | Message retained on disk                                     |
| **Ordering**             | May be reordered with load balancing                        | Preserved within partition                                   |
| **Replay**               | Not possible (destructive)                                  | Possible (read-only, offset manipulation)                    |
| **Best for**             | Task queues, message-by-message parallelism                 | High throughput, ordering important, replayability           |

**Where Streams Come From:**  
- User activity events, sensors, data feeds  
- Database changelogs via change data capture (CDC) ya event sourcing  
- Log compaction stream ko database ki full copy retain karne deta hai  

**Processing Streams:**  
- **CEP** — Event sequences par pattern matching  
- **Stream analytics** — Windowed aggregations and statistics  
- **Materialized views** — Derived data systems ko up to date rakhna  
- **Stream joins** — Stream-stream (window), stream-table (enrichment), table-table (materialized view maintenance)  

**Reasoning About Time:**  
- Event time ≠ processing time. Event timestamps use karo, stragglers handle karo.  
- Untrusted device clocks ke liye three-timestamp approach.  
- Char window types: tumbling, hopping, sliding, session.  

**Fault Tolerance:**  
- Microbatching (Spark Streaming), checkpointing (Flink), atomic commit, idempotence.  
- Exactly-once / effectively-once semantics framework ke andar achievable; external side effects ke liye atomic commit ya idempotence chahiye.
