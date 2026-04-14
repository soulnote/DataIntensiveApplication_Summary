Teen alag types ke systems hote hain:

| Type                      | Description                                                                                  | Performance Measure                           |
|---------------------------|----------------------------------------------------------------------------------------------|-----------------------------------------------|
| **Services (online)**     | Request ka wait karta hai, jaldi handle karta hai, response bhejta hai                        | Response time, availability                   |
| **Batch processing (offline)** | Badi matra mein input data leta hai, job run karta hai, output produce karta hai. Koi user wait nahi karta. Aam taur par periodically scheduled hota hai. | Throughput (ek fixed size ka dataset kitni der mein crunch hua) |
| **Stream processing (near-real-time)** | Inputs consume karta hai, outputs produce karta hai (batch ki tarah), lekin events ke turant baad operate karta hai (batch se kam latency) | Latency                                      |

MapReduce, jo 2004 mein publish hua, commodity hardware par processing ke scale mein ek bada kadam tha. Halanki ab iski importance kam ho rahi hai, yeh saaf tasveer deta hai ki batch processing kyun aur kaise useful hai.

---

Section 1: Batch Processing with Unix Tools  
**Simple Log Analysis**  
Unix tools ki madad se website ke paanch sabse popular pages dhundhna:

```bash
cat /var/log/nginx/access.log |
  awk '{print $7}' |    # URL extract karo (7th field)
  sort             |    # URLs ko alphabetically sort karo
  uniq -c          |    # Lagataar duplicates count karo
  sort -r -n       |    # Count ke hisaab se descending sort karo
  head -n 5             # Top 5 lo
```

Yeh gigabytes ke log files ko seconds mein process kar leta hai. Hairani ki baat hai ki `awk`, `sed`, `grep`, `sort`, `uniq`, aur `xargs` ka use karke bahut saare data analyses minutes mein kiye ja sakte hain.

**Sorting vs In-Memory Aggregation**  
- **In-memory hash table** (Ruby/Python script) – Accha kaam karta hai agar working set (distinct URLs) memory mein fit ho.
- **Sorting approach** (Unix sort) – Memory se bade datasets handle karta hai disk par spill karke (mergesort with sequential access patterns). GNU Coreutils `sort` automatically yeh handle karta hai aur multiple CPU cores par parallelize bhi kar deta hai.

---

Section 2: The Unix Philosophy  
Doug McIlroy (Unix pipes ke inventor, 1964): "Humein programs ko garden hose ki tarah jodne ke kuch tareeke hone chahiye."

**Four Principles (1978)**  
1. Har program ko ek kaam acchi tarah karna chahiye. Purane programs ko complicate karne ki jagah naye sire se banao.
2. Har program ke output ka doosre program ka input banna expect karo. Output ko clutter mat karo. Binary input formats se bacho. Interactive input par zor mat do.
3. Software ko jaldi try karne ke liye design aur build karo (kuch hafton mein). Clumsy parts ko phenkne mein hichkichao mat.
4. Programming task ko halka karne ke liye unskilled help ki jagah tools ko preference do.

Yeh aaj ke Agile aur DevOps se remarkably milta-julta lagta hai.

**Uniform Interface**  
Unix mein, interface ek **file** (file descriptor) hai — bytes ka ordered sequence.  
Bahut saari cheezein yeh interface share karti hain: actual files, communication channels (Unix sockets, stdin, stdout), device drivers, TCP connections.  
Convention ke hisaab se, bahut saare Unix programs input ko ASCII text maante hain jismein `\n` record separator hota hai.  
Har record ka parsing thoda vague hai: Unix tools aam taur par line ko whitespace ya tab characters se fields mein todte hain, lekin CSV, pipe-separated, aur doosre encodings bhi use hote hain. Ek fairly simple tool jaise `xargs` ke paas bhi input parse karne ke liye aadha darjan command-line options hote hain.  
ASCII text ka uniform interface zyadatar kaam kar leta hai, lekin exactly beautiful nahi hai: URL extract karne ke liye `{print $7}` likhna bahut readable nahi hai. Ideal world mein yeh `{print $request_url}` ho sakta tha.

**Separation of Logic and Wiring**  
Programs default mein stdin aur stdout use karte hain. Pipes ek process ke stdout ko doosre ke stdin se jodte hain.  
Program nahi jaanta aur nahi care karta ki input kahan se aa raha hai ya output kahan ja raha hai — ek tarah ka **loose coupling / inversion of control**.

**Transparency and Experimentation**  
- Input files ko immutable maana jaata hai — jitni baar chahein commands run karo, input ko nuksan nahi pahunchta.
- Kisi bhi point par output ko `less` mein pipe karke intermediate results inspect kar sakte ho.
- Intermediate output ko file mein likh kar baad ke stages dobara chala sakte ho bina poori pipeline dobara chalaye.

**Sabse badi limitation:** Unix tools sirf ek machine par chalte hain — yahi wahan hai jahan Hadoop aata hai.

---

Section 3: MapReduce and Distributed Filesystems  
MapReduce Unix tools jaisa hai, lekin potentially hazaaron machines par distributed. Unix tools ki tarah, yeh input modify nahi karta aur output produce karne ke alawa koi side effects nahi rakhta.

**HDFS (Hadoop Distributed File System)**  
Google File System (GFS) ka open source reimplementation.  
**Shared-nothing** principle par based hai (koi special hardware nahi, sirf conventional network se jude computers).  
Har machine par ek daemon process chalta hai; central **NameNode** track karta hai ki kaunsi file blocks kahan stored hain.  
File blocks multiple machines par replicate hote hain (ya Reed-Solomon codes jaise erasure coding use karte hain).  
Sabse bade deployments: tens of thousands of machines, hundreds of petabytes.

**MapReduce Job Execution**  
Char steps (Unix pipeline ke analogous):

1. Input files padho, records mein todo (e.g., `\n` separator ki tarah).
2. **Mapper** ko bulao har input record se key aur value extract karne ke liye.
3. Saare key-value pairs ko key ke hisaab se sort karo (MapReduce mein implicit).
4. **Reducer** ko bulao sorted key-value pairs iterate karne ke liye jo same key rakhte hain.

- **Mapper** — Har input record ke liye ek baar bulaya jaata hai. Key-value pairs extract karta hai. Stateless (har record independently handle hota hai).
- **Reducer** — Same key ke saare values receive karta hai. Output records produce karta hai.

**Distributed Execution**  
<img width="2880" height="1920" alt="image" src="https://github.com/user-attachments/assets/f36e02d1-e9ec-4458-b066-ad91e196dd8b" />


**Figure 10-1: A MapReduce job with three mappers and three reducers**  
(Aapki pehli image: Map tasks 1,2,3 HDFS se padhte hain, shuffle ke through reducers tak jaata hai, reducers HDFS mein likhte hain.)

- Input file blocks ke hisaab se partitioned hota hai; har block ko alag map task process karta hai.
- Scheduler har mapper ko usi machine par chalane ki koshish karta hai jahan input file replica stored hai (**computation near the data**).
- Reducer partitioning key ke hash se determine karta hai ki kaunsa reducer kaunsa key-value pair receive karega.
- Mappers sorted output local disk par likhte hain, reducer ke hisaab se partitioned. Reducers in sorted files ko fetch aur merge karte hain. Is process ko **shuffle** kehte hain.
- Reducer output distributed filesystem mein likha jaata hai (replicated).

**Workflows**  
Ek akela MapReduce job limited hota hai. Complex analyses ke liye jobs ko chain karna padta hai — ek ka output agle ka input banta hai.  
Chaining implicitly directory name se hoti hai (Unix ki tarah piped nahi).  
Workflow schedulers: Oozie, Azkaban, Luigi, Airflow, Pinball.  
Higher-level tools: Pig, Hive, Cascading, Crunch, FlumeJava — automatically multi-stage workflows set up karte hain.

### 3.1 Reduce-Side Joins and Grouping  

**Sort-Merge Joins**  
<img width="2880" height="1238" alt="image" src="https://github.com/user-attachments/assets/25be6a25-f972-4bb1-96a7-afb55e6fae8d" />

**Figure 10-2: A join between a log of user activity events and a database of user profiles**  
(Doosri image: User activity events ka log aur user database table dikhaya gaya hai.)
<img width="2880" height="1308" alt="image" src="https://github.com/user-attachments/assets/1dff4806-dcec-4ba2-8ffd-1deeb40af30b" />

**Figure 10-3: A reduce-side sort-merge join on user ID**  
(Teesri image: User activity mapper user ID extract karta hai, user database mapper bhi user ID extract karta hai, reducer join karta hai.)

- Mappers ka ek set activity events se user ID extract karta hai; doosra set user database se user ID extract karta hai.
- MapReduce key se partition karta hai aur sort karta hai → same user ID wale saare activity events aur user record reducer input mein adjacent ho jaate hain.
- **Secondary sort** — Records ko aise arrange karo ki reducer user database record pehle dekhe, phir activity events timestamp order mein.
- Reducer user ki date of birth local variable mein store karta hai, phir activity events iterate karta hai → `(viewed-url, viewer-age-in-years)` pairs output karta hai.
- Ek time mein sirf ek user record memory mein rakhne ki zaroorat hoti hai. Koi network requests nahi chahiye.

**GROUP BY and Sessionization**  
Grouping records by key MapReduce mein joins ki tarah hi kaam karta hai.  
**Sessionization** — Kisi particular user session ke saare activity events collate karna (e.g., A/B testing ke liye). Grouping key ki tarah session cookie ya user ID use karo.

**Handling Skew (Hot Keys)**  
Koi celebrity jiske millions of followers ho → ek reducer significantly zyada records process karega → **hot spot**.  
- Pig ka **skewed join** — Sampling job hot keys identify karta hai. Hot key records random reducers ko bheje jaate hain (deterministic hash nahi). Doosra join input un sabhi reducers ko replicate kiya jaata hai jo us key ko handle kar rahe hain.  
- Crunch ka **sharded join** — Similar lekin hot keys explicitly specified.  
- Hive ka **skewed join** — Hot keys alag files mein stored; map-side join use karta hai.  
- **Two-stage aggregation** — First stage: random reducers subsets aggregate karein. Second stage: first-stage results combine karke final value per key banaayein.

### 3.2 Map-Side Joins  
Reduce-side joins input data ke baare mein koi assumptions nahi banate lekin expensive hote hain (sorting, copying, merging). Agar aap input ke baare mein assumptions bana sakte ho, to map-side joins fast hote hain — no reducers, no sorting.

**Broadcast Hash Joins**  
- Ek input itna chhota ho ki memory mein fit ho jaaye. Har mapper chhote dataset ko in-memory hash table mein load karta hai, phir bade input ko scan karta hai aur har record lookup karta hai.
- Chhota input bade input ke saare partitions ko "broadcast" kiya jaata hai.
- Supported by: Pig ("replicated join"), Hive ("MapJoin"), Cascading, Crunch, Impala.

**Partitioned Hash Joins**  
- Dono inputs same tareeke se partitioned ho (same key, same hash function, same number of partitions).
- Har mapper sirf chhote input ka ek partition apne hash table mein load karta hai.
- Hive mein **bucketed map joins** kehlata hai.

**Map-Side Merge Joins**  
- Dono inputs same tareeke se partitioned ho aur same key se sorted ho.
- Mapper dono input files ko incrementally ascending key order mein padhta hai, same key wale records match karta hai (reducer ke merge operation ki tarah).

### 3.3 The Output of Batch Workflows  

**Building Search Indexes**  
Google ka original MapReduce use. Mappers documents partition karte hain; har reducer apne partition ke liye index build karta hai (term dictionary → postings list). Index files ek baar likhe jaane ke baad immutable hoti hain.  
Puri indexing workflow periodically rerun ki ja sakti hai, ya incrementally index build kiya ja sakta hai (Lucene ka segment merging).

**Key-Value Stores as Batch Process Output**  
Batch job ke andar ek brand-new database build karo aur use files ke roop mein distributed filesystem mein likho. Phir bulk mein read-only servers mein load karo.  
Mappers/reducers se directly database mein mat likho (poor performance, database overwhelm ho sakti hai, all-or-nothing guarantee kho jaati hai).  
Used by: Voldemort, Terrapin, ElephantDB, HBase bulk loading.

**Philosophy of Batch Process Outputs**  
Unix philosophy ki tarah — inputs immutable hain, outputs previous outputs replace karte hain, no side effects:

- **Roll back easily** — Agar buggy code galat output produce kare, to code revert karo aur rerun karo. Purana output ab bhi alag directory mein available hai (human fault tolerance).
- **Minimizing irreversibility** — Agile development ke liye beneficial.
- **Automatic retry** — Failed map/reduce tasks same input par dobara schedule kiye jaate hain (safe kyunki inputs immutable hain, failed output discard ho jaata hai).
- **Reuse** — Same input files alag-alag jobs ke liye use ho sakti hain.
- **Separation of concerns** — Logic wiring se separated (input/output directories).

### 3.4 Comparing Hadoop to Distributed Databases  

|                          | MPP Databases                                                | Hadoop/MapReduce                                             |
|--------------------------|--------------------------------------------------------------|--------------------------------------------------------------|
| **Storage**              | Structured data require karta hai (relational/document model) | Files sirf byte sequences hain — koi bhi data model aur encoding |
| **Processing**           | Sirf SQL queries                                              | Arbitrary code (MapReduce, ML, image analysis, NLP)          |
| **Data modeling**        | Careful up-front modeling required                           | Pehle data dump karo, baad mein processing figure out karo ("data lake", "sushi principle: raw data is better") |
| **Fault handling**       | Node failure par poori query abort; resubmit karo             | Individual failed tasks retry karo; intermediate state disk par likho |
| **Design for**           | Short queries (seconds to minutes)                           | Long jobs likely to experience task failures; shared clusters mein preemption |

**Designing for Frequent Faults**  
Google par, ek ghante chalne wale MapReduce task ke paas ~5% risk hota hai preempt hone ka (higher-priority processes ke liye jagah banane ke liye terminate kiya jaana) — hardware failure rate se ek order of magnitude zyada.  
Google ke paas mixed-use datacenters hain jahan online production services aur offline batch jobs same machines par chalte hain. Har task ke paas resource allocation (CPU, RAM, disk) containers ke through enforced hota hai, aur ek priority. Higher-priority processes zyada cost karte hain; lower-priority tasks terminate kiye ja sakte hain resources free karne ke liye.  
Yeh architecture non-production computing resources ko **overcommitted** karne deta hai — system jaanta hai ki zaroorat padne par wo resources reclaim kar sakta hai. Overcommitting better utilization aur greater efficiency deta hai. Batch jobs effectively "table ke neeche ke scraps" pick karte hain — jo bhi computing resources high-priority processes ke baad bach jaate hain.  
Is preemption rate par, agar ek job mein 100 tasks hain aur har task 10 minute chalta hai, to >50% risk hai ki kam se kam ek task finish hone se pehle terminate ho jaayega.  
MapReduce isliye design kiya gaya hai ki wo frequent unexpected task termination tolerate kar sake — isliye nahi ki hardware unreliable hai, balki isliye ki arbitrarily processes terminate karne ki freedom computing cluster mein better resource utilization enable karti hai (Google ka Borg scheduler).  
Open source schedulers (YARN, Mesos, Kubernetes) mein preemption kam widely used hai, isliye MapReduce ke design decisions wahan kam sense banate hain.

---

Section 4: Beyond MapReduce  
MapReduce robust hai (arbitrarily large data unreliable multi-tenant systems par process karta hai) lekin doosre tools kuch types of processing ke liye orders of magnitude faster hain.

### 4.1 Materialization of Intermediate State  
Har MapReduce job independent hoti hai. Ek job ka output agle job ka input distributed filesystem par files ke through banta hai. Yeh intermediate state poora files mein likha jaata hai — ise **materialization** kehte hain.

Unix pipes, iske vipreet, output ko input mein incrementally stream karte hain small in-memory buffer use karke.

**Downsides of Materialization**  
- Job tabhi start ho sakti hai jab poori preceding job ke saare tasks complete ho jaayein (**straggler tasks** sab kuch slow kar dete hain).
- Mappers aksar redundant hote hain — wo bas wahi padhte hain jo reducer ne abhi likha aur use agle stage ke liye prepare karte hain. Directly chain kiya ja sakta tha.
- Intermediate state HDFS mein kai nodes par replicated hota hai — temporary data ke liye overkill.

### 4.2 Dataflow Engines  
MapReduce ki problems fix karne ke liye: **Spark**, **Tez**, aur **Flink**. Ye poore workflow ko ek hi job ki tarah handle karte hain rather than independent subjobs mein todte hain. Inhe **dataflow engines** kehte hain kyunki yeh explicitly data ke flow ko processing stages ke through model karte hain.

**Operators (Generalized Map and Reduce)**  
Functions jinhe **operators** kehte hain flexible ways mein assemble kiye ja sakte hain (sirf alternating map aur reduce nahi). Operators connect karne ke teen options:

1. **Repartition and sort by key** (MapReduce shuffle ki tarah) — sort-merge joins aur grouping enable karta hai.
2. **Repartition without sorting** — partitioned hash joins ke liye (order irrelevant hai kyunki hash table randomize kar deta hai).
3. **Broadcast** — Same output join operator ke saare partitions ko bhejo (broadcast hash joins ke liye).

**Advantages Over MapReduce**  
- Sorting sirf wahan jahan actually required ho (default mein har stage ke beech nahi).
- Koi unnecessary map tasks nahi (mapper kaam preceding reduce operator mein incorporate ho jaata hai).
- Locality optimizations — Scheduler consuming task ko usi machine par rakh sakta hai jahan producing task hai (network copy ki jagah shared memory buffer).
- Intermediate state memory ya local disk mein rakha jaata hai (HDFS mein replicated nahi).
- Operators input ready hote hi execute hona shuru ho jaate hain (poori preceding stage ke complete hone ka wait nahi).
- JVM reuse — Existing processes naye operators run karte hain (MapReduce ki tarah har task ke liye naya JVM nahi).

Pig, Hive, ya Cascading ke workflows ko simple configuration change se MapReduce se Tez ya Spark par switch kiya ja sakta hai.

### 4.3 Fault Tolerance in Dataflow Engines  
- **MapReduce**: HDFS par intermediate state durable hai → failed tasks restart karna easy.
- **Spark/Flink/Tez**: HDFS par intermediate state likhne se bachte hain → agar machine fail ho jaaye, to doosre available data (prior stage ya original HDFS input) se recompute karo.
- Spark **Resilient Distributed Datasets (RDD)** use karta hai data ki ancestry track karne ke liye (kaunse input partitions aur operators ne use produce kiya).
- Flink operator state ka **checkpointing** use karta hai fault ke baad resume karne ke liye.
- **Determinism matters** — Agar recomputed data original lost data se different ho, to downstream operators contradictions face karte hain. Nondeterministic operators (hash table iteration order, random numbers, system clock) reliable recovery ke liye eliminate karne padte hain.
- Agar intermediate data source data se bahut chhota ho ya computation bahut CPU-intensive ho, to recompute karne se materialize karna sasta pad sakta hai.

### 4.4 Graphs and Iterative Processing  
Graph algorithms (e.g., PageRank) ko tab tak repeat karna padta hai jab tak done na ho — ek single MapReduce pass mein express nahi ho sakte. Iterative approach: har step ke liye batch process chalao, completion condition check karo, repeat. MapReduce iske liye bahut inefficient hai (har iteration poori input padhta hai aur entirely new output produce karta hai, chahe sirf chhota hissa change hua ho).

**The Pregel Model (Bulk Synchronous Parallel — BSP)**  
Implemented by: Apache Giraph, Spark's GraphX, Flink's Gelly.  
- Ek vertex doosre vertex ko "message bhej" sakta hai (typically graph ke edges ke along).  
- Har iteration mein, har vertex ke liye ek function call hota hai, jisme uske liye bheje gaye saare messages pass kiye jaate hain.  
- MapReduce ke vipreet, vertex apna state memory mein yaad rakhta hai ek iteration se agle tak — sirf naye incoming messages process karta hai.  
- Actor model jaisa, lekin fault-tolerant durable state aur fixed-round communication ke saath.  
- Fault tolerance periodic checkpointing of all vertex state ke through. Failure par, last checkpoint par roll back.

**Parallel Execution**  
Vertices vertex IDs ko messages bhejte hain; framework partitioning aur routing handle karta hai.  
Graph algorithms mein aksar bahut saara cross-machine communication overhead hota hai.  
Agar graph single computer ki memory mein fit ho jaata hai, to single-machine algorithm distributed batch process se outperform karega. GraphChi jaise single-machine frameworks bhi disk par fit hone wale graphs handle kar sakte hain.

### 4.5 High-Level APIs and Languages  
Higher-level APIs (Hive, Pig, Cascading, Crunch, Spark, Flink) relational-style building blocks use karte hain: field values par join karna, key se group karna, filter karna, aggregate karna (count, sum).

**The Move Toward Declarative Query Languages**  
- Joins ko relational operators ki tarah specify karne se framework automatically best join algorithm choose kar sakta hai (cost-based query optimizers in Hive, Spark, Flink).  
- Simple filtering/mapping declaratively expressed karne se query optimizer column-oriented storage layouts use kar sakta hai, sirf required columns padh sakta hai, aur vectorized execution use kar sakta hai (tight inner loops friendly to CPU caches).  
- Spark JVM bytecode generate karta hai; Impala LLVM use karta hai inner loops ke liye native code generate karne ke liye.  
- Batch processing frameworks ab MPP databases ki tarah dikhne lage hain (comparable performance) lekin arbitrary code run karne aur arbitrary formats padhne ki flexibility retain karte hain.

**Specialization for Different Domains**  
- Machine learning: Mahout (MapReduce/Spark/Flink par), MADlib (relational MPP database Apache HAWQ ke andar).  
- Spatial algorithms: similarity search ke liye k-nearest neighbors.  
- Genome analysis: approximate string matching.

---

Summary  
**Key Takeaways**  

**Design Principles (Unix se MapReduce tak):**  
- Inputs immutable hain, outputs doosre program ka input banne ke liye intended hain.  
- Complex problems chhote tools compose karke solve hote hain jo "ek kaam acchi tarah karte hain".  
- Unix mein: files aur pipes. MapReduce mein: distributed filesystem. Dataflow engines mein: pipe-like data transport.

**Two Main Problems Solved by Distributed Batch Processing:**  

| Problem            | MapReduce                                                                 | Dataflow Engines (Spark/Tez/Flink)                                    |
|--------------------|---------------------------------------------------------------------------|-----------------------------------------------------------------------|
| **Partitioning**   | Mappers input file blocks se partitioned; output repartitioned, sorted, merged into reducer partitions | Similar partitioning, lekin sorting sirf wahan jahan required         |
| **Fault tolerance** | Frequently disk par likhta hai; individual tasks recover karna easy lekin failure-free case mein slower | Zyadatar memory mein rakhta hai; failure par recompute karta hai (deterministic operators chahiye) |

**Join Algorithms:**  
- **Sort-merge joins (reduce-side)** — Dono inputs mappers se guzarte hain join key extract karte hue. Partitioning, sorting, aur merging same key wale saare records same reducer tak laate hain.  
- **Broadcast hash joins (map-side)** — Chhota input poora hash table mein har mapper mein load hota hai. Bada input scan hota hai aur lookup hota hai.  
- **Partitioned hash joins (map-side)** — Dono inputs same tareeke se partitioned. Har mapper chhote input ka ek partition load karta hai.

**Batch Processing Output:**  
- Search indexes, key-value stores, ML models build karo.  
- Batch job ke andar naye database files build karo → bulk mein read-only servers mein load karo.  
- Mappers/reducers se directly production database mein kabhi mat likho.

**Beyond MapReduce:**  
- Dataflow engines (Spark, Tez, Flink) poore workflows ko ek job ki tarah handle karte hain, intermediate state materialize nahi karte, aur aksar orders of magnitude faster hote hain.  
- Graph processing Pregel/BSP model use karta hai (vertex-centric, message-passing, iterative).  
- High-level declarative APIs with cost-based query optimizers batch processing frameworks ko MPP databases ke kareeb la rahe hain.

**Key distinction:** Batch processing input **bounded** hota hai (known, fixed size). Job eventually complete hoti hai. Stream processing (Chapter 11) **unbounded** input handle karta hai — job kabhi complete nahi hoti.
