
Aaj kal ki bahut saari applications data-intensive hoti hain, na ki compute-intensive. Raw CPU power rarely limiting factor hoti hai—usually bade problems hote hain: data ki quantity, data ki complexity, aur jis speed se wo change ho raha hai.

Ek data‑intensive application typically in standard building blocks se bani hoti hai:

- **Databases** – Data ko store karna taaki wo (ya koi aur application) baad mein use kar sake.
- **Caches** – Expensive operation ka result yaad rakhna, taaki reads fast ho jaayein.
- **Search Indexes** – Users ko keyword se search karne dena ya alag‑alag tarah se filter karne dena.
- **Stream Processing** – Ek process se doosre process ko message bhejna, jo asynchronously handle ho.
- **Batch Processing** – Periodically bade accumulated data ko crunch karna.

Section 1: Thinking About Data Systems  
Hum normally databases, queues, caches etc. ko alag‑alag categories ke tools samajhte hain. Lekin in categories ke beech ki boundaries ab blur hoti ja rahi hain:

- Datastores ko message queue ki tarah use kiya jaata hai (jaise Redis).
- Message queues ke paas database‑jaisi durability guarantees hoti hain (jaise Apache Kafka).
- Increasingly, applications ki requirements itni demanding hoti hain ki ek single tool saare data processing aur storage needs poore nahi kar sakta. Kaam ko alag‑alag tasks mein todna padta hai jo different tools perform karte hain, aur application code unhe aapas mein jodta hai.

For example, agar aapke paas application‑managed caching layer hai (Memcached ya similar), ya ek full‑text search server (Elasticsearch ya Solr) jo main database se alag hai, to generally **application code ki responsibility** hoti hai ki wo caches aur indexes ko main database ke saath sync mein rakhe.
<img width="2880" height="2049" alt="image" src="https://github.com/user-attachments/assets/cc35a5cf-7028-4a47-9b5f-2d6f41c99313" />

Figure 1‑1: Ek possible architecture for a data system jo kai components ko combine karta hai.  
Jab aap kai tools ko combine karte ho ek service provide karne ke liye, to service ki API usually implementation details ko clients se chhupa leti hai.  
Aapka composite data system kuch guarantees provide kar sakta hai: e.g., cache sahi tareeke se invalidate ya update hoga writes ke time, taaki outside clients consistent results dekhein.  
Aapne essentially ek naya, special‑purpose data system create kar liya hai, chhote general‑purpose components se.  
Ab aap sirf application developer nahi hain, balki ek **data system designer** bhi hain.

### Key Design Questions  
Jab aap ek data system ya service design karte ho, to tricky questions aate hain:

- Kaise ensure karein ki data sahi aur complete rahe, even jab internally cheezein wrong ho jaayein?
- Kaise consistently good performance dein clients ko, even jab system ke kuch parts degraded hain?
- Load increase hone par kaise scale karein?
- Service ke liye accha API kaisa dikhega?

Bahut saare factors design ko influence karte hain: involved logon ki skills aur experience, legacy system dependencies, delivery ka timescale, organization ka risk tolerance, regulatory constraints, etc.

### Three Core Concerns  
Yeh book teen concerns par focus karti hai jo almost har software system mein important hain:

1. **Reliability** – System ko sahi se kaam karte rehna chahiye, even jab adversity aaye (hardware/software faults, aur human error).
2. **Scalability** – Jab system grow kare (data volume, traffic volume, ya complexity mein), to growth handle karne ke reasonable tareeke hone chahiye.
3. **Maintainability** – Time ke saath bahut saare alag log system par kaam karenge, aur un sabko productively kaam karne layak hona chahiye.

---

Section 2: Reliability  
Reliability ka matlab hai "jab cheezein galat ho jaayein tab bhi sahi se kaam karte rehna." Software ke liye typical expectations:

- Application wahi function kare jo user ne expect kiya.
- Wo tolerate kare ki user mistakes kare ya unexpected tareeke se software use kare.
- Performance expected load aur data volume ke under use case ke liye kaafi acchi ho.
- System kisi bhi unauthorized access aur abuse ko roke.

**Faults vs Failures**  
- **Fault** = system ka ek component apni spec se deviate kare.
- **Failure** = system as a whole user ko required service dena band kar de.

Fault ki probability ko zero karna impossible hai; isliye better hai fault‑tolerance mechanisms design karna jo faults ko failures banne se rokein.  
Kabhi kabhi deliberately faults induce karna samajhdaari hota hai taaki fault‑tolerance machinery test ho sake (e.g., Netflix Chaos Monkey—randomly individual processes ko bina warning ke kill kar deta hai taaki fault‑tolerance continuously exercise hoti rahe aur test hoti rahe).

Bahut saare critical bugs actually poor error handling ki wajah se hote hain; deliberately faults induce karne se confidence badhta hai ki natural faults ko sahi se handle kiya jaayega.  
Although generally hum faults ko tolerate karna prefer karte hain, kuch cases mein prevention better hota hai (e.g., security—agar attacker system compromise kar chuka hai to wo event undo nahi ho sakta).

### 2.1 Hardware Faults  
Hard disks crash ho jaate hain, RAM faulty ho jaati hai, power grids blackout ho jaate hain, network cables unplug ho jaati hain.  
Hard disks ka mean time to failure (MTTF) around 10 se 50 saal hota hai.  
10,000 disks wale storage cluster mein average per day ek disk fail hogi.

**Approaches**  
- Individual hardware components mein redundancy add karna:
  - Disks ke liye RAID configuration.
  - Dual power supplies aur hot‑swappable CPUs.
  - Batteries aur diesel generators for backup power.
- Yeh approach hardware problems ko failures banne se completely nahi rok sakti, lekin yeh acchi tarah samjhi hui hai aur aksar ek machine ko saalon tak uninterrupted chala sakti hai.
- Until recently, hardware redundancy hi kaafi thi zyadatar applications ke liye—multi‑machine redundancy sirf kuch applications ko chahiye thi jahan high availability absolutely essential thi.
- **Software fault‑tolerance techniques** – Jaise data volumes aur computing demands badhi hain, zyada applications bade numbers of machines use kar rahi hain, jisse hardware faults ki rate proportionally badh jaati hai.
- Kuch cloud platforms mein (e.g., AWS) virtual machine instances ka bina warning unavailable ho jaana fairly common hai, kyunki platforms single‑machine reliability se zyada flexibility aur elasticity ko prioritize karte hain.
- Isliye move ho raha hai un systems ki taraf jo poori machine ke loss ko tolerate kar sakein.
- **Operational advantage**: rolling upgrades allow karta hai—ek node ko ek time par patch karna, bina poore system ke downtime ke (e.g., OS security patches apply karna).

### 2.2 Software Errors  
Systematic errors (system ke andar) anticipate karna mushkil hota hai aur yeh uncorrelated hardware faults se kahin zyada failures cause karte hain kyunki yeh correlated hote hain across nodes. Examples:

- Ek software bug jo kisi particular bad input par har instance ko crash kar de (e.g., June 30, 2012 leap second bug Linux kernel mein).
- Ek runaway process jo shared resources—CPU time, memory, disk space, ya network bandwidth—kha jaaye.
- Ek dependent service jo slow ho jaaye, unresponsive ho jaaye, ya corrupted responses return kare.
- **Cascading failures**, jahan ek component mein chhoti si fault doosre components mein faults trigger kar deti hai.

**Mitigations**  
- Is type ke software faults ke bugs aksar lamba time dormant rehte hain jab tak koi unusual set of circumstances unhe trigger na kare. Software apne environment ke baare mein kuch assumption bana leta hai—aur jab wo assumption usually true hoti hai, lekin eventually kisi reason se false ho jaati hai.
- Systematic faults ka koi quick fix nahi hai. Bahut saari chhoti cheezein help kar sakti hain:
  - Carefully assumptions aur interactions ke baare mein sochna.
  - Thorough testing aur process isolation.
  - Processes ko crash aur restart hone dena.
  - Production mein system behavior ko measure, monitor aur analyze karna.
  - Agar system koi guarantee provide karta hai (e.g., message queue mein incoming messages ka number outgoing messages ke number ke barabar hona), to wo continuously khud ko check kar sakta hai aur alert raise kar sakta hai agar discrepancy mile.

### 2.3 Human Errors  
Humans unreliable hote hain—operators dwara configuration errors outages ka leading cause hain, jabki hardware faults sirf 10–25% outages mein role play karte hain.

**Approaches to Mitigate Human Errors**  
1. **Error ke opportunities minimize karo** – Well‑designed abstractions, APIs, aur admin interfaces jo "the right thing" karna easy banayein aur "the wrong thing" ko discourage karein. Lekin agar interfaces bahut restrictive hain to log unhe work‑around kar lenge, jo benefit ko negate kar deta hai—yeh tricky balance hai.
2. **Mistake‑prone areas ko failure‑prone areas se decouple karo** – Fully featured non‑production sandbox environments provide karo jahan log safely explore aur experiment kar sakein, real data ke saath, without affecting real users.
3. **Thoroughly test karo at all levels** – Unit tests, whole‑system integration tests, aur manual tests. Automated testing especially valuable hai corner cases cover karne ke liye jo normal operation mein rarely aate hain.
4. **Quick aur easy recovery allow karo** – Configuration changes ko rollback karna fast banao, naye code ko gradually roll out karo (taaki bugs sirf users ke chhote subset ko affect karein), aur data recompute karne ke tools provide karo (agar purani computation galat thi).
5. **Detailed aur clear monitoring set up karo** – Performance metrics aur error rates (telemetry). Jaise rocket launch ke baad, telemetry essential hai track karne ke liye ki kya ho raha hai aur failures ko samajhne ke liye. Monitoring early warning signals deta hai aur check karne deta hai ki assumptions ya constraints violate ho rahe hain ya nahi.
6. **Good management practices aur training** – Ek complex aur important aspect.

**How Important Is Reliability?**  
- Business applications mein bugs lost productivity aur legal risks cause karte hain.
- Ecommerce sites ke outages ka huge cost hota hai lost revenue aur reputation damage mein.
- Even "noncritical" applications ki users ke prati responsibility hoti hai (e.g., ek photo app jo family memories store karta hai).
- Kabhi kabhi cost reasons ki wajah se reliability sacrifice ki jaati hai (prototypes, narrow profit margins), lekin yeh conscious decision hona chahiye.

---

Section 3: Scalability  
**Scalability** wo term hai jo describe karta hai system ki ability ko cope with increased load. Yeh ek one‑dimensional label nahi hai—scalability discuss karte waqt aise sawaal sochte hain:

- "Agar system kisi particular tareeke se grow kare, to growth handle karne ke liye humaare paas kya options hain?"
- "Additional load handle karne ke liye hum computing resources kaise add kar sakte hain?"

### 3.1 Describing Load  
Load ko kuch numbers se describe kiya ja sakta hai jinhe **load parameters** kehte hain. Best choice architecture par depend karta hai:

- Web server per requests per second.
- Database mein reads vs writes ka ratio.
- Chat room mein simultaneously active users ki sankhya.
- Cache ka hit rate.

**Twitter Example**  
Twitter ke do main operations (data from November 2012):

1. **Post tweet** – User apne followers ke liye naya message publish kare (4.6k requests/sec average, peak par 12k+).
2. **Home timeline** – User un tweets ko dekhe jo unke follow kiye hue logon ne post kiye hain (300k requests/sec).

Scaling challenge hai **fan‑out**—har user bahut logon ko follow karta hai, aur har user ko bahut log follow karte hain. Fan‑out electronics engineering se liya hua term hai, jo batata hai ki ek incoming request serve karne ke liye doosre services ko kitni requests bhejni padti hain.

**Approach 1: Query on Read**  
- Naya tweet ek global collection mein insert karo.
- Jab user apna home timeline request kare, to sabhi follow kiye gaye logon ko lookup karo, unke saare tweets dhundho, aur unhe time ke hisaab se sorted merge karo.
- Relational database mein yeh SQL se ho sakta hai:
  ```sql
  SELECT tweets.*, users.* FROM tweets
    JOIN users   ON tweets.sender_id    = users.id
    JOIN follows ON follows.followee_id = users.id
    WHERE follows.follower_id = current_user
  ```
- Twitter ne initially yeh approach use kiya lekin systems home timeline queries ke load se struggle karne lage.
<img width="2880" height="1037" alt="image" src="https://github.com/user-attachments/assets/0d67d1bc-894f-43e8-878e-68200ee353c5" />

Figure 1‑2: Simple relational schema for implementing a Twitter home timeline.

**Approach 2: Pre‑compute on Write**  
- Har user ke home timeline ke liye ek cache maintain karo (jaise mailbox).
- Jab user tweet post kare, to sabhi followers ko lookup karo aur tweet ko unke har home timeline cache mein insert karo.
- Home timeline read karna ab cheap ho jaata hai kyunki wo pehle se pre‑computed hai.
<img width="2880" height="1037" alt="image" src="https://github.com/user-attachments/assets/a1cf365d-4b80-48c0-a808-23bfc82d67bd" />

Figure 1‑3: Twitter's data pipeline for delivering tweets to followers.

Yeh approach better kaam karti hai kyunki published tweets ki average rate home timeline reads ki rate se lagbhag do orders of magnitude kam hai—isliye write time par zyada kaam karna aur read time par kam kaam karna preferable hai.  
Trade‑off: Ek tweet average 75 followers ko deliver hota hai, to 4.6k tweets/sec ban jaate hain **345k writes/sec** home timeline caches mein.  
Kuch users ke 30 million se zyada followers hain—ek single tweet se 30 million+ writes ho sakti hain.  
Twitter tweets ko followers tak 5 seconds ke andar deliver karne ka aim rakhta hai.

**Hybrid Approach (Final Solution)**  
- Zyadatar users ke tweets write time par fan‑out kiye jaate hain (Approach 2).
- Celebrities (bahut bade follower count wale users) ko fan‑out se except kar diya jaata hai.
- Celebrity tweets ko separately fetch kiya jaata hai aur read time par merge kiya jaata hai (Approach 1).

Yeh hybrid consistently good performance deta hai.  
Key insight: **Followers per user ka distribution** ek key load parameter hai scalability discuss karne ke liye, kyunki yeh fan‑out load determine karta hai.

### 3.2 Describing Performance  
Jab load describe ho jaaye, to investigate karo ki load increase hone par kya hota hai:

- Jab aap ek load parameter badhao aur system resources unchanged rakho, to performance kaise affect hoti hai?
- Jab aap load parameter badhao, to performance unchanged rakhne ke liye kitne resources badhane padenge?

**Key Metrics**  
- **Batch processing systems** (e.g., Hadoop): Care about **throughput**—records processed per second, ya total time for a job.
- **Online systems**: Care about **response time**—client request bhejne aur response receive karne ke beech ka time.

**Latency vs Response Time**  
- **Response time** wo hai jo client dekhta hai: service time + network delays + queueing delays.
- **Latency** wo duration hai jab request handle hone ka wait kar rahi ho (awaiting service).
- Inhe aksar synonymously use kiya jaata hai, lekin yeh same nahi hain.

**Response Time as a Distribution**  
- Ek hi request ko baar baar karne par bhi har baar response time thoda alag aata hai.
- Random additional latency in cheezon se aa sakti hai: context switch to a background process, network packet loss aur TCP retransmission, garbage collection pause, page fault forcing disk read, server rack mein mechanical vibrations, aur bahut saare aur reasons.
- Isliye response time ko ek single number nahi, balki **distribution of values** samjho.
<img width="1064" height="330" alt="image" src="https://github.com/user-attachments/assets/d8bc044d-de69-45cf-809c-b66be1104b2e" />

Figure 1‑4: Illustrating mean and percentiles: response times for a sample of 100 requests to a service.

**Percentiles**  
- Response time ko distribution of values ki tarah sochna chahiye, single number nahi.
- **Mean (average)** accha metric nahi hai—yeh nahi batata ki kitne users ne wo delay experience kiya.
- **Median (p50)** – Halfway point. Aadhi requests faster, aadhi slower. Isse 50th percentile bhi kehte hain.
  - Note: median ek single request ko refer karta hai. Agar user kai requests karta hai (session ke dauran, ya ek page mein multiple resources include hain), to probability ki kam se kam ek request median se slower ho, 50% se kahin zyada hai.
- **Higher percentiles** – p95, p99, p999 wo response time thresholds hain jahan 95%, 99%, ya 99.9% requests faster hoti hain.
  - Inhe **tail latencies** bhi kehte hain.
  - Amazon internal services ke liye 99.9th percentile use karta hai kyunki slowest requests aksar most valuable customers se aati hain (most data, most purchases).
  - 100 ms increase in response time se sales 1% kam ho jaati hain.
  - 1‑second slowdown customer satisfaction 16% reduce kar deta hai.
  - 99.99th percentile optimize karna aksar bahut mehnga hota hai diminishing returns ke saath.

**SLOs and SLAs**  
- **Service Level Objectives (SLOs)** aur **Service Level Agreements (SLAs)** expected performance aur availability define karte hain.
- Example SLA: median response time < 200 ms, 99th percentile < 1 s, service up at least 99.9% of the time.

**Head‑of‑Line Blocking**  
- Queueing delays aksar high percentiles par response time ka bada hissa hote hain.
- Ek server ek time par sirf kuch chizein parallel process kar sakta hai (CPU cores se limited). Chhoti si number of slow requests baaki subsequent requests ko hold up kar sakti hain—ise **head‑of‑line blocking** kehte hain.
- Chahe subsequent requests process karne mein fast hon, client ko overall response time slow dikhta hai kyunki usse pehle wali request complete hone ka wait karna padta hai.
- **Response times client side measure karna important hai.**
- Jab load artificially generate karte ho scalability test karne ke liye, to load‑generating client ko independently requests bhejte rehna chahiye, response time se independent. Agar client pehli request complete hone ka wait kare next bhejne se pehle, to wo artificially queues ko reality se chhota rakh deta hai, jo measurements ko distort karta hai.

**Tail Latency Amplification**  
- Backend services jo ek end‑user request serve karne ke liye multiple times call hoti hain, unmein ek bhi slow call poori request slow kar deti hai.
- Jitni zyada backend calls chahiye, utna hi chance badh jaata hai kisi slow call ke hit hone ka.
<img width="2880" height="1304" alt="image" src="https://github.com/user-attachments/assets/091f21f5-9154-42ad-8ae0-55040be50dcd" />

Figure 1‑5: When several backend calls are needed to serve a request, it takes just a single slow backend request to slow down the entire end‑user request.

**Calculating Percentiles**  
- Efficient approximation ke algorithms: forward decay, t‑digest, HdrHistogram.
- **Percentiles ko kabhi average mat karo**—sahi tareeka hai histograms ko add karna.

### 3.3 Approaches for Coping with Load  
Jo architecture ek level ke load ke liye appropriate hai, wo 10x load ke saath cope kar paye aisa unlikely hai.  
Har order of magnitude load increase par architecture rethink karna pad sakta hai.

**Scaling Up vs Scaling Out**  
- **Scaling up (vertical scaling)** – Zyada powerful machine par move karna.
- **Scaling out (horizontal scaling)** – Load ko multiple smaller machines par distribute karna (ise **shared‑nothing architecture** bhi kehte hain).
- Reality mein acchi architectures dono approaches ka pragmatic mixture hoti hain.

**Elastic vs Manual Scaling**  
- **Elastic systems** automatically computing resources add kar dete hain jab load badhta hai.
  - Useful agar load highly unpredictable ho.
- **Manually scaled systems** simpler hote hain aur operational surprises kam de sakte hain.

**Stateless vs Stateful**  
- Stateless services ko multiple machines par distribute karna fairly straightforward hai.
- Stateful data systems ko single node se distributed banane mein bahut saari additional complexity introduce hoti hai.
- Common wisdom: apne database ko single node par rakho (scale up) jab tak scaling cost ya high‑availability requirements force na karein distributed hone ko.

**No Magic Scaling Sauce**  
- Large‑scale systems ki architecture usually highly specific to the application hoti hai.
- Problem ho sakti hai: volume of reads, writes, data to store, complexity, response time requirements, access patterns, ya inka mixture.
- For example, ek system jo 100,000 requests/sec handle karta hai, har request 1 kB ki, wo bilkul alag dikhega ek system se jo 3 requests/min handle karta hai, har request 2 GB ki—chahe dono ka data throughput same ho.
- Ek architecture jo acchi tarah scale karti hai wo un assumptions par built hoti hai ki kaunse operations common hain aur kaunse rare—**load parameters**. Agar wo assumptions galat niklein, to scaling ki engineering effort best case mein waste jaati hai, worst case mein counterproductive.
- Early‑stage startup ya unproven product mein, usually zyada important hota hai product features par jaldi iterate karna, rather than kisi hypothetical future load ke liye scale karna.
- Scalable architectures general‑purpose building blocks se banti hain jo familiar patterns mein arranged hote hain.

---

Section 4: Maintainability  
Software ki cost ka majority initial development mein nahi, balki ongoing maintenance mein hota hai—bugs fix karna, systems operational rakhna, failures investigate karna, naye platforms ke liye adapt karna, naye use cases ke liye modify karna, technical debt repay karna, aur naye features add karna.

Bahut saare log jo software systems par kaam karte hain, maintenance of so‑called **legacy systems** ko dislike karte hain—shaayad ismein doosron ki galtiyan fix karni padti hain, outdated platforms ke saath kaam karna padta hai, ya systems ko aise kaam karne padte hain jinke liye wo kabhi design nahi kiye gaye the. Har legacy system apne tareeke se unpleasant hota hai.

Lekin hum **design kar sakte hain** taaki maintenance ke dauraan pain minimize ho, aur khud legacy software create karne se bacha ja sake. Teen design principles:

### 4.1 Operability: Making Life Easy for Operations  
_"Good operations can often work around the limitations of bad software, but good software cannot run reliably with bad operations."_

**Operations Team Responsibilities**  
- System health monitor karna aur service jaldi restore karna.
- Problems ke causes track karna (failures, degraded performance).
- Software aur platforms ko up to date rakhna, including security patches.
- Dhyan rakhna ki alag systems ek doosre ko kaise affect karte hain.
- Future problems anticipate karna (e.g., capacity planning).
- Deployment aur configuration management ke liye acchi practices establish karna.
- Complex maintenance tasks perform karna (e.g., platform migrations).
- Configuration changes ke saath system security maintain karna.
- Predictable operations aur stable production environment ke liye processes define karna.
- Logon ke aane‑jaane par organizational knowledge preserve karna.

**Good Operability Means**  
- Runtime behavior mein visibility dena acchi monitoring ke saath.
- Automation aur standard tools ke saath integration ke liye accha support.
- Individual machines par dependency avoid karna.
- Acchi documentation aur easy‑to‑understand operational model.
- Good default behavior lekin defaults override karne ki freedom.
- Self‑healing jahan appropriate ho, lekin zaroorat padne par manual control.
- Predictable behavior, minimizing surprises.

### 4.2 Simplicity: Managing Complexity  
Jaise projects bade hote hain, wo aksar bahut complex aur difficult to understand ho jaate hain, jo sabko slow kar deta hai aur maintenance cost badha deta hai. Ek software project jo complexity mein dooba ho use kabhi **big ball of mud** kaha jaata hai.

**Symptoms of Complexity**  
- State space ka explosion.
- Modules ka tight coupling.
- Tangled dependencies.
- Inconsistent naming aur terminology.
- Performance problems ke liye hacks.
- Issues ko work around karne ke liye special‑casing.

Jab complexity maintenance ko mushkil bana deti hai, to budgets aur schedules aksar overrun ho jaate hain. Complex software mein koi change karte waqt bugs introduce karne ka risk bhi zyada hota hai—hidden assumptions, unintended consequences, aur unexpected interactions aasani se overlook ho jaate hain.

**Accidental vs Essential Complexity**  
- **Accidental complexity** wo hai jo problem ke inherent nature se nahi aati (jaise users dekhte hain) balki sirf implementation se aati hai.
- System ko simpler banana necessarily functionality kam karna nahi hai—iska matlab hai accidental complexity ko hatana.

**Abstraction**  
- Accidental complexity hatane ka best tool hai **abstraction**.
- Ek acchi abstraction implementation detail ko ek clean, simple‑to‑understand façade ke peeche chhupa leti hai. Use wide range of applications ke liye bhi use kiya ja sakta hai—reuse is more efficient than reimplementing, aur abstracted component mein quality improvements se sabhi applications ko fayda hota hai jo use karte hain.
- Examples:
  - High‑level programming languages abstract away machine code, CPU registers, aur syscalls.
  - SQL abstracts away complex on‑disk aur in‑memory data structures, doosre clients se concurrent requests, aur crashes ke baad inconsistencies.
- Acchi abstractions dhundhna bahut mushkil hai, especially distributed systems mein—bahut saare acchi algorithms hain, lekin unhe aise abstractions mein package karna jisse complexity manageable rahe, yeh bahut kam clear hai.

### 4.3 Evolvability: Making Change Easy  
System requirements hamesha constant flux mein hote hain—naye facts, previously unanticipated use cases emerge hote hain, business priorities change hoti hain, users naye features request karte hain, naye platforms purane platforms replace karte hain, legal ya regulatory requirements badalte hain, growth architectural changes force karti hai.

- **Agile** working patterns change adapt karne ka framework dete hain. Technical tools mein test‑driven development (TDD) aur refactoring shaamil hain.
- Zyadatar Agile techniques fairly small, local scale par focus karti hain (ek hi application ke andar kuch source code files). Yeh book ways of increasing agility at the level of a larger data system dhundhti hai—shaayad kai alag applications ya services jinki different characteristics hain.
- For example, aap Twitter ki architecture for assembling home timelines ko Approach 1 se Approach 2 mein kaise "refactor" karoge?
- Kisi data system ko modify karne ki ease closely linked hoti hai uski simplicity aur abstractions se—simple aur easy‑to‑understand systems usually complex systems se zyada aasani se modify hote hain.
- **Evolvability** (ya **extensibility**, **modifiability**, **plasticity**) = agility on a data system level.

---

Summary  
**Key Takeaways**  
- Ek application ko **functional requirements** (kya karna chahiye) aur **nonfunctional requirements** (reliability, scalability, maintainability, security, compliance, compatibility) meet karne hote hain.
- **Reliability** – Systems ko faults hone par bhi sahi se kaam karte rehna. Faults ho sakte hain: hardware (random, uncorrelated), software (systematic, hard to deal with), aur humans (inevitable mistakes). Fault‑tolerance techniques kuch faults ko end users se chhupa leti hain.
- **Scalability** – Performance acchi rakhne ke strategies even when load increases. Load aur performance ko quantitatively describe karne ke tareeke chahiye (load parameters, percentiles). High load par reliable rahne ke liye processing capacity add karna.
- **Maintainability** – Engineering aur operations teams ke liye life better banana. Acchi abstractions complexity kam karti hain. Good operability means good visibility aur effective management. Evolvability ke liye design karo.
- Koi easy fix nahi hai—lekin kuch patterns aur techniques alag‑alag applications mein baar baar dikhte hain.
