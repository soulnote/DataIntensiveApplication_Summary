Data systems ki kadwi sachchai mein, bahut kuch galat ho sakta hai: database software/hardware failure mid-write, application crash mid-operation, network interruptions, concurrent writes ek doosre ko overwrite karna, clients partially updated data padhna, race conditions.

**Transactions** in issues ko simplify karne ka mechanism hain. Ek transaction kai reads aur writes ko ek logical unit mein group karta hai: ya to poori transaction succeed hoti hai (**commit**) ya fail hoti hai (**abort/rollback**). Error handling bahut simple ho jaati hai — partial failure ki chinta nahi karni padti.

Transactions applications ko database access karne ka programming model simple banane ke liye create kiye gaye the. Yeh koi natural law nahi hain — inke advantages aur limitations hain.

---

Section 1: The Slippery Concept of a Transaction  
### 1.1 The Meaning of ACID  
Transactions dwara diye gaye safety guarantees ko **ACID** describe karta hai: Atomicity, Consistency, Isolation, and Durability (1983 mein Härder aur Reuter ne term coin kiya). Practice mein, ek database ka ACID ≠ doosre ka ACID — ACID zyadatar marketing term ban gaya hai.

Jo systems ACID meet nahi karte unhe kabhi **BASE** (Basically Available, Soft state, Eventual consistency) kaha jaata hai — ACID se bhi zyada vague.

**Atomicity**  
ACID mein atomicity concurrency ke baare mein nahi hai (wo isolation hai). Yeh batata hai ki agar client kai writes kare aur beech mein fault aa jaaye to kya hoga.  
Agar transaction complete nahi ho sakti, to use abort kar diya jaata hai aur database ab tak kiye gaye saare writes discard/undo kar deta hai.  
Application sure ho sakti hai ki aborted transaction ne kuch nahi badla, isliye safely retry kar sakta hai.  
Shayad **abortability** better term hota.

**Consistency**  
"Consistency" shabd bahut overloaded hai (kam se kam 4 alag meanings):

- Replica consistency (Chapter 5)
- Consistent hashing (Chapter 6)
- Linearizability in CAP theorem (Chapter 9)
- ACID consistency — Application-specific invariants jo hamesha true hone chahiye (e.g., credits aur debits balance hone chahiye)

ACID consistency application ki property hai, database ki nahi. Application transactions ko sahi se define karta hai taaki invariants preserve ho. Letter C actually ACID mein belong nahi karta.

**Isolation**  
Concurrently execute ho rahi transactions ek doosre se isolated hoti hain — wo ek doosre ke kaam mein dakhal nahi de sakti. Classic formalization hai **serializability**: result waisa hi ho jaise transactions serially (ek ke baad ek) run hue hon.
<img width="2880" height="832" alt="image" src="https://github.com/user-attachments/assets/a8383bb7-a9a0-4eb0-b230-edee1b367c2f" />


**Figure 7-1: A race condition between two clients concurrently incrementing a counter**  
(Aapki pehli image: User 1 aur User 2 dono counter 42 se 43 karte hain, lekin ek ka update kho jaata hai.)

Practice mein, serializable isolation rarely use hoti hai performance penalty ki wajah se. Kuch popular databases (e.g., Oracle 11g) to implement hi nahi karte — Oracle ka "serializable" actually snapshot isolation implement karta hai.

**Durability**  
Ek baar transaction successfully commit ho jaaye, to usne jo bhi data likha wo kabhi nahi bhoola jaayega, chahe hardware fault ya crash ho.

- Single-node: data nonvolatile storage (hard drive/SSD) + write-ahead log mein likha jaata hai.
- Replicated database: data successfully kuch number of nodes par copy ho jaata hai.
- Perfect durability exist nahi karti — alag-alag risk-reduction techniques (disk par likhna, replication, backups) saath mein use karni chahiye.

### 1.2 Single-Object and Multi-Object Operations  
**Single-Object Writes**  
Storage engines single object ke level par atomicity (crash recovery ke liye log) aur isolation (har object par lock) provide karte hain. Kuch databases yeh bhi provide karte hain:

- Atomic increment operations (read-modify-write cycle ki zaroorat nahi)
- Compare-and-set operations (tabhi likho agar value change nahi hui)

Yeh useful hain lekin usual sense mein transactions nahi hain — ek transaction multiple objects par multiple operations group karta hai.

**The Need for Multi-Object Transactions**  
- **Foreign key references** — Jo records ek doosre ko refer karte hain unhe insert karte waqt references valid rakhni padti hain.
- **Denormalized data** — Document databases jo joins nahi karte, denormalization encourage karte hain; denormalized info update karne ke liye kai documents atomically update karne padte hain.
- **Secondary indexes** — Data ke saath consistently update hone chahiye; bina isolation ke, record ek index mein dikhe aur doosre mein nahi.

**Handling Errors and Aborts**  
Transactions ka key feature: abort kiya ja sakta hai aur error par safely retry kiya ja sakta hai. Lekin retrying perfect nahi hai:

- Transaction succeed ho gayi ho lekin network acknowledgment ke dauran fail ho → do baar perform ho gayi (deduplication chahiye).
- Error overload ki wajah se ho → retry situation aur kharab karega (exponential backoff use karo).
- Sirf transient errors (deadlock, network interruption) ke liye retry worth hai, permanent errors (constraint violation) ke liye nahi.
- Database ke bahar side effects (e.g., email bhejna) ho sakte hain chahe transaction abort ho jaaye.

---

Section 2: Weak Isolation Levels  
Concurrency bugs testing se pakadna mushkil hai (sirf unlucky timing par trigger hote hain). Databases concurrency issues chhupane ke liye transaction isolation provide karte hain. Serializable isolation guarantee karta hai ki transactions ka effect waisa hi ho jaise serially run hue hon — lekin iski performance cost hai, isliye bahut saare systems weaker levels use karte hain.

### 2.1 Read Committed  
Sabse basic level. Do guarantees:

1. **No dirty reads** — Sirf committed data dikhta hai.
2. **No dirty writes** — Sirf committed data overwrite hota hai.

**No Dirty Reads**  
<img width="2880" height="832" alt="image" src="https://github.com/user-attachments/assets/52e955ed-b0fe-4017-8bf5-4e6d9a975977" />


**Figure 7-4: No dirty reads: user 2 sees the new value for x only after user 1's transaction has committed**  
(Doosri image: User 1 transaction commit karne se pehle User 2 purani value dekhta hai.)

Dirty reads prevent kyun karein:
- Partially updated state dekhna confusing hai aur galat decisions cause kar sakta hai.
- Agar transaction abort ho jaaye, to aapne aisa data dekh liya jo kabhi committed tha hi nahi (rollback ho gaya).

**No Dirty Writes**  
<img width="2880" height="1301" alt="image" src="https://github.com/user-attachments/assets/a1af1792-f467-4d59-bcf7-247bffc5a9a5" />


**Figure 7-5: With dirty writes, conflicting writes from different transactions can be mixed up**  
(Teesri image: Alice aur Bob same listing update karte hain, buyer aur recipient mismatch.)

Example: Alice aur Bob simultaneously same car khareedte hain. Bina dirty writes prevent kiye, sale Bob ko award ho sakti hai lekin invoice Alice ko bhej di jaaye.

**Note:** Read committed Figure 7-1 wali race condition (do counter increments) prevent nahi karta — wo alag problem hai (lost updates).

**Implementation**  
- **Dirty writes** row-level locks se prevent hote hain: transaction ko object modify karne se pehle lock acquire karna padta hai, jo commit/abort tak hold hota hai.
- **Dirty reads** prevent karne ke liye database old committed value aur new uncommitted value dono yaad rakhta hai. Doosri transactions old value padhti hain jab tak nayi value commit na ho jaaye. (Read locks use karne se performance kharab hoti — ek lambi write transaction saare reads block kar deti.)

Read committed default hai Oracle 11g, PostgreSQL, SQL Server 2012, MemSQL, aur bahut saare aur systems mein.

### 2.2 Snapshot Isolation and Repeatable Read  
Read committed ab bhi concurrency bugs allow karta hai. Example: **read skew** (nonrepeatable read).
<img width="2880" height="1301" alt="image" src="https://github.com/user-attachments/assets/9bf90e5a-3a92-45b7-adc9-5e126a6029f2" />


**Figure 7-6: Read skew: Alice observes the database in an inconsistent state**  
(Chauthi image: Alice ke paas do accounts mein $500-$500 hain, transfer ke beech mein ek account pehle padh leti hai, doosra baad mein, total $900 dikhta hai.)

Alice ke paas $1,000 do accounts mein split hain ($500 each). Ek transaction $100 transfer karta hai. Agar Alice ek account transfer se pehle padhe aur doosra baad mein, to use sirf $900 dikhte hain — $100 "gayab" ho gaya. Read committed ke under yeh acceptable hai (dono values padhne ke time committed thi), lekin problematic hai:

- **Backups** — Writes ke dauran liya gaya backup inconsistencies contain kar sakta hai jo restore par permanent ho jaati hain.
- **Analytic queries aur integrity checks** — Database ke bade hisse scan karte waqt agar alag points in time observe ho to nonsensical results milte hain.

**Snapshot isolation** ise solve karta hai: har transaction database ke consistent snapshot se padhti hai — transaction start hone ke time committed saara data. Chahe baad mein data change ho jaaye, har transaction sirf us point in time ka purana data dekhti hai.

Supported by: PostgreSQL, MySQL (InnoDB), Oracle, SQL Server, aur others.

**Implementation: Multi-Version Concurrency Control (MVCC)**  
<img width="2880" height="2181" alt="image" src="https://github.com/user-attachments/assets/4062eebf-d3b6-4fe6-950d-89d2f154049b" />


**Figure 7-7: Implementing snapshot isolation using multi-version objects**  
(Paanchvi image: Accounts table mein created_by aur deleted_by fields ke saath multiple versions dikhaye gaye hain.)

Key principle: **readers kabhi writers ko block nahi karte, aur writers kabhi readers ko block nahi karte.**  
Database object ke kai alag committed versions rakhta hai (alag-alag in-progress transactions ko alag points in time chahiye ho sakte hain).  
Har transaction ko ek unique, hamesha badhta hua **transaction ID (txid)** milta hai.  
Har row mein `created_by` field (jis txid ne insert kiya) aur `deleted_by` field (jis txid ne delete request kiya, initially empty) hota hai.  
Update internally delete + create mein translate hota hai.  
**Garbage collection** process un rows ko hatata hai jo delete marked hain jab koi transaction unhe access nahi kar sakta.

**Visibility Rules**  
Ek object transaction ke liye visible hai agar:

- Reader ki transaction start hone ke time, object create karne wali transaction commit ho chuki thi.
- Object delete marked nahi hai, ya agar hai to delete request karne wali transaction reader start hone ke time commit nahi hui thi.

Zyada specifically:
- Snapshot time par in-progress transactions ke koi bhi writes ignore hote hain (chahe baad mein commit ho jaayein).
- Aborted transactions ke writes ignore hote hain.
- Later txid wali transactions ke writes ignore hote hain (commit status se farak nahi padta).
- Baaki saare writes visible hain.

**Indexes and Snapshot Isolation**  
- Ek option: index object ke saare versions ko point karta hai; index queries invisible versions filter out karte hain.
- Doosra approach (CouchDB, Datomic, LMDB): append-only/copy-on-write B-trees — pages overwrite nahi karte, naye copies banate hain. Har write transaction ek naya B-tree root create karta hai jo ek consistent snapshot hota hai.

**Repeatable Read and Naming Confusion**  
- SQL standard ke paas snapshot isolation ka concept nahi hai (wo iske invention se pehle ka hai).
- PostgreSQL aur MySQL apne snapshot isolation level ko "repeatable read" kehte hain (standards compliance claim karne ke liye).
- Oracle ise "serializable" kehta hai (chahe wo true serializability se weak ho).
- IBM DB2 "repeatable read" ka matlab serializability ke liye use karta hai.
- Koi nahi jaanta repeatable read ka asli matlab kya hai.

### 2.3 Preventing Lost Updates  
**Lost update problem** read-modify-write cycle mein hota hai: do transactions value padhti hain, modify karti hain, aur wapas likhti hain. Doosra write pehle modification ko include nahi karta — ek modification lost ho jaati hai (later write earlier write ko clobber kar deti hai).

Scenarios: counter increment karna, account balance update karna, JSON list mein element add karna, do users simultaneously wiki page edit karna.

**Solutions**  

1. **Atomic Write Operations (best choice jab applicable ho)**  
   ```sql
   UPDATE counters SET value = value + 1 WHERE key = 'foo';
   ```
   - Implemented by: read karte waqt object par exclusive lock lena (**cursor stability**), ya force karna ki saare atomic operations single thread par execute ho.
   - MongoDB JSON documents ke local modifications ke liye atomic operations provide karta hai; Redis data structures jaise priority queues ke liye.
   - ORM frameworks galti se unsafe read-modify-write cycles ke saath atomic operations bypass kar sakte hain.

2. **Explicit Locking**  
   ```sql
   BEGIN TRANSACTION;
   SELECT * FROM figures
     WHERE name = 'robot' AND game_id = 222
     FOR UPDATE;  -- lock all returned rows
   -- Check whether move is valid, then update
   UPDATE figures SET position = 'c4' WHERE id = 1234;
   COMMIT;
   ```
   - `FOR UPDATE` database ko batata hai ki query se return hue saare rows lock kar do.
   - Useful jab atomic operations kaafi na ho (e.g., game logic jo database query mein express nahi ho sakta).
   - Zaroori lock bhoolna aasan hai → race condition introduce ho jaati hai.

3. **Automatically Detecting Lost Updates**  
   - Read-modify-write cycles ko parallel execute hone do; agar transaction manager lost update detect kare, to abort karo aur retry force karo.
   - PostgreSQL ka repeatable read, Oracle ka serializable, aur SQL Server ka snapshot isolation automatically lost updates detect karte hain.
   - MySQL/InnoDB ka repeatable read lost updates detect nahi karta.
   - Great feature: special application code nahi chahiye, less error-prone.

4. **Compare-and-Set**  
   ```sql
   UPDATE wiki_pages SET content = 'new content'
     WHERE id = 1234 AND content = 'old content';
   ```
   - Update tabhi allow karo agar value last read ke baad change nahi hui. Agar change ho gayi, to update ka koi effect nahi hota → retry.
   - **Caution:** Agar database `WHERE` clause ko purane snapshot se padhne deta hai, to yeh safe nahi ho sakta.

5. **Conflict Resolution and Replication**  
   - Multi-leader ya leaderless replication mein, locks aur compare-and-set apply nahi hote (koi single up-to-date copy nahi).
   - Concurrent writes ko conflicting versions (siblings) create karne do, baad mein resolve aur merge karo.
   - Commutative atomic operations (e.g., counter increment) replicated contexts mein acche kaam karte hain — Riak 2.0 datatypes lost updates across replicas prevent karte hain.
   - **Last Write Wins (LWW)** lost updates ke liye prone hai — unfortunately bahut saari replicated databases mein default hai.

### 2.4 Write Skew and Phantoms  
**Write skew** lost update problem ka generalization hai: do transactions same objects padhti hain, phir jo padha uske based par **different objects** update karti hain.

**Doctor On-Call Example**  
<img width="2880" height="2019" alt="image" src="https://github.com/user-attachments/assets/ceb78aa6-8946-4044-8ee8-863397e3bf8e" />


**Figure 7-8: Example of write skew causing an application bug**  
(Chhathi image: Alice aur Bob dono check karte hain ki 2 doctors on call hain, dono off call ho jaate hain, result 0 doctors on call.)

Hospital requirement: kam se kam ek doctor on call ho. Alice aur Bob dono on call hain, dono ki tabiyat kharab, dono ek saath "go off call" click karte hain. Dono transactions check karti hain: 2 doctors on call → safe to go off call. Dono proceed karti hain. Result: koi doctor on call nahi — requirement violate ho gayi.

**Options for Preventing Write Skew**  
- Atomic single-object operations help nahi karte (multiple objects involved).
- Automatic lost update detection help nahi karta (PostgreSQL, MySQL/InnoDB, Oracle, ya SQL Server ke snapshot isolation automatically detect nahi karte).
- Database constraints kuch cases mein help kar sakte hain, lekin multi-object constraints rarely supported hain.
- **Explicit locking** with `SELECT ... FOR UPDATE` un rows par jin par transaction depend karta hai.
- **True serializable isolation** hi complete solution hai.

**More Examples of Write Skew**  
- **Meeting room booking** — Do users same room same time book karte hain. Snapshot isolation conflict prevent nahi karta.
- **Multiplayer game** — Do players alag figures same position par move karte hain.
- **Claiming a username** — Do users simultaneously same username register karna chahte hain. (Unique constraint ise solve kar deta hai.)
- **Preventing double-spending** — Do spending items concurrently insert ho jo mil kar balance negative kar dein.

**Phantoms Causing Write Skew**  
Pattern:
1. `SELECT` query check karta hai ki kuch requirement satisfied hai ya nahi.
2. Application code result ke based par decide karta hai aage kya karna hai.
3. Ek write (`INSERT`, `UPDATE`, ya `DELETE`) step 2 ki precondition change kar deta hai.

**Phantom** tab hota hai jab ek transaction ka write doosri transaction ke search query ka result change kar de. Snapshot isolation read-only queries mein phantoms avoid karta hai, lekin read-write transactions mein phantoms write skew lead kar sakte hain.

Problem: kuch cases mein (meeting room booking, username claiming), query **absence of rows** check karta hai, aur write row add karta hai. `SELECT FOR UPDATE` un rows par locks attach nahi kar sakta jo abhi exist nahi karte.

**Materializing Conflicts**  
Artificially database mein ek lock object introduce karo. Example: saare possible room/time-slot combinations ki table pehle se create kar lo. Transaction relevant rows ko lock kare (`SELECT FOR UPDATE`) check aur insert karne se pehle.

- Phantom ko concrete rows par lock conflict mein badal deta hai.
- Figure out karna hard aur error-prone hai; concurrency control ka application data model mein leak hona ugly hai.
- Should be considered a **last resort** — serializable isolation is much preferable.

---

Section 3: Serializability  
**Serializable isolation** strongest isolation level hai. Guarantee karta hai ki chahe transactions parallel execute ho, end result waisa hi hai jaise wo ek ke baad ek, serially execute hue hon. Database **saari possible race conditions** prevent karta hai.

Serializability implement karne ki teen techniques:

1. Actual Serial Execution
2. Two-Phase Locking (2PL)
3. Serializable Snapshot Isolation (SSI)

### 3.1 Actual Serial Execution  
Sabse simple tareeka: concurrency hata do — ek time par sirf ek transaction execute karo, serial order mein, single thread par.

**Why This Became Feasible (~2007)**  
- RAM itni sasti ho gayi ki poori active dataset memory mein rakh sakein → transactions bahut fast execute hoti hain (disk I/O ka wait nahi).
- OLTP transactions usually chhoti hoti hain aur small number of reads/writes karti hain. Long-running analytic queries consistent snapshot (snapshot isolation) par serial execution loop ke bahar run ho sakti hain.
- Implemented in: VoltDB/H-Store, Redis, Datomic.

**Stored Procedures**  
Interactive client/server transactions ka zyadatar time application aur database ke beech network communication mein jaata hai. Single-threaded execution ke saath, yeh dreadful throughput hota.

Solution: application poori transaction code database ko pehle se **stored procedure** ke roop mein submit kar de.
<img width="2880" height="1663" alt="image" src="https://github.com/user-attachments/assets/8b9e8d94-2b6d-4cbb-9211-cd1b9bf69aa0" />


**Figure 7-9: The difference between an interactive transaction and a stored procedure**  
(Saatvi image: Interactive mein network hops hain, stored procedure ek call mein poora kaam database ke andar.)

Modern implementations general-purpose languages use karte hain: VoltDB Java/Groovy, Datomic Java/Clojure, Redis Lua.

VoltDB stored procedures replication ke liye use karta hai: same stored procedure har replica par execute hoti hai (deterministic procedures chahiye).

**Partitioning**  
Single-threaded execution throughput ek CPU core tak limit karta hai. Scale karne ke liye: data partition karo taaki har partition ka apna transaction processing thread independently chale.

- **Single-partition transactions** CPU cores ke saath linearly scale hote hain.
- **Cross-partition transactions** ko saare partitions ke across coordination chahiye → vastly slower (VoltDB: ~1,000 cross-partition writes/sec vs much higher single-partition throughput).

**Constraints of Serial Execution**  
- Har transaction chhoti aur fast honi chahiye (ek slow transaction sab kuch stall kar deti hai).
- Active dataset memory mein fit hona chahiye.
- Write throughput itni kam honi chahiye ki single CPU core handle kar sake (ya transactions partitionable hon).
- Cross-partition transactions possible hain lekin limited.

### 3.2 Two-Phase Locking (2PL)  
~30 saal tak serializability ke liye widely used algorithm. (Note: 2PL ≠ 2PC — two-phase locking two-phase commit se completely different hai.)

**How It Works**  
Kai transactions concurrently same object padh sakti hain, lekin jaise hi koi write karna chahe, exclusive access chahiye:

- Agar A ne object padha hai aur B write karna chahta hai → B ko wait karna hoga jab tak A commit/abort na kare.
- Agar A ne object likha hai aur B padhna chahta hai → B ko wait karna hoga jab tak A commit/abort na kare.

Snapshot isolation se key difference: 2PL mein, **writers readers ko block karte hain AUR readers writers ko block karte hain.** (Snapshot isolation: readers kabhi writers ko block nahi karte, writers kabhi readers ko block nahi karte.)

**Implementation: Shared and Exclusive Locks**  
- **Read** → shared mode mein lock acquire karo (multiple transactions simultaneously shared lock hold kar sakte hain; exclusive lock exist kare to wait karna padega).
- **Write** → exclusive mode mein lock acquire karo (koi doosra transaction koi bhi lock hold nahi kar sakta).
- **Read then write** → shared lock ko exclusive mein upgrade karo.
- Locks transaction ke end tak hold kiye jaate hain (commit ya abort). First phase: locks acquire; second phase: saare locks release.

Used by: MySQL (InnoDB) aur SQL Server serializable isolation ke liye; DB2 repeatable read ke liye.

**Deadlocks**  
Transaction A B ke lock ka wait karta hai, B A ke lock ka wait karta hai. Database automatically deadlocks detect karta hai aur ek transaction abort kar deta hai taaki baaki proceed kar sakein. Aborted transaction retry karni padti hai. Deadlocks 2PL ke under weak isolation levels ke muqable bahut zyada frequently occur karte hain.

**Performance**  
Bada nuksan: significantly worse throughput aur response times weak isolation ke muqable.

- Locks acquire/release karne ka overhead.
- Reduced concurrency — agar do transactions koi bhi race condition cause kar sakti hain, ek wait karegi.
- Unstable latencies — ek slow transaction ya jo bahut saare locks acquire kar le, poori system ko grind to halt kar sakti hai.

**Predicate Locks**  
Phantoms prevent karne ke liye, humein aise locks chahiye jo search condition match karne wale objects par apply ho (sirf specific rows nahi). **Predicate lock** un saare objects se belong karta hai jo kuch condition match karte hain:

- Transaction A objects padhta hai jo condition match karte hain → shared predicate lock acquire karta hai.
- Transaction A insert/update/delete karna chahta hai → check karna hoga ki purani ya nayi value kisi existing predicate lock se match karti hai ya nahi.

Key idea: predicate locks un objects par bhi apply hote hain jo abhi exist nahi karte (**phantoms**).

**Index-Range Locks (Next-Key Locking)**  
Predicate locks perform nahi karte (matching locks check karna time-consuming hai). Zyadatar 2PL wale databases **index-range locking** use karte hain — ek simplified approximation:

- Instead of locking "bookings for room 123 between noon and 1pm," lock "all bookings for room 123" ya "all bookings between noon and 1pm."
- Index entry (e.g., `room_id = 123` ya time-based index mein time range) ke saath shared lock attach kar do.
- Koi bhi transaction jo matching record insert/update/delete karna chahe, use same index update karna hoga → lock encounter karega → wait force ho jaayega.

Predicate locks jitna precise nahi (bada range lock kar sakta hai), lekin bahut kam overhead — accha compromise.  
Agar suitable index nahi hai, to database poori table par shared lock le leta hai.

### 3.3 Serializable Snapshot Isolation (SSI)  
Ek promising algorithm jo snapshot isolation ke muqable sirf small performance penalty ke saath full serializability provide karta hai. Pehli baar 2008 mein describe kiya gaya (Michael Cahill ki PhD thesis).

Used in: PostgreSQL (since version 9.1), FoundationDB.

**Pessimistic vs Optimistic Concurrency Control**  
- **2PL pessimistic hai** — Agar kuch bhi galat ho sakta hai, tab tak wait karo jab tak safe na ho. Multi-threaded programming mein mutual exclusion ki tarah.
- **Serial execution pessimistic to the extreme hai** — Har transaction ke paas poore database/partition par exclusive lock hai.
- **SSI optimistic hai** — Transactions bina block hue continue karti hain. Jab transaction commit karna chahti hai, database check karta hai ki isolation violate hui ya nahi; agar haan, to transaction abort aur retry.

Optimistic concurrency control high contention mein bura perform karta hai (bahut saari transactions same objects access karein → high abort rate). Lekin agar enough spare capacity ho aur low contention ho, to pessimistic approaches se better perform karta hai.

SSI snapshot isolation + algorithm for detecting serialization conflicts among writes par based hai.

**Detecting Stale MVCC Reads**  
<img width="2880" height="1748" alt="image" src="https://github.com/user-attachments/assets/e1af74fd-320b-48dc-859f-f2d3c3a5b07f" />


**Figure 7-10: Detecting when a transaction reads outdated values from an MVCC snapshot**  
(Aathvi image: Transaction 42 read karta hai, Transaction 43 update karta hai, 42 ko abort karna padega.)

Snapshot isolation ke under, transaction uncommitted transactions ke writes ignore karta hai (MVCC visibility rules).  
SSI track karta hai ki kab transaction doosre transaction ke writes ignore karta hai. Jab transaction commit karna chahti hai, database check karta hai ki kya ignored writes mein se koi ab commit ho chuki hai. Agar haan → abort.

Commit tak wait kyun? Read-only transaction abort karne ki zaroorat nahi. Doosri transaction abort ho sakti hai. Unnecessary aborts avoid karke, SSI snapshot isolation ka support for long-running reads preserve karta hai.

**Detecting Writes That Affect Prior Reads**  
<img width="2880" height="1792" alt="image" src="https://github.com/user-attachments/assets/c8f8cc20-b1b7-4208-94ec-203653bfcfee" />


**Figure 7-11: In SSI, detecting when one transaction modifies another transaction's reads**  
(Nauvi image: Index-range locks track karte hain kaunsi transactions ne kya padha, update detect hota hai.)

Jab transaction database mein likhta hai, to wo indexes mein dekhta hai ki kya recently kisi aur transaction ne affected data padha hai.  
Similar to acquiring a write lock, lekin block karne ki jagah, lock **tripwire** ka kaam karta hai: transactions ko notify karta hai ki unka data shayad ab up-to-date nahi hai.  
Pehli transaction commit hone mein success hoti hai; conflicting transaction commit karne ki koshish karti hai, to conflicting write already committed ho chuki hoti hai → abort karna padta hai.

**Performance of SSI**  
- **vs 2PL:** Ek transaction doosre ke locks ka wait nahi karta. Writers readers ko block nahi karte, readers writers ko block nahi karte → much more predictable latency. Read-only queries bina locks ke consistent snapshot par run hoti hain.
- **vs Serial Execution:** Single CPU core tak limited nahi. FoundationDB conflict detection multiple machines par distribute karta hai → very high throughput scale karta hai. Transactions multiple partitions ke across read aur write kar sakti hain.

Abort rate overall performance affect karta hai. Read-write transactions fairly chhoti honi chahiye (long-running read-only transactions okay hain). SSI probably 2PL ya serial execution ke muqable slow transactions ke liye less sensitive hai.

---

Summary  
**Race Conditions and Isolation Levels**

| Race Condition                   | Read Committed        | Snapshot Isolation            | Serializable        |
|----------------------------------|-----------------------|-------------------------------|---------------------|
| Dirty reads                      | ✅ Prevented          | ✅ Prevented                  | ✅ Prevented        |
| Dirty writes                     | ✅ Prevented          | ✅ Prevented                  | ✅ Prevented        |
| Read skew (nonrepeatable reads)  | ❌                    | ✅ Prevented (MVCC)           | ✅ Prevented        |
| Lost updates                     | ❌                    | ⚠️ Some implementations detect | ✅ Prevented        |
| Write skew                       | ❌                    | ❌                            | ✅ Prevented        |
| Phantom reads                    | ❌                    | ⚠️ Read-only OK; read-write problematic | ✅ Prevented |

**Three Approaches to Serializable Transactions**

|                    | Serial Execution                         | Two-Phase Locking (2PL)                    | SSI                                           |
|--------------------|------------------------------------------|--------------------------------------------|-----------------------------------------------|
| **Approach**       | Single-threaded, one at a time           | Shared/exclusive locks, readers block writers | Optimistic, check at commit time              |
| **Performance**    | Limited to one CPU core                  | Significantly worse throughput, unstable latencies | Small penalty vs snapshot isolation           |
| **Scalability**    | Partitioning helps, but cross-partition is slow | Deadlocks, lock contention                  | Distributes across multiple machines          |
| **Best for**       | Small, fast transactions; dataset fits in memory | Traditional RDBMS serializability           | General purpose; read-heavy workloads         |
