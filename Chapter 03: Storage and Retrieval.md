Sabse basic level par, ek database ko do kaam karne hote hain: jab aap use kuch data dein, to wo data store kare, aur jab aap baad mein dobara maangein, to wo data aapko wapas de de.

Ek application developer hone ke naate, aapko apne application ke liye appropriate storage engine choose karna hota hai. Apne workload par accha perform karne ke liye storage engine ko tune karne ke liye, aapko ek rough idea hona chahiye ki under the hood kya ho raha hai.

Transactional workloads ke liye optimized storage engines aur analytics ke liye optimized storage engines mein bada farak hota hai. Do families of storage engines:

- **Log-structured storage engines**
- **Page-oriented storage engines** (jaise B-trees)

Section 1: Data Structures That Power Your Database  
**The World's Simplest Database**  
Ek key-value store imagine karo jo do Bash functions se implement ho:

```bash
#!/bin/bash
db_set () {
    echo "$1,$2" >> database
}
db_get () {
    grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

- `db_set` file ke end mein append karta hai — bahut efficient (O(1) writes).
- `db_get` poori file scan karta hai — performance bahut kharab (O(n) reads).

Bahut saare databases internally ek **log** use karte hain: ek append-only sequence of records (zaroori nahi ki human-readable ho).

**The Need for Indexes**  
Kisi particular key ki value efficiently dhundhne ke liye, humein ek **index** ki zaroorat hoti hai — additional metadata jo signpost ka kaam karta hai.  
Index primary data se derive kiya gaya ek additional structure hai. Isse database ke contents affect nahi hote, sirf queries ki performance affect hoti hai.  
Important trade-off: Acche se chosen indexes read queries ko fast karte hain, lekin **har index writes ko slow kar deta hai** (kyunki har write par index update karna padta hai).  
Databases aam taur par default mein sab kuch index nahi karte — aap manually indexes choose karte ho apni application ke typical query patterns ke hisaab se.

---

Section 2: Hash Indexes  
Simplest indexing strategy: ek **in-memory hash map** rakho jismein har key ko data file mein ek byte offset se map kiya jaata hai — woh location jahan value mil sakti hai.
<img width="1004" height="514" alt="image" src="https://github.com/user-attachments/assets/27f31953-af35-4fe4-9137-e8c9f6b09ead" />

**Figure 3-1: Storing a log of key-value pairs in a CSV-like format, indexed with an in-memory hash map**  
(Jaisa aap image mein dekh sakte ho: ek taraf in-memory hash map hai jismein key aur byte offset stored hai, aur doosri taraf disk par log-structured file hai.)

Jab aap naya key-value pair append karte ho, hash map update karo taaki naye data ka offset reflect ho.  
Value lookup karne ke liye, hash map se offset dhundho, wahan seek karo, aur value padho.

**Bitcask**  
Yahi essentially Bitcask (Riak ka default storage engine) karta hai:

- High-performance reads aur writes provide karta hai.
- Requirement: saari keys available RAM mein fit honi chahiye (hash map poori tarah memory mein hota hai).
- Values available memory se zyada space use kar sakti hain — disk se sirf ek seek ke saath load hoti hain.
- Un workloads ke liye accha hai jahan har key ki value frequently update hoti hai (e.g., cat video ke liye URL → play count). Bahut saare writes per key, lekin saari keys memory mein rakhna feasible hai.

**Compaction aur Segment Merging**  
Append-only log se disk space khatam hone se kaise bachein:

- **Log ko segments mein tod do** — Ek segment file close kar do jab wo certain size par pahunch jaaye, naye data ko naye segment mein likho.
- **Compaction** — Log mein se duplicate keys hata do, har key ki sirf most recent update rakho.
<img width="2880" height="1047" alt="image" src="https://github.com/user-attachments/assets/7ef5ac08-ca02-441e-a74c-e3c4109a53e5" />

**Figure 3-2: Compaction of a key-value update log, retaining only the most recent value for each key**  
(Image mein dikhaya hai ki kaise purane duplicate values hata kar sirf latest value rakhi jaati hai.)

Compaction karte waqt segments ko merge bhi karo. Segments likhe jaane ke baad kabhi modify nahi kiye jaate — merged segment naye file mein likha jaata hai. Merging background thread mein hoti hai; reads aur writes purane segments use karte rahte hain jab tak merging complete na ho jaaye, phir purane segments delete kar diye jaate hain.
<img width="1016" height="534" alt="image" src="https://github.com/user-attachments/assets/0125a6fa-e079-4d12-9b48-ad822133dc30" />

**Figure 3-3: Performing compaction and segment merging simultaneously**  
(Yahan dikhaya hai ki do segments ko merge karke ek naya compacted segment ban raha hai.)

Har segment ka apna in-memory hash table hota hai jo keys ko file offsets se map karta hai.  
Key dhundhne ke liye: sabse recent segment ka hash map check karo, phir second-most-recent, aur aise karte jaao.  
Merging segments ki sankhya kam rakhti hai, taaki lookups mein zyada hash maps check na karne padein.

**Implementation Details**  
- **File format** — Binary format CSV se fast aur simple hota hai (string length bytes mein encode karo, phir raw string).
- **Deleting records** — Ek special deletion record append karo jise **tombstone** kehte hain. Merging ke dauran, tombstone process ko batata hai ki us key ke previous values discard kar do.
- **Crash recovery** — In-memory hash maps restart par lost ho jaate hain. Poori segment files padhkar restore kar sakte ho (slow), ya har segment ke hash map ka snapshot disk par store kar sakte ho (Bitcask ka approach) faster recovery ke liye.
- **Partially written records** — Checksums use karo taaki log ke corrupted parts detect aur ignore kiye ja sakein.
- **Concurrency control** — Sirf ek writer thread (strictly sequential appends). Segment files append-only aur immutable hain, isliye multiple threads unhe concurrently padh sakti hain.

**Append-Only Kyun?**  
- Sequential writes random writes se kahin zyada fast hote hain, khaas kar magnetic spinning-disk hard drives par. SSDs par bhi preferable hai.
- Immutable segment files ke saath concurrency aur crash recovery bahut simple ho jaati hai.
- Purane segments ko merge karte rehne se data file fragmentation time ke saath nahi hoti.

**Hash Table Index Ki Limitations**  
- Hash table memory mein fit hona chahiye. On-disk hash map performant banana mushkil hai (bahut saara random I/O, grow karna expensive, hash collisions ke liye fiddly logic).
- Range queries efficient nahi hain — aap aasani se `kitty00000` se `kitty99999` ke beech ki saari keys scan nahi kar sakte; har key individually lookup karni padegi.

---

Section 3: SSTables and LSM-Trees  
**Sorted String Tables (SSTables)**  
Ek simple lekin powerful change: yeh require karo ki key-value pairs ka sequence **key ke hisaab se sorted** ho. Aur yeh bhi require karo ki har merged segment file mein har key sirf ek baar aaye (compaction ensure karta hai).

**Hash Indexes Ke Muqable Fayde**  
1. **Efficient Merging** — Mergesort algorithm ki tarah: input files ko side by side padho, lowest key ko output file mein copy karo, repeat. Jab same key multiple segments mein aaye, to most recent segment wali value rakho.
<img width="2880" height="1920" alt="image" src="https://github.com/user-attachments/assets/14089bc0-c382-4853-8a2f-4076e7d07461" />

**Figure 3-4: Merging several SSTable segments, retaining only the most recent value for each key**  
(Image mein dikh raha hai kaise sorted segments ko merge karte hain aur latest value rakhte hain.)

2. **Sparse In-Memory Index** — Ab aapko saari keys ka index memory mein rakhne ki zaroorat nahi hai. Kyunki keys sorted hain, aap known offset par jump kar sakte ho aur wahan se scan kar sakte ho. Ek **sparse index** (har kuch kilobytes ke segment file ke liye ek key) kaafi hota hai.
<img width="2880" height="1416" alt="image" src="https://github.com/user-attachments/assets/1d45e70e-db83-4b82-9d9d-4c8906232766" />

**Figure 3-5: An SSTable with an in-memory index**  
(Yahan memory mein sparse index hai jo kuch keys ke offsets batata hai, aur disk par sorted segment file hai.)

3. **Block Compression** — Read requests range mein kai key-value pairs scan karte hain, isliye records ko block mein group karke disk par likhne se pehle compress kiya ja sakta hai. Sparse index ka har entry compressed block ke start ko point karta hai. Disk space bachata hai aur I/O bandwidth kam karta hai.

**SSTables Construct aur Maintain Karna**  
Incoming writes kisi bhi order mein aa sakte hain, to data sorted kaise karein:

- **Memtable** — Jab write aaye, use in-memory balanced tree data structure (jaise red-black tree ya AVL tree) mein add karo. Is in-memory tree ko **memtable** kehte hain.
- Jab memtable ek threshold (typically kuch megabytes) se zyada ho jaaye, use disk par SSTable file ke roop mein likh do (efficient kyunki tree already sorted order maintain karta hai). Yeh sabse recent segment ban jaata hai. Writes naye memtable instance mein continue hoti hain.
- Read request serve karne ke liye: pehle memtable check karo, phir most recent on-disk segment, phir next-older, etc.
- Background mein periodically merging aur compaction process run karo taaki segment files combine ho jaayein aur overwritten/deleted values discard ho jaayein.

**Crash Recovery**  
Agar database crash ho jaaye, to memtable mein most recent writes (jo abhi disk par nahi likhi gayi) lost ho jaati hain.  
Solution: disk par ek alag **write-ahead log (WAL)** rakho jismein har write immediately append ho (sorted nahi — sirf crash recovery ke liye).  
Jab bhi memtable SSTable mein likha jaata hai, corresponding log discard kiya ja sakta hai.

**SSTables se LSM-Tree Banana**  
Yeh algorithm essentially un systems mein use hota hai jaise:

- **LevelDB aur RocksDB** — Key-value storage engine libraries jo doosre applications mein embed hone ke liye design hain.
- **Cassandra aur HBase** — Dono Google ki Bigtable paper se inspire hain (jisne SSTable aur memtable terms introduce kiye).
- **Lucene** (Elasticsearch aur Solr dwara use) — apne term dictionary ke liye similar method use karta hai (key = term, value = list of document IDs containing that term, yani postings list).

Originally Patrick O'Neil et al. ne ise **Log-Structured Merge-Tree (LSM-Tree)** ke naam se describe kiya tha. Storage engines jo sorted files ko merge aur compact karne ke is principle par based hain, unhe **LSM storage engines** kehte hain.

**Performance Optimizations**  
- **Bloom filters** — Ek memory-efficient data structure jo set ke contents ko approximate karta hai. Bata sakta hai ki koi key database mein nahi hai, jisse nonexistent keys ke liye unnecessary disk reads bach jaate hain. (LSM-tree mein nonexistent keys ke lookups bina iske slow hote hain — memtable + saare segments check karne padte hain.)
- **Compaction strategies**:
  - **Size-tiered compaction** — Newer aur smaller SSTables successively older aur larger SSTables mein merge kiye jaate hain. HBase use karta hai; Cassandra supported hai.
  - **Leveled compaction** — Key range ko chhote SSTables mein split kiya jaata hai, older data alag "levels" mein move hota hai, jisse compaction zyada incrementally proceed kar sakta hai aur disk space kam lagta hai. LevelDB aur RocksDB use karte hain.

**LSM-Trees ki Key Properties**:
- Data sorted order mein store hota hai → efficient range queries.
- Disk writes sequential hote hain → remarkably high write throughput.
- Tab bhi accha kaam karta hai jab dataset available memory se bahut bada ho.

---

Section 4: B-Trees  
Sabse widely used indexing structure. 1970 mein introduce hua aur 10 saal se bhi kam mein "ubiquitous" kehlaaya. Lagbhag sabhi relational databases mein standard index implementation hai, aur bahut saari nonrelational databases mein bhi.

SSTables ki tarah, B-trees bhi key-value pairs ko key ke hisaab se sorted rakhte hain, jisse efficient key-value lookups aur range queries possible hain. Lekin design philosophy bahut alag hai.

**Key Differences from Log-Structured Indexes**  

| Log-Structured (LSM) | B-Trees |
|----------------------|---------|
| **Unit** | Variable-size segments (kuch MB se zyada) | Fixed-size blocks/pages (traditionally 4 KB) |
| **Write pattern** | Hamesha sequentially likho | Page ko disk par in-place overwrite karo |
| **Organization** | Append-only files | Pages ka tree with references |

**B-Trees Kaise Kaam Karte Hain**  
Database fixed-size pages (traditionally 4 KB, kabhi kabhi bade) mein toota hota hai, ek time mein ek page padha ya likha jaata hai. Yeh underlying hardware ke closely correspond karta hai (disks bhi fixed-size blocks mein arrange hote hain).  
Har page ek address ya location se identify hota hai (jaise pointer, lekin disk par memory ki jagah). Pages doosre pages ko refer karte hain, pages ka ek tree construct karte hain.
<img width="2880" height="1596" alt="image" src="https://github.com/user-attachments/assets/992b2bf6-0849-47e7-ad74-82dab676590c" />

**Figure 3-6: Looking up a key using a B-tree index**  
(Yeh image dikhata hai kaise root page se start karke child pages ke through traverse karte hain.)

Ek page **root** hota hai. Key lookup ke liye root se start karo.  
Har page mein kuch keys aur child pages ke references hote hain. Har child ek continuous range of keys ke liye responsible hota hai, aur references ke beech ki keys un ranges ki boundaries batati hain.  
Akhir mein aap **leaf page** par pahunchte ho jismein individual keys hoti hain, jo ya to value inline contain karta hai ya un pages ke references jahan values mil sakti hain.

Ek page mein child pages ke references ki number ko **branching factor** kehte hain (typically kuch sau).  

**Capacity**  
4 KB pages aur 500 branching factor wala four-level tree 256 TB tak store kar sakta hai.

*n* keys wale B-tree ki depth hamesha **O(log n)** hoti hai — zyadatar databases teen ya chaar levels deep B-tree mein fit ho jaate hain.

**Updates aur Inserts**  
- **Update**: Leaf page dhundho jismein key ho, value change karo, page wapas disk par likho. Us page ke saare references valid rehte hain.
- **Insert**: Wo page dhundho jiski range mein nayi key aati hai aur use add karo. Agar free space nahi hai, to page **split** ho jaata hai do half-full pages mein, aur parent page update hota hai naye subdivision ke hisaab se.
<img width="2880" height="1618" alt="image" src="https://github.com/user-attachments/assets/b229dd6c-5bbf-4cac-a8d3-07fe71910825" />

**Figure 3-7: Growing a B-tree by splitting a page**  
(Image mein page split dikhaya gaya hai.)

Yeh algorithm ensure karta hai ki tree balanced rahe.

**Making B-Trees Reliable**  
**Write-Ahead Log (WAL)**  
Kisi page ko overwrite karna dangerous operation hai. Kuch operations ke liye kai pages overwrite karne padte hain (e.g., page split ke liye do split pages likhne + parent update karna). Agar database mid-operation crash ho jaaye, to aapko corrupted index milta hai (e.g., orphan page).  
Solution: **write-ahead log (WAL)**, jise **redo log** bhi kehte hain — ek append-only file jismein har B-tree modification ko tree pages par apply karne se pehle likhna zaroori hai. Crash ke baad B-tree ko consistent state mein restore karne ke liye use hota hai.

**Concurrency Control**  
Pages ko in-place update karte waqt careful concurrency control ki zaroorat hoti hai agar multiple threads simultaneously B-tree access karein.  
Typically **latches** (lightweight locks) se tree ke data structures protect kiye jaate hain.  
Log-structured approaches simple hote hain: merging background mein hoti hai bina incoming queries ko interfere kiye, aur old segments atomically swap kiye jaate hain naye segments se.

**B-Tree Optimizations**  
- **Copy-on-write** — Pages overwrite karne + WAL ki jagah, modified pages ko alag location par likho aur naye parent page versions banao jo nayi location point karein (LMDB use karta hai). Concurrency control (snapshot isolation) ke liye bhi useful hai.
- **Key abbreviation** — Interior pages mein poori key store nahi karte, sirf utna hissa jo boundaries act kar sake. Higher branching factor aur kam levels allow karta hai (kabhi **B+ tree** kehlata hai).
- **Sequential leaf layout** — Leaf pages ko disk par sequential order mein rakhne ki koshish karo taaki range scans efficient ho. Tree grow karne ke saath maintain karna mushkil hai. LSM-trees ka yahan fayda hai kyunki wo merging ke dauran bade segments ek saath rewrite karte hain.
- **Sibling pointers** — Har leaf page ke paas apne left aur right sibling pages ke references hote hain, jisse keys ko order mein scan karte waqt parent pages par wapas nahi jaana padta.
- **Fractal trees** — B-tree variants jo log-structured ideas borrow karte hain disk seeks kam karne ke liye.

---

Section 5: Comparing B-Trees and LSM-Trees  
Ek aam rule of thumb:

- **LSM-trees typically writes ke liye faster hote hain.**
- **B-trees reads ke liye faster maane jaate hain** (LSM-trees ko kai data structures aur alag-alag stages of compaction mein SSTables check karne padte hain).

Lekin benchmarks aksar inconclusive hote hain aur workload details ke hisaab se sensitive hote hain. Apne particular workload ke saath test karo.

**Advantages of LSM-Trees**  
1. **Lower Write Amplification**  
   - **Write amplification** = database ko ek write karne par uski lifetime mein disk par multiple writes hona.
   - B-trees har piece of data ko kam se kam do baar likhte hain (WAL + tree page), plus overhead of writing entire page chahe sirf kuch bytes change hue hon. Kuch engines page ko do baar overwrite karte hain partially updated pages se bachne ke liye.
   - LSM-trees bhi data ko multiple baar rewrite karte hain (compaction aur merging), lekin kabhi kabhi B-trees se kam write amplification de sakte hain.
   - Write amplification khaas kar SSDs par concern ka vishay hai, jo blocks ko limited number of times overwrite kar sakte hain wear out hone se pehle.

2. **Higher Write Throughput**  
   - LSM-trees sequentially compact SSTable files likhte hain rather than tree mein kai pages overwrite karte hain.
   - Khaas kar magnetic hard drives par important jahan sequential writes random writes se bahut fast hote hain.

3. **Better Compression**  
   - LSM-trees better compress ho sakte hain aur disk par chhoti files produce karte hain.
   - B-trees fragmentation ki wajah se kuch disk space unused chhod dete hain (page splits, rows jo existing pages mein fit nahi hote).
   - LSM-trees periodically SSTables rewrite karte hain fragmentation hatane ke liye → lower storage overheads, khaas kar leveled compaction ke saath.
   - SSDs par: lower write amplification aur reduced fragmentation available I/O bandwidth ke andar zyada read/write requests allow karte hain.

**Downsides of LSM-Trees**  
1. **Compaction Interference**  
   - Compaction ongoing reads aur writes ke saath interfere kar sakti hai. Higher percentiles par response time kaafi high ho sakta hai. B-trees zyada predictable ho sakte hain.

2. **Compaction Can't Keep Up**  
   - High write throughput par, disk bandwidth initial writes aur compaction threads ke beech share hoti hai.
   - Agar compaction incoming writes ke saath pace nahi rakh sakti: unmerged segments ki sankhya badhti hai → disk space bhar jaata hai, reads slow ho jaati hain (zyada segment files check karne padte hain).
   - SSTable-based engines aam taur par incoming writes throttle nahi karte chahe compaction keep up na kar paaye → explicit monitoring ki zaroorat hoti hai.

3. **Key Uniqueness**  
   - B-trees mein, har key index mein exactly ek jagah exist karti hai.
   - LSM-trees mein same key ki multiple copies alag-alag segments mein ho sakti hain.
   - Strong transactional semantics ke liye B-trees attractive hain: transaction isolation keys ke ranges par locks use kar sakta hai, jo directly tree se attached hote hain.

**Bottom Line**  
- B-trees database architecture mein bahut gehrai se base hue hain aur bahut saare workloads ke liye consistently good performance provide karte hain.
- Naye datastores mein, log-structured indexes increasingly popular ho rahe hain.
- Kaunsa better hai iska koi quick rule nahi hai — empirically test karo.

---

Section 6: Other Indexing Structures  
**Secondary Indexes**  
- **Primary key** uniquely ek row/document/vertex ko identify karta hai. **Secondary index** unique nahi hota — ek key ke saath kai rows ho sakti hain.
- Solved by: har value ko matching row identifiers ki list banake (jaise postings list), ya har key ko unique banake row identifier append karke.
- B-trees aur LSM-trees dono secondary indexes ke roop mein use ho sakte hain.
- Joins efficiently perform karne ke liye crucial hain.

**Storing Values Within the Index**  
- **Heap Files**  
  - Index mein value actual row ho sakti hai, ya row ka reference jo kahin aur stored ho.
  - Jahan rows store hoti hain use **heap file** kehte hain (data kisi particular order mein nahi hota).
  - Heap file approach data duplicate hone se bachata hai jab multiple secondary indexes ho — har index sirf heap file mein location reference karta hai.
  - Jab value update karte hain bina key change kiye: efficient agar nayi value badi na ho. Agar badi ho, to record move karna pad sakta hai → ya to saare indexes update karo ya forwarding pointer chhod do.

- **Clustered Index**  
  - Index ke andar hi indexed row store karta hai (extra hop index se heap file tak eliminate karta hai).
  - MySQL ke InnoDB mein, primary key hamesha clustered index hota hai; secondary indexes primary key ko refer karte hain (heap file location nahi).
  - SQL Server mein, aap ek clustered index per table specify kar sakte ho.

- **Covering Index (Index with Included Columns)**  
  - Ek compromise: table ke kuch columns index ke andar store karta hai.
  - Allows some queries to be answered using the index alone (index query ko *cover* kar leta hai).
  - Trade-off: reads fast karta hai, lekin extra storage require karta hai aur writes par overhead add karta hai.

**Multi-Column Indexes**  
- **Concatenated Index**  
  - Kai fields ko ek key mein combine karta hai ek column ko doosre ke saath jodkar.
  - Purane phone book ki tarah: (lastname, firstname) se phone number par index.
  - Kisi particular lastname ke saare log dhundh sakte ho, ya particular lastname-firstname combination. Lekin sirf firstname se dhundhne ke liye useless hai.

- **Multi-Dimensional Indexes**  
  - Geospatial data ke liye important. Example: ek rectangular map area mein saare restaurants dhundho.
    ```sql
    SELECT * FROM restaurants WHERE latitude  > 51.4946 AND latitude  < 51.5079
                                AND longitude > -0.1162 AND longitude < -0.1004;
    ```
  - Standard B-tree ya LSM-tree aapko latitudes ya longitudes ki range mein restaurants de sakta hai, lekin dono simultaneously nahi.
  - Options:
    - Space-filling curve use karke 2D location ko single number mein translate karo, phir regular B-tree index use karo.
    - Specialized spatial indexes jaise **R-trees** (e.g., PostGIS PostgreSQL ke Generalized Search Tree use karke geospatial indexes ko R-trees ke roop mein implement karta hai).
  - Multi-dimensional indexes sirf geography ke liye nahi hain: ecommerce par color ranges (red, green, blue), ya weather observations ke liye (date, temperature). HyperDex use karta hai.

**Full-Text Search and Fuzzy Indexes**  
- Standard indexes exact data maante hain. Fuzzy querying (e.g., misspelled words) alag techniques maangti hai.
- Full-text search engines allow: synonym expansion, grammatical variations ignore karna, words ko aas paas dhundhna, aur doosri linguistic analysis.
- Lucene words ko certain **edit distance** ke andar search kar sakta hai (1 edit distance = ek letter add, remove, ya replace).
- Lucene keys mein characters par **finite state automaton** (trie jaisa) use karta hai as in-memory index, jise **Levenshtein automaton** mein transform kiya ja sakta hai efficient fuzzy search ke liye.

**Keeping Everything in Memory**  
- **In-Memory Databases**  
  - Jaise RAM sasti hoti ja rahi hai, poori datasets ko memory mein rakhna feasible ho gaya hai (possibly distributed across several machines).
  - Kuch sirf caching ke liye hain (e.g., Memcached — restart par data lose hona acceptable hai).
  - Doosre durability ke liye aim karte hain: battery-powered RAM, changes ka log disk par likhna, periodic snapshots, ya doosre machines par replicate karna.
  - Examples: VoltDB, MemSQL, Oracle TimesTen (relational), RAMCloud (key-value with durability), Redis aur Couchbase (weak durability via async disk writes).

- **In-Memory Databases Kyun Fast Hote Hain**  
  - Counterintuitively, isliye nahi ki wo disk se read nahi karte (OS recently used disk blocks ko memory mein cache kar leta hai anyway).
  - Balki, wo in-memory data structures ko disk par likhne layak form mein encode karne ke overheads avoid karte hain.
  - Wo data models bhi provide kar sakte hain jo disk-based indexes ke saath implement karna mushkil ho (e.g., Redis priority queues aur sets offer karta hai).

- **Anti-Caching**  
  - In-memory database architecture jo available memory se bade datasets support karne ke liye extended hai.
  - Least recently used data ko memory se disk par evict karta hai, aur jab dobara access ho to wapas load karta hai.
  - OS virtual memory/swap jaisa, lekin database memory ko zyada efficiently manage karta hai (individual records ke granularity par, poori memory pages nahi).
  - Phir bhi indexes ko poori tarah memory mein fit hona padta hai.

---

Section 7: Transaction Processing or Analytics?  
**OLTP vs OLAP**  

| Property | OLTP (Transaction Processing) | OLAP (Analytic Processing) |
|----------|-------------------------------|----------------------------|
| **Main read pattern** | Small number of records per query, fetched by key | Aggregate over large number of records |
| **Main write pattern** | Random-access, low-latency writes from user input | Bulk import (ETL) ya event stream |
| **Primarily used by** | End user/customer, via web application | Internal analyst, for decision support |
| **What data represents** | Latest state of data (current point in time) | History of events that happened over time |
| **Dataset size** | Gigabytes to terabytes | Terabytes to petabytes |
| **Bottleneck** | Disk seek time | Disk bandwidth |

**Transaction** ka matlab zaroori nahi ki ACID properties hon — iska matlab sirf clients ko low-latency reads aur writes allow karna (batch processing ke opposite).

**Data Warehousing**  
- OLTP systems business operations ke liye critical hote hain → database administrators analysts ko unpar expensive ad hoc queries run karne se hichkichate hain.
- **Data warehouse** ek alag database hai jise analysts query kar sakte hain bina OLTP operations ko affect kiye.
- Company ke sabhi alag-alag OLTP systems ke data ka read-only copy contain karta hai.
- **ETL (Extract–Transform–Load)**  
  Data OLTP databases se extract kiya jaata hai, analysis-friendly schema mein transform kiya jaata hai, clean up kiya jaata hai, aur data warehouse mein load kiya jaata hai.
<img width="2880" height="2047" alt="image" src="https://github.com/user-attachments/assets/e165b185-82ff-4b3b-898f-d17ec3a4f3ae" />

**Figure 3-8: Simplified outline of ETL into a data warehouse**  
(Image mein dikhaya hai ki OLTP systems se data extract karke warehouse mein load hota hai.)

Data warehouse ko analytic access patterns ke liye optimize kiya ja sakta hai (OLTP ke indexing algorithms analytic queries answer karne mein acche nahi hote).  
Surface par, data warehouse aur relational OLTP database similar dikhte hain (dono ke paas SQL interface hota hai), lekin internally kaafi alag hote hain.  
Data warehouse vendors: Teradata, Vertica, SAP HANA, ParAccel (Amazon RedShift hosted version hai). Open source SQL-on-Hadoop: Apache Hive, Spark SQL, Cloudera Impala, Facebook Presto, Apache Drill (kuch Google ke Dremel par based hain).

**Stars and Snowflakes: Schemas for Analytics**  
- **Star Schema (Dimensional Modeling)**  
  - Center mein ek **fact table** hota hai — har row ek event ko represent karta hai jo particular time par hua (e.g., customer ki purchase). Fact tables extremely large ho sakte hain (Apple, Walmart, eBay jaise companies mein tens of petabytes).
  <img width="2628" height="2881" alt="image" src="https://github.com/user-attachments/assets/ca4c703e-83b5-4477-a114-ab0595316271" />

  **Figure 3-9: Example of a star schema for use in a data warehouse**  
  (Image mein fact_sales table aur surrounding dimension tables dikh rahe hain.)
  
  - Kuch columns attributes hote hain (e.g., price, cost).
  - Doosre columns foreign key references hote hain **dimension tables** ki taraf — representing *who, what, where, when, how, why* of the event.
  - Date aur time ko bhi aksar dimension tables se represent kiya jaata hai (extra info jaise public holidays encode karne ke liye).
  - "Star schema" naam visualization se aata hai: fact table beech mein, dimension tables uske around jaise star ki rays.

- **Snowflake Schema**  
  - Ek variation jahan dimensions further subdimensions mein tode jaate hain (e.g., brands aur product categories ke liye alag tables).
  - Star schemas se zyada normalized hai, lekin star schemas aksar prefer kiye jaate hain kyunki analysts ke liye simple hote hain.

**Table Width**  
- Fact tables mein aksar 100 se zyada columns hote hain, kabhi kabhi kuch sau.
- Dimension tables bhi bahut wide ho sakti hain (saara metadata jo analysis ke liye relevant ho).

---

Section 8: Column-Oriented Storage  
Fact tables mein trillions of rows aur petabytes of data ke saath, unhe efficiently store aur query karna challenging problem hai.

- Typical data warehouse query ek time mein sirf 4 ya 5 columns access karta hai (`SELECT *` analytics ke liye rarely needed hota hai).
- **Row-oriented storage** mein (zyadatar OLTP databases), ek row ke saare values aas paas stored hote hain. Query ko saari rows (har ek 100+ attributes ke saath) disk se load karni padti hain, parse karni padti hain, aur filter karna padta hai — bahut slow.

**The Column-Oriented Idea**  
- Ek row ke saare values saath store mat karo — har column ke saare values alag se store karo. Agar har column alag file mein stored hai, to query ko sirf unhi columns ko read aur parse karna padta hai jo query mein use ho rahe hain.
<img width="1010" height="760" alt="image" src="https://github.com/user-attachments/assets/91f191a4-4d34-4c96-93ea-7d91c24d75aa" />

**Figure 3-10: Storing relational data by column, rather than by row**  
(Image mein column-wise storage ka concept dikhaya gaya hai.)

- Column-oriented layout rely karta hai ki har column file mein rows **same order** mein hon. Ek row reassemble karne ke liye, har column file se *k*th entry lo.
- Nonrelational data par bhi equally apply hota hai (e.g., **Parquet** ek columnar storage format hai jo document data model support karta hai, Google ke Dremel par based).

**Column Compression**  
Column-oriented storage compression ke liye bahut suitable hai kyunki column values aksar repetitive hote hain.

- **Bitmap Encoding**  
<img width="1010" height="746" alt="image" src="https://github.com/user-attachments/assets/8b459e42-b9ba-4c1d-8acb-faf1927ed4d7" />

  **Figure 3-11: Compressed, bitmap-indexed storage of a single column**  
  (Image mein dikhaya hai ki column values ko bitmaps mein kaise badalte hain aur run-length encoding se compress karte hain.)
  
  - Ek column jismein *n* distinct values hon, use *n* alag bitmaps mein badal do: har distinct value ke liye ek bitmap, jismein har row ke liye ek bit (1 agar row mein wo value ho, 0 nahi).
  - Agar *n* chhota hai (e.g., ~200 countries), bitmaps ko ek bit per row store kar sakte ho.
  - Agar *n* bada hai (sparse bitmaps), to additionally **run-length encoding** apply karo → remarkably compact.

- **Bitmap Operations for Queries**  
  - `WHERE product_sk IN (30, 68, 69)` → Teen bitmaps load karo, bitwise OR calculate karo.
  - `WHERE product_sk = 31 AND store_sk = 3` → Do bitmaps load karo, bitwise AND calculate karo.
  - Works because columns mein rows same order mein hote hain — ek column ke bitmap ka *k*th bit doosre column ke bitmap ke *k*th bit ki tarah same row ko correspond karta hai.

**Column Families ≠ Column-Oriented**  
Cassandra aur HBase mein **column families** hote hain (Bigtable se inherit), lekin wo har column family ke andar ek row ke saare columns saath store karte hain aur column compression use nahi karte. Bigtable model ab bhi mostly row-oriented hai.

**Memory Bandwidth and Vectorized Processing**  
- Millions of rows scan karne wali queries ke liye, bottlenecks include disk-to-memory bandwidth aur main memory to CPU cache bandwidth.
- Column-oriented storage efficient CPU usage ke liye accha hai: query engine compressed column data ka ek chunk le sakta hai jo CPU ke L1 cache mein fit ho, aur use tight loop mein iterate kar sakta hai (no function calls).
- Operators jaise bitwise AND/OR directly compressed column data par operate kar sakte hain.
- Is technique ko **vectorized processing** kehte hain.

**Sort Order in Column Storage**  
- Data ko sort kiya ja sakta hai (poori row ek saath, chahe column-wise store ho) taaki indexing mechanism ka kaam kare.
- Administrator common queries ke hisaab se sort columns choose karta hai (e.g., `date_key` first sort key, `product_sk` second).
- Sorted order compression mein help karta hai: first sort column mein repeated values ke lambe sequences honge → simple run-length encoding use karke kuch kilobytes mein compress ho sakta hai, chahe billions of rows kyun na ho.
- Compression effect first sort key par strongest hota hai; baad ke columns zyada jumbled hote hain.

**Several Different Sort Orders**  
- C-Store mein introduce hua aur Vertica ne adopt kiya: same data ko kai alag tareekon se sorted store karo (kyunki data waise bhi multiple machines par replicate hota hai).
- Query process karte waqt, wo version use karo jo query pattern ke liye best fit ho.
- Row-oriented stores ke secondary indexes se alag hai: column store mein data kahin aur pointers nahi hote, sirf columns containing values.

**Writing to Column-Oriented Storage**  
- Column-oriented storage, compression, aur sorting reads fast karte hain lekin writes mushkil bana dete hain.
- Compressed columns ke saath update-in-place (jaise B-trees) possible nahi hai.
- Solution: **LSM-trees**. Saare writes pehle in-memory store mein jaate hain (sorted structure), phir jab kaafi writes accumulate ho jaayein to bulk mein column files ke saath disk par merge kar diye jaate hain (essentially Vertica yahi karta hai).
- Queries column data on disk aur recent writes in memory dono examine karte hain, dono ko combine karte hue. Query optimizer user se yeh distinction chhupa leta hai.

**Aggregation: Data Cubes and Materialized Views**  
- **Materialized Views**  
  - Materialized view query results ki actual copy hoti hai jo disk par likhi jaati hai (versus virtual view jo sirf query shortcut hai).
  - Jab underlying data change hota hai, materialized view update karna padta hai → writes expensive ho jaate hain.
  - OLTP databases mein aksar use nahi hota, lekin read-heavy data warehouses mein sense bana sakta hai.

- **Data Cubes (OLAP Cubes)**  
  - Materialized view ka common special case: alag-alag dimensions dwara grouped aggregates ka grid.
  <img width="2880" height="1484" alt="image" src="https://github.com/user-attachments/assets/c8c8e464-1080-4fed-bbda-42c8e9743191" />

  **Figure 3-12: Two dimensions of a data cube, aggregating data by summing**  
  (Image mein product aur date dimensions ke saath cube dikhaya gaya hai.)
  
  - Example: ek axis par dates, doosri par products. Har cell mein us date-product combination ka aggregate (e.g., SUM of net_price) hota hai.
  - Same aggregate har row ya column ke along apply karke summaries pa sakte ho jo ek dimension se reduced ho.
  - Agar 5 dimensions ho (date, product, store, promotion, customer), to yeh five-dimensional hypercube hoga.
  - **Fayda**: Kuch queries bahut fast ho jaati hain (effectively precomputed).
  - **Nuksan**: Raw data query karne jaisi flexibility nahi hoti (e.g., agar price dimension nahi hai to $100 se zyada ke items ki sales proportion calculate nahi kar sakte).
  - Zyadatar data warehouses jitna ho sake utna raw data rakhte hain aur data cubes ko sirf kuch queries ke liye performance boost ke roop mein use karte hain.

---

Summary  
**Key Takeaways**  
Storage engines do broad categories mein fall karte hain:

**1. OLTP (Transaction Processing)**  
- User-facing, huge volume of requests, har request small number of records touch karta hai.
- Application kisi key se records request karta hai; storage engine data dhundhne ke liye index use karta hai.
- Disk seek time aksar bottleneck hota hai.
- Do main schools of thought:
  - **Log-structured school** — Sirf files mein append karta hai, kabhi in-place update nahi karta. Bitcask, SSTables, LSM-trees, LevelDB, Cassandra, HBase, Lucene.
  - **Update-in-place school** — Disk ko fixed-size pages ki tarah treat karta hai jo overwrite kiye ja sakte hain. B-trees (sabhi major relational databases use karte hain).
- Log-structured engines systematically random-access writes ko sequential writes mein badalte hain → higher write throughput.

**2. OLAP (Analytics)**  
- Business analysts use karte hain, queries ka volume bahut kam, lekin har query millions of records scan karta hai.
- Disk bandwidth (seek time nahi) bottleneck hota hai.
- **Column-oriented storage** increasingly popular solution hai: data ko compactly encode karta hai, disk se padhe jaane wale data ko minimize karta hai.
- Techniques: column compression, bitmap encoding, run-length encoding, vectorized processing, sort order optimization, materialized views, data cubes.
