Yeh final chapter perspective ko future ki taraf shift karta hai — ideas propose karta hai ki hum fundamentally kaise improve kar sakte hain tarike jinse hum reliable, scalable, aur maintainable applications design aur build karte hain. Goal: applications jo robust, correct, evolvable, aur ultimately humanity ke liye beneficial ho.

---

Section 1: Data Integration  
Kisi bhi given problem ke liye koi ek right solution nahi hai — bahut saare approaches alag-alag circumstances mein appropriate hote hain. Complex applications mein, data aksar kai alag tareekon se use hota hai, aur koi bhi single piece of software sab ke liye suitable nahi hai. Aap inevitably kai alag pieces of software ko jodne par majboor ho jaate ho.

### 1.1 Combining Specialized Tools  
Database aur search index ke alawa, aapko zaroorat pad sakti hai: analytics systems (data warehouses, batch/stream processing), caches, denormalized views, machine learning/recommendation systems, aur notification systems. Data integration ki need tabhi clear hoti hai jab aap zoom out karte ho aur poori organization ke across dataflows consider karte ho.

### 1.2 Reasoning About Dataflows  
Jab same data ki copies kai storage systems mein maintain ki jaati hain, to inputs aur outputs ke baare mein bahut clear raho: data sabse pehle kahan likha jaata hai, aur kaunsi representations kis source se derive hoti hain?

- Data ko pehle **system of record database** mein likho, changes CDC (Change Data Capture) ke through capture karo, aur unhe search index mein same order mein apply karo.  
- Agar CDC hi index update karne ka ek maatra tareeka hai, to aap confident ho sakte ho ki index entirely system of record se derived hai aur isliye uske saath consistent hai.  
- Application ko directly database aur search index dono mein likhne dena **dual writes** ke problems introduce karta hai (race conditions, partial failures).  
- Event log ke based par derived data update karna **deterministic** aur **idempotent** banaya ja sakta hai, jisse fault recovery easy ho jaati hai.

### 1.3 Derived Data vs Distributed Transactions  

| Distributed Transactions                     | Log-Based Derived Data                                         |
|----------------------------------------------|----------------------------------------------------------------|
| **Ordering**                                 | Locks for mutual exclusion                                     | Log for ordering                                               |
| **Atomicity**                                | Atomic commit (2PC)                                            | Deterministic retry + idempotence                              |
| **Consistency**                              | Linearizability (read-your-writes)                             | Asynchronous (no timing guarantees by default)                 |
| **Fault tolerance**                          | XA has poor fault tolerance                                    | More robust and practical                                      |

**Log-based derived data** is the most promising approach for integrating different data systems.

### 1.4 The Limits of Total Ordering  
Chhote systems ke liye (single-leader replication) totally ordered event log construct karna feasible hai. Lekin scale par limitations ubharti hain:

- **Throughput ek single machine se exceed kare** → log ko multiple machines par partition karna padta hai → partitions ke beech order ambiguous hota hai.  
- **Multiple geographically distributed datacenters** → separate leader per datacenter → across datacenters ordering undefined.  
- **Microservices** → har service ka independent durable state → different services se aane wale events ka koi defined order nahi.  
- **Client-side state (offline-capable apps)** → clients aur servers events ko different orders mein dekhte hain.  

Total order decide karna consensus ke equivalent hai — zyadatar consensus algorithms single node ke throughput ke liye design hain. Partial ordering ke approaches: logical timestamps, unique event identifiers se causal dependencies record karna, conflict resolution algorithms (CRDTs).

### 1.5 Batch and Stream Processing  
Batch aur stream processors data ko system of record se derived datasets (search indexes, materialized views, recommendations, aggregate metrics) mein transform karne ke tools hain.

- Spark stream processing ko batch ke upar perform karta hai (**microbatching**).  
- Apache Flink batch processing ko stream ke upar perform karta hai.  
- In principle, ek doosre ko emulate kar sakte hain, lekin performance characteristics alag hoti hain.

**Maintaining Derived State**  
- Batch processing ka **functional flavor** hai: deterministic, pure functions, immutable inputs, append-only outputs.  
- Stream processing ise managed, fault-tolerant state ke saath extend karta hai.  
- Derived data ko ek **data pipeline** ki tarah sochna helpful hai (transformation function applied to a system of record).

**Reprocessing Data for Application Evolution**  
- Stream processing changes ko low delay ke saath reflect karta hai.  
- Batch processing bade amounts of historical data reprocess karne deta hai taaki naye views derive kiye ja sakein.  
- Reprocessing dataset ko completely different model mein restructure karne enable karta hai (simple schema changes jaise optional field add karne se beyond).  
- **Gradual migration** — Purane aur naye schemas ko side by side do independently derived views ki tarah maintain karo. Users ko gradually shift karo. Har stage easily reversible hai.

**Schema Migrations on Railways**  
19th-century England mein railway building ke early days mein, gauge (do rails ke beech ka distance) ke liye kai competing standards the. Ek gauge ki trains doosre gauge ke tracks par nahi chal sakti thi. 1846 mein single standard gauge decide hone ke baad, doosre gauges ke tracks convert karne the — lekin yeh kaise karein bina months ke liye line shutdown kiye? Solution: pehle track ko **dual gauge** mein convert karo ek teesri rail add karke. Yeh gradually kiya ja sakta hai, aur dono gauges ki trains do rails use karke chal sakti hain. Eventually, jab saari trains convert ho jaayein, to nonstandard rail hata di jaaye. Yeh analogous hai purane aur naye schemas ko side by side maintain karne se. Phir bhi, yeh expensive hai, isliye aaj bhi nonstandard gauges exist karte hain (e.g., San Francisco ka BART system US ke majority se alag gauge use karta hai).

**The Lambda Architecture**  
Incoming data immutable events ke roop mein record hota hai. Do parallel systems: **batch** (Hadoop MapReduce) for correctness, **stream** (Storm) for speed.  
- Stream processor quickly approximate updates produce karta hai; batch processor baad mein corrected version produce karta hai.  
- Reasoning: batch processing simpler aur less bug-prone hai, jabki stream processors less reliable aur fault-tolerant banana mushkil maane jaate hain.  

**Problems:**  
1. Same logic ko batch aur stream dono frameworks mein maintain karna significant additional effort hai. Summingbird jaise libraries abstraction provide karte hain, lekin do alag systems ko debug, tune, aur maintain karne ki operational complexity bani rehti hai.  
2. Stream aur batch pipelines alag outputs produce karte hain, to user requests respond karne ke liye merge karna padta hai. Simple tumbling windows ke aggregations ke liye fairly easy hai, lekin joins aur sessionization jaise complex operations ke liye significantly harder ho jaata hai.  
3. Poori historical dataset reprocess karna great hai, lekin large datasets par frequently karna expensive hai. Batch pipeline ko aksar incremental batches (e.g., har ghante ke end mein ek ghante ka data) process karna padta hai, jo stragglers aur windows jo batch boundaries cross karte hain unke problems raise karta hai. Batch computation ko incrementalize karna complexity add karta hai, use streaming layer jaisa bana deta hai — batch layer ko simple rakhne ke goal ke against jaata hai.

**Unifying Batch and Stream Processing**  
Recent work same system mein batch aur stream dono computations allow karta hai, requiring:  
- Historical events ko same engine se replay karne ki ability jo recent events handle karta hai.  
- Stream processors ke liye **exactly-once semantics**.  
- **Windowing by event time**, not processing time.

---

Section 2: Unbundling Databases  
Sabse abstract level par, databases, Hadoop, aur operating systems sab same functions perform karte hain: data store karna aur aapko process aur query karne dena.

### 2.1 Composing Data Storage Technologies  
Ek database already internally bahut kuch karta hai jo derived data systems jaisa lagta hai:

- **Secondary indexes** — Primary data se derived, har write par sync mein rakha jaata hai.  
- **Materialized views** — Query results ka precomputed cache.  
- **Replication logs** — Doosre nodes par copies up to date rakhna.  
- **Full-text search indexes** — Kuch relational databases mein built-in.  

`CREATE INDEX` karna remarkably similar hai naye follower replica set up karne ya CDC bootstrap karne se — database existing dataset reprocess karta hai aur index ko naye view ke roop mein derive karta hai.

Batch aur stream processors triggers, stored procedures, aur materialized view maintenance ke elaborate implementations ki tarah hain — lekin various different pieces of software dwara provide kiye gaye, alag machines par chalte hue, alag teams dwara administer kiye gaye.

### 2.2 Federated Databases vs Unbundled Databases  

| Federated Databases (Polystore)                              | Unbundled Databases                                           |
|--------------------------------------------------------------|---------------------------------------------------------------|
| **Focus**                                                    | Unifying reads — unified query interface across storage engines | Unifying writes — synchronizing writes across systems         |
| **Approach**                                                 | High-level query language over diverse backends (e.g., PostgreSQL foreign data wrappers) | Asynchronous event log with idempotent writes                 |
| **Philosophy**                                               | Relational tradition — single integrated system               | Unix tradition — composing loosely coupled components         |

Writes synchronize karne ka traditional approach (distributed transactions / XA) **galat solution** hai. **Ordered log of events with idempotent consumers** is a much simpler abstraction that works across heterogeneous systems.

**Advantages of Log-Based Integration (Loose Coupling)**  
- **System level** — Asynchronous event streams system ko outages ke liye zyada robust banate hain. Agar consumer fail ho jaaye, to event log messages buffer karta hai; faulty consumer fix hone par catch up kar leta hai.  
- **Human level** — Different teams independently different derived data systems develop, improve, aur maintain kar sakte hain. Har team **ek kaam acchi tarah karne** par focus karti hai.

### 2.3 Designing Applications Around Dataflow  
**"Database inside-out" approach** — specialized storage aur processing systems ko application code ke saath compose karna. Related to dataflow languages (Oz, Juttle), functional reactive programming (Elm), logic programming (Bloom), aur even spreadsheets (automatic recalculation when inputs change).

**Application Code as a Derivation Function**  
- Secondary index → straightforward transformation (indexed fields pick karo, sort karo).  
- Full-text search index → NLP functions (language detection, stemming, synonyms) + inverted index.  
- Machine learning model → training data se derived via feature extraction aur statistical analysis.  
- Cache → aggregation of data in the form displayed in the UI.  

Jab derivation function application-specific ho (ML, full-text search, caching), to use custom application code hona chahiye, sirf database ka built-in feature nahi. Modern deployment tools (Mesos, YARN, Docker, Kubernetes) is code ko run karne ke liye database stored procedures se better suited hain.

**Stream Processors and Services**  
Stream operators ko dataflow systems mein compose karna microservices jaisa hai, lekin synchronous request/response ki jagah **one-directional, asynchronous message streams** ke saath.

Example: purchase ke liye currency conversion. Synchronous RPC exchange rate service ko karne ki jagah (jo latency aur failure dependency add karta hai), purchase events aur exchange rate update events ke beech **stream-table join** use karo. Stream processor exchange rate changelog subscribe karta hai aur local copy maintain karta hai — query time par koi network request nahi chahiye.

### 2.4 Observing Derived State  

**Write Path and Read Path**  

<img width="2880" height="1101" alt="image" src="https://github.com/user-attachments/assets/0fa70c84-9802-40b1-b370-8620d08b34a9" />

**Figure 12-1: In a search index, writes (document updates) meet reads (queries)**  
(Aapki image: Write path mein document update linguistic analysis se guzar kar index mein store hota hai, read path mein query execution index se results lata hai.)

- **Write path** — Eagerly precomputed when data comes in (batch and stream processing, updating derived datasets). **Eager evaluation** ki tarah.  
- **Read path** — Lazily computed when someone asks for it (serving user requests). **Lazy evaluation** ki tarah.  

Derived dataset wahi jagah hai jahan write path aur read path milte hain — trade-off between work done at write time vs read time.  
Caches, indexes, aur materialized views boundary shift karte hain: read path par effort bachane ke liye write path par zyada kaam (precomputing).

**Stateful, Offline-Capable Clients**  
- On-device state server par state ka cache hai. Screen par pixels materialized view hain; model objects remote state ki local replica hain.  
- **Server-sent events** aur **WebSockets** server ko actively state changes browser tak push karne dete hain.  
- Write path ko end-user device tak extend karna — state changes ek device se, event logs aur stream processors ke through, doosre device ki UI tak flow karte hain. Fairly low delay (under one second end-to-end) ke saath propagate hote hain.

**Reads Are Events Too**  
Read requests ko streams of events ki tarah represent kiya ja sakta hai, same stream operator ko route kiya ja sakta hai jo writes handle karta hai — read queries aur database ke beech **stream-table join** perform karna. One-off read join se guzar kar bhool jaata hai; subscribe request past aur future events ke saath persistent join hai.

Read events record karne se causal dependencies aur data provenance better track hota hai (e.g., purchase decision lene se pehle user ne kya dekha).

---

Section 3: Aiming for Correctness  
Hum chahte hain applications jo reliable aur correct ho — well-defined semantics even in the face of faults. Transactions (atomicity, isolation, durability) traditional tools rahe hain, lekin unki foundations weaker hain jitni lagti hain (weak isolation levels, single datacenter tak limited, limited scale).

### 3.1 The End-to-End Argument for Databases  
Sirf isliye ki application strong safety properties (e.g., serializable transactions) wala data system use karta hai, iska guarantee nahi ki application data loss ya corruption se free hai. Application bugs ab bhi galat data likh sakte hain.

**Exactly-Once Execution**  
Operations ko **idempotent** banana sabse effective approaches mein se ek hai — same effect chahe ek baar execute karo ya multiple baar.  
Requires additional metadata (e.g., set of operation IDs that have updated a value) aur failover par fencing.

**Duplicate Suppression**  
- TCP ek single connection ke andar duplicate packets suppress karta hai, lekin across connections nahi.  
- Agar client `COMMIT` bheje lekin acknowledgment receive karne se pehle connection drop ho jaaye, to client nahi jaanta ki transaction commit hua ya nahi. Retry karne se do baar execute ho sakta hai.  
- Database-level deduplication bhi end-user device se aane wale duplicates (e.g., user weak cellular connection par HTTP POST retry kare) ke liye help nahi karta.

**Operation Identifiers**  
- Client side par har operation ke liye unique identifier (UUID) generate karo. Use hidden form field ki tarah include karo.  
- Operation ID ko poore raaste database tak pass karo. Request ID par uniqueness constraint use karo ensure karne ke liye ki har operation sirf ek baar execute ho.  
- **Requests table** ek event log ki tarah kaam karta hai (event sourcing ki taraf ishara).

**The End-to-End Argument**  
Saltzer, Reed, and Clark (1984): *"The function in question can completely and correctly be implemented only with the knowledge and help of the application standing at the endpoints of the communication system."*

- TCP duplicate suppression, stream processor exactly-once semantics, aur database transactions sab useful low-level reliability features hain, lekin yeh apne aap mein end-to-end correctness ke liye sufficient nahi hain.  
- Application ko khud **end-to-end measures** lene padte hain (e.g., operation IDs se duplicate suppression).  
- Low-level mechanisms higher levels par problems ki probability reduce karte hain, lekin remaining higher-level faults ab bhi handle karne padte hain.

### 3.2 Enforcing Constraints  

**Uniqueness Constraints Require Consensus**  
Distributed setting mein, uniqueness enforce karne ke liye **consensus** chahiye — typically implemented by funneling all events through a single leader node.  
Uniqueness ke liye value ke based par partition karke scale out kiya ja sakta hai (e.g., username se partition karo).

**Uniqueness in Log-Based Messaging**  
Ek stream processor log partition ke saare messages ko sequentially single thread par consume karta hai:

1. Username ke liye har request us partition mein append hoti hai jo username ke hash se determine hoti hai.  
2. Stream processor requests ko sequentially padhta hai, track karta hai kaunse usernames taken hain, aur success ya rejection messages output stream mein emit karta hai.  
3. Client output stream watch karta hai apne result ke liye.  

Yeh wahi algorithm hai jo total order broadcast use karke linearizable storage implement karta hai. Partitions ki sankhya badha kar scale karta hai.

**Multi-Partition Request Processing Without Atomic Commit**  
Example: account A se account B mein paise transfer karna (different partitions):

1. Transfer request ko unique request ID milta hai aur request ID ke based par log partition mein append hota hai.  
2. Stream processor request padhta hai aur do messages emit karta hai: debit instruction (A se partitioned) aur credit instruction (B se partitioned), dono mein original request ID included.  
3. Further processors debit/credit streams consume karte hain, request ID se deduplicate karte hain, aur account balances mein changes apply karte hain.  

**No distributed transaction needed** — request durably single message ke roop mein logged hota hai pehle, phir derived instructions emit kiye jaate hain. Agar stream processor crash ho jaaye, to wo apne last checkpoint se resume karta hai aur messages re-emit kar sakta hai, lekin step 3 mein deduplication double-processing prevent karta hai.

### 3.3 Timeliness and Integrity  
"Consistency" term do alag requirements ko conflate karta hai:

|                      | Timeliness                                                   | Integrity                                                    |
|----------------------|--------------------------------------------------------------|--------------------------------------------------------------|
| **Meaning**          | Users system ko up-to-date state mein observe karte hain      | No corruption, no contradictory or false data                 |
| **Violation**        | "Eventual consistency" — temporary, wait karne se resolve     | "Perpetual inconsistency" — permanent, explicit repair chahiye |
| **Importance**       | Annoying agar violate ho                                      | Catastrophic agar violate ho                                  |
| **Example**          | Credit card transaction abhi statement par nahi dikha (normal) | Statement balance ≠ sum of transactions (very bad)            |

ACID transactions dono provide karte hain: timeliness (linearizability) aur integrity (atomic commit). Event-based dataflow systems inhe decouple karte hain:

- **No guarantee of timeliness** (asynchronous by design), unless explicitly built.  
- **Integrity** achieve ki ja sakti hai through:  
  - Writes ko single immutable message ke roop mein represent karna (event sourcing).  
  - Deterministic derivation functions for all other state updates.  
  - End-to-end operation identifiers for duplicate suppression aur idempotence.  
  - Immutable messages allowing reprocessing to recover from bugs.

**Loosely Interpreted Constraints**  
Bahut saari real applications uniqueness ke weaker notions ke saath kaam chala sakti hain:

- Do log same username register karein → ek ko apology bhejo, doosra choose karne ko kaho (compensating transaction).  
- Stock se zyada items order ho jaayein → more stock order karo, delay ke liye apologize karo, discount offer karo.  
- Airlines routinely flights overbook karte hain — compensation processes excess demand handle karte hain.  
- Bank overdraft → overdraft fee charge karo.  

Apology ka cost business decision hai. Agar acceptable ho, to aap optimistically likh sakte ho aur constraints ko after the fact check kar sakte ho — linearizable constraints ki zaroorat nahi.

**Coordination-Avoiding Data Systems**  
Yeh observations batate hain ki dataflow systems bina coordination require kiye strong integrity guarantees provide kar sakte hain, better performance aur fault tolerance achieve karte hue:

- Multiple datacenters ke across multi-leader configuration mein asynchronous replication ke saath operate kar sakte hain.  
- Weak timeliness (coordination ke bina linearizable nahi) lekin strong integrity.  
- Serializable transactions ab bhi useful hain small scope par derived state maintain karne ke liye.  
- Synchronous coordination sirf wahan introduce karo jahan strictly needed ho (e.g., irrecoverable operation se pehle).

### 3.4 Trust, but Verify  
Data disks par, networks mein (TCP checksums evade karte hue), aur software bugs (even mature databases jaise MySQL aur PostgreSQL) ke through corrupt ho sakta hai.

**Auditing**  
Data ki integrity checking. HDFS aur Amazon S3 background processes chalate hain jo continuously files read back karte hain aur replicas se compare karte hain.  
Apne **backups test karo** — sirf trust mat karo ki wo kaam karte hain.

**Designing for Auditability**  
Event-based systems better auditability provide karte hain: user input single immutable event ke roop mein represent hota hai, state updates deterministically derived hote hain.  
Explicit dataflow ke saath data ki **provenance** much clearer hoti hai.  
Batch/stream processors rerun karke verify kar sakte ho ki derived state expectations match karta hai.  
Continuous end-to-end integrity checks confidence badhate hain aur faster iteration allow karte hain.

**Cryptographic Tools**  
Blockchains aur distributed ledgers (Bitcoin, Ethereum, etc.) cryptographic proofs of integrity explore karte hain.  
Certificate transparency aur Merkle trees prove kar sakte hain ki log tampered nahi hua.  
In algorithms ko general data systems ke liye scalable banana ongoing research ka area hai.

---

Section 4: Doing the Right Thing  
Har system kisi purpose ke liye banta hai, lekin consequences us purpose se kahin aage tak pahunchte hain. Bahut saare datasets logon ke baare mein hain — unka behavior, interests, identity. Humein aise data ke saath humanity aur respect ke saath treat karna chahiye. Software development increasingly involves making important ethical choices.

**Predictive Analytics**  
Data use karna weather ya disease spread predict karne ke liye ek baat hai; predicting whether a convict will reoffend, a loan applicant will default, ya an insurance customer will make expensive claims — yeh directly individual people's lives ko affect karta hai.  
Kisi algorithm ne jise (accurately ya falsely) risky label kiya hai, wo large number of "no" decisions suffer kar sakta hai. Systematically jobs, air travel, insurance, property rental, aur financial services se excluded hona **"algorithmic prison"** kehlaaya gaya hai — without proof of guilt and with little chance of appeal.

**Bias and Discrimination**  
Historical data par trained algorithms existing biases learn aur systematize kar sakte hain. Chahe explicitly protected characteristics use na karein, wo proxies use kar sakte hain (e.g., racially segregated neighborhoods mein zip code ya IP address race ka proxy).  
*"Machine learning is like money laundering for bias"* — algorithm biased data input leta hai aur output produce karta hai jo objective lagta hai lekin discrimination codify karta hai.  
Predictive analytics sirf past se extrapolate karte hain; agar past discriminatory hai, to wo discrimination codify karte hain. Agar hum future better chahte hain, to **moral imagination** chahiye — something only humans can provide.

**Responsibility and Accountability**  
Jab algorithms mistakes karte hain, to accountable kaun hai? Agar self-driving car accident cause kare, to responsible kaun? Agar automated credit scoring algorithm systematically discriminate kare, to recourse hai kya?  
Credit score summarize karta hai *"How did you behave in the past?"* jabki predictive analytics kaam karta hai *"Who is similar to you, and how did people like you behave?"* — yeh implies **stereotyping people**.  
Bahut saara data statistical hai: even if probability distribution overall correct ho, individual cases well wrong ho sakte hain.

**Feedback Loops**  
Predictive systems self-reinforcing cycles create kar sakte hain: bad credit score → can't find work → worse credit score → even harder to find work. A downward spiral due to poisonous assumptions, hidden behind a camouflage of mathematical rigor.  
Recommendation systems logon ko sirf wahi opinions dikha sakte hain jisse wo already agree karte hain, leading to **echo chambers** jahan stereotypes, misinformation, aur polarization breed karte hain.

**Privacy and Tracking**  

**Surveillance**  
As a thought experiment, "data" ko "surveillance" se replace karo: *"In our surveillance-driven organization we collect real-time surveillance streams and store them in our surveillance warehouse. Our surveillance scientists use advanced analytics and surveillance processing in order to derive new insights."*  
Humne duniya ka sabse bada mass surveillance infrastructure build kar diya hai. Even the most totalitarian regimes sirf dream kar sakte the har room mein microphone lagane ka aur har person ko location-tracking device carry karne ko force karne ka — yet we voluntarily throw ourselves into this world. Farak sirf itna hai ki data corporations collect karti hain rather than government agencies.

**Consent and Freedom of Choice**  
Users ko bahut kam knowledge hoti hai ki wo databases mein kya data feed kar rahe hain ya use kaise process kiya jaata hai. Zyadatar privacy policies obscure karne ke liye zyada kaam karti hain, illuminate karne ke liye kam.  
Jo user surveillance ko consent nahi karta, uske liye ek hi alternative hai: service use mat karo. Lekin agar service *"regarded by most people as essential for basic social participation"* hai, to opting out real choice nahi hai — surveillance inescapable ho jaata hai.

**Privacy and Use of Data**  
Privacy ka matlab sab kuch secret rakhna nahi hai; iska matlab hai **freedom to choose which things to reveal to whom**. Yeh ek decision right hai, autonomy ka aspect.  
Jab data surveillance infrastructure ke through extract kiya jaata hai, to privacy rights individual se data collector ke paas transfer ho jaate hain. Companies outcome ka bahut hissa secret rakhti hain kyunki reveal karna creepy perceived hota.

**Data as Assets and Power**  
Behavioral data ko kabhi **"data exhaust"** kaha jaata hai — lekin economic point of view se, agar targeted advertising service ke liye pay karta hai, to behavioral data service ka **core asset** hai. Application sirf ek means hai users ko lure karne ka taaki wo personal information surveillance infrastructure mein feed karein.  
**Data brokers** — ek shady industry jo secrecy mein operate karti hai, intrusive personal data purchase, aggregate, analyze, aur resell karti hai, mostly marketing purposes ke liye.  
Data sirf asset nahi balki **"toxic asset"** hai (Bruce Schneier) — jab bhi hum data collect karte hain, benefits ko risk ke saath balance karna padta hai ki wo galat haathon mein padega (breaches, hostile governments, unscrupulous management).  
*"It is poor civic hygiene to install technologies that could someday facilitate a police state."*

**The Way Forward**  
Data is the pollution problem of the information age (Bruce Schneier). Protecting privacy is the environmental challenge. Jaise Industrial Revolution ka dark side tha (pollution, exploitation) jise regulation ki zaroorat padi, humara transition to the information age problems laata hai jinhe humein confront karna hoga.  
Humein culture shift chahiye: users ko metrics ki tarah dekhna band karo jinko optimize karna hai. Yaad rakho wo humans hain jo respect, dignity, aur agency deserve karte hain.  
Data collection practices self-regulate karo. End users ko educate karo ki unka data kaise use hota hai.  
Data forever retain mat karo — purge karo jab needed na ho.  
Access control cryptographic protocols se enforce karo, not merely by policy.  
Sirf aaj ka political environment nahi, balki **all possible future governments** consider karo jo data access kar sakte hain.

---

Summary  
**Key Takeaways**  

**Data Integration:**  
- No single tool satisfies all needs. Specialized tools combine karo with clear dataflow reasoning.  
- **Log-based derived data** (CDC, event sourcing) heterogeneous systems integrate karne ke liye distributed transactions se zyada robust hai.  
- Batch aur stream processing converge ho rahe hain. Lambda architecture unified systems se superseded ho chuka hai.  

**Unbundling Databases:**  
- Batch/stream processors triggers, stored procedures, aur materialized view maintenance ki tarah hain — lekin across different systems.  
- Federated databases **reads unify** karte hain; unbundled databases **writes unify** karte hain (via asynchronous event logs with idempotent consumers).  
- **"Database inside-out" approach**: application code as derivation functions, stream processors as services, write path vs read path.  
- Write path ko end-user devices tak extend karo (server-sent events, WebSockets). Reads are events too.  

**Aiming for Correctness:**  
- **End-to-end argument**: low-level reliability features (TCP, transactions) necessary hain lekin sufficient nahi. Applications ko end-to-end measures chahiye (operation identifiers, idempotence).  
- Uniqueness constraints log-based messaging se enforce kiye ja sakte hain (sequential processing per partition).  
- Multi-partition operations without atomic commit: request ko single message ki tarah log karo, instructions derive karo, downstream deduplicate karo.  
- **Timeliness** (eventual consistency) vs **Integrity** (perpetual inconsistency) — integrity is much more important.  
- Coordination-avoiding data systems: strong integrity without coordination, weak timeliness. Coordination sirf wahan jahan strictly needed.  
- Auditability ke liye design karo: immutable events, deterministic derivations, end-to-end integrity checks.  

**Doing the Right Thing:**  
- Data about people ke saath humanity aur respect se treat karo.  
- Bias, feedback loops, surveillance, aur consent ki limits ke baare mein aware raho.  
- Data ek **toxic asset** hai — benefits aur risks balance karo. Privacy fundamental right hai.
