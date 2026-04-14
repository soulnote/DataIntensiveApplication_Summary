Applications time ke saath inevitably change hoti hain. Jab aap application ke features badalte ho, to aksar jo data wo store karti hai use bhi change karna padta hai. Lekin ek badi application mein, code changes instantaneously nahi ho sakte:

- **Server-side applications** – Aap **rolling upgrade** (staged rollout) perform karna chahte ho, naye version ko ek baar mein kuch nodes par deploy karte ho. Isse bina service downtime ke deployment ho jaata hai aur frequent releases encourage hoti hain.
- **Client-side applications** – Aap user ke raham-o-karam par hote ho, jo shayad update ko kuch time tak install na kare.

Iska matlab hai ki code ke purane aur naye versions, aur data ke purane aur naye formats, ek saath system mein coexist kar sakte hain.

**Backward and Forward Compatibility**  
- **Backward compatibility** – Naya code wo data padh sakta hai jo purane code ne likha tha. (Normally mushkil nahi — aap purana format jaante ho aur handle kar sakte ho.)
- **Forward compatibility** – Purana code wo data padh sakta hai jo naye code ne likha hai. (Zyada tricky — purane code ko naye version dwara kiye gaye additions ko ignore karna padta hai.)

---

Section 1: Formats for Encoding Data  
Programs data ke saath do alag representations mein kaam karte hain:

- **In memory** – Objects, structs, lists, arrays, hash tables, trees (CPU access ke liye optimized, pointers use karte hain).
- **On disk / over the network** – Bytes ka self-contained sequence (pointers doosre process ke liye meaningless hote hain).

Dono ke beech translation:
- In-memory → bytes = **encoding** (ya serialization, marshalling)
- Bytes → in-memory = **decoding** (ya parsing, deserialization, unmarshalling)

### 1.1 Language-Specific Formats  
Bahut saari languages ke paas built-in encoding hoti hai: Java ka `java.io.Serializable`, Ruby ka `Marshal`, Python ka `pickle`, Java ke liye Kryo, etc.

**Problems**  
- **Ek particular language se tied** – Data ko doosri language mein padhna bahut mushkil hai. Aap apni current language ke saath lambe time ke liye commit ho jaate ho.
- **Security problems** – Decoding ke liye arbitrary classes instantiate karne padte hain. Attacker aapki application se arbitrary byte sequence decode karwa sakta hai → remotely arbitrary code execute ho sakta hai.
- **Versioning ek afterthought hai** – Forward aur backward compatibility aksar ignore ho jaati hai.
- **Efficiency bhi afterthought hai** – Java ki built-in serialization kharab performance aur bloated encoding ke liye badnaam hai.

Generally, language-specific encoding ko bahut transient purposes ke alawa use karna bura idea hai.

### 1.2 JSON, XML, and Binary Variants  
JSON aur XML widely known hain, widely supported hain, aur lagbhag utne hi widely disliked bhi hain. CSV ek aur popular language-independent format hai.

**Subtle Problems**  
- **Number encoding ambiguity** – XML aur CSV numbers ko digit-strings se distinguish nahi kar sakte. JSON integers aur floating-point mein fark nahi karta, aur precision specify nahi karta.
  - 2^53 se bade integers IEEE 754 double-precision floating-point mein exactly represent nahi ho sakte. Twitter apne JSON API mein tweet IDs do baar include karta hai: ek baar number ki tarah, ek baar string ki tarah, taaki JavaScript ki parsing issues se bacha ja sake.
- **No binary string support** – JSON aur XML Unicode strings support karte hain lekin binary strings (bytes ke sequences) nahi. Workaround: Base64 encoding, jo data size 33% badha deta hai.
- **Schema support optional aur complex hai** – XML Schema kaafi widespread hai; bahut saare JSON tools schemas ke saath bother nahi karte. CSV mein to bilkul schema nahi hota.

In flaws ke bawajood, JSON, XML, aur CSV bahut saare purposes ke liye kaafi acche hain, khaas kar organizations ke beech data interchange formats ke roop mein.

**Binary Encoding**  
Internal use ke liye kisi organization mein, binary formats zyada compact aur parse karne mein faster ho sakte hain. JSON ke liye binary encodings hain: MessagePack, BSON, BJSON, UBJSON, BISON, Smile. XML ke liye: WBXML, Fast Infoset.

Kyunki yeh schema prescribe nahi karte, inhe encoded data mein saare field names include karne padte hain.

**MessagePack Example**  
Neeche diye gaye record ko encode karna:

```json
{
    "userName": "Martin",
    "favoriteNumber": 1337,
    "interests": ["daydreaming", "hacking"]
}
```
<img width="2880" height="2432" alt="image" src="https://github.com/user-attachments/assets/3686b630-3337-4e7b-8f5c-3b5aef3928fb" />

**Figure 4-1: Example record encoded using MessagePack**  
(Aapke dwara di gayi image mein byte sequence dikhaya gaya hai: 66 bytes.)

Binary encoding 66 bytes lambi hai, jo textual JSON (bina whitespace ke 81 bytes) se bas thodi kam hai.  
Yeh clear nahi hai ki itni chhoti space reduction human-readability khone ke layak hai ya nahi.

### 1.3 Thrift and Protocol Buffers  
**Apache Thrift** (originally Facebook mein develop hua) aur **Protocol Buffers** (protobuf, originally Google mein develop hua) binary encoding libraries hain jo same principle par based hain. Dono 2007–08 mein open source kiye gaye.

Dono ko kisi bhi data ko encode karne ke liye **schema** ki zaroorat hoti hai.

**Thrift Schema (IDL)**
```thrift
struct Person {
  1: required string       userName,
  2: optional i64          favoriteNumber,
  3: optional list<string> interests
}
```

**Protocol Buffers Schema**
```protobuf
message Person {
    required string user_name       = 1;
    optional int64  favorite_number = 2;
    repeated string interests       = 3;
}
```

Dono ke saath ek code generation tool aata hai jo alag-alag programming languages mein schema implement karne wali classes produce karta hai.

**Key Difference from MessagePack**  
Encoded data mein field names ki jagah **field tags** (numbers 1, 2, 3) hote hain. Field tags aliases ki tarah hain — fields ko identify karne ka compact tareeka bina naam likhe.

**Thrift BinaryProtocol (59 bytes)**  
<img width="2880" height="2154" alt="image" src="https://github.com/user-attachments/assets/e6a93991-4a1a-4df4-a205-9af5d6dd2200" />

**Figure 4-2: Example record encoded using Thrift's BinaryProtocol**  
(Image mein byte sequence aur breakdown dikhaya gaya hai.)

**Thrift CompactProtocol (34 bytes)**  
Field type aur tag number ko ek single byte mein pack karta hai, variable-length integers use karta hai (numbers –64 se 63 ek byte mein, –8192 se 8191 do bytes mein, etc.).
<img width="2880" height="1968" alt="image" src="https://github.com/user-attachments/assets/4fb5f516-c940-4055-abae-42ae45a05d72" />

**Figure 4-3: Example record encoded using Thrift's CompactProtocol**

**Protocol Buffers (33 bytes)**  
Thrift ke CompactProtocol jaisi bit packing.
<img width="2880" height="1775" alt="image" src="https://github.com/user-attachments/assets/699d985a-0432-4852-8af2-fb5abe8a813b" />

**Figure 4-4: Example record encoded using Protocol Buffers**

**Note:** `required` vs `optional` se encoding par koi farak nahi padta — yeh sirf runtime check hai jo fail ho jaata hai agar required field set nahi hai.

**Field Tags and Schema Evolution**  
- Encoded record apne encoded fields ka concatenation hota hai, har field apne tag number se identified hota hai aur datatype ke saath annotated hota hai.
- Aap field ka **naam** change kar sakte ho (encoded data kabhi field names refer nahi karta), lekin **field tag** nahi badal sakte (saara existing encoded data invalid ho jaayega).
- **Forward Compatibility (purana code naya data padhe)**: Naye fields naye tag numbers ke saath add karo. Purana code jo naya tag number nahi pehchanta, simply us field ko ignore kar deta hai (datatype annotation batata hai ki kitne bytes skip karne hain).
- **Backward Compatibility (naya code purana data padhe)**: Naya code hamesha purana data padh sakta hai kyunki tag numbers ka wahi meaning hota hai.
- Initial deployment ke baad add kiya gaya har field `optional` hona chahiye ya uski default value honi chahiye (agar `required` hai, to naya code purana data padhte waqt check fail kar dega).
- **Fields Hatana**: Sirf wahi field hata sakte ho jo `optional` ho (`required` fields kabhi nahi hata sakte). Aur kabhi bhi wahi tag number dobara use nahi kar sakte.
- **Datatype Changes**: 32-bit integer ko 64-bit mein badalna: naya code purana data padh leta hai (missing bits zero fill karta hai). Purana code naya data padhe to value truncate ho sakti hai.
- Protocol Buffers: `optional` → `repeated` safe hai (purana code sirf list ka last element dekhta hai). Thrift mein dedicated list datatype hai jo aise evolution allow nahi karta lekin nested lists support karta hai.

### 1.4 Avro  
**Apache Avro** 2009 mein Hadoop ke subproject ke roop mein shuru hua, kyunki Thrift Hadoop ke use cases ke liye accha fit nahi tha.

Do schema languages: Avro IDL (human editing ke liye) aur JSON (machine-readable).

**Avro IDL Schema**
```avro
record Person {
    string               userName;
    union { null, long } favoriteNumber = null;
    array<string>        interests;
}
```

**Key difference:** Schema mein koi tag numbers nahi hain.

**Avro Binary Encoding (32 bytes — sabse compact)**  
<img width="2880" height="1900" alt="image" src="https://github.com/user-attachments/assets/b7e7c442-c1f7-422f-b9ca-be958cc03661" />

**Figure 4-5: Example record encoded using Avro**  
(Image mein byte sequence dikhaya gaya hai.)

Encoded data mein kuch bhi fields ya unke datatypes ko identify nahi karta. Encoding bas values ko concatenate kar deti hai.  
Parse karne ke liye, fields ko us order mein padho jis order mein wo schema mein hain. Binary data sirf tabhi correctly decode ho sakta hai jab code padhne wala **exact wahi schema** use kare jo likhne wale ne use kiya tha.

**The Writer's Schema and the Reader's Schema**  
- **Writer's schema** – Jo schema application data encode karte waqt use karti hai.
- **Reader's schema** – Jo schema application data decode karte waqt expect karti hai.

**Key idea:** Writer's schema aur reader's schema ka same hona zaroori nahi hai — sirf **compatible** hona chahiye.  
Avro library dono schemas ko side by side dekhkar differences resolve karti hai aur translate karti hai.
<img width="2880" height="1033" alt="image" src="https://github.com/user-attachments/assets/a9e685da-1e07-4ccf-9dd6-9e7c8e64b1ad" />

**Figure 4-6: An Avro reader resolves differences between the writer's schema and the reader's schema**  
(Image mein dikhaya hai ki reader schema mein alag field names aur types ho sakte hain, aur Avro unhe map karta hai.)

- Fields **field name** se match hote hain (position se nahi). Fields different order mein ho to bhi chalega.
- Field writer's schema mein ho lekin reader's schema mein nahi → ignored.
- Field reader expect kare lekin writer's schema mein na ho → default value se fill kiya jaata hai.

**Schema Evolution Rules**  
- Sirf wahi field add ya remove kar sakte ho jiski **default value** ho.
- Bina default value ke field add karna **backward compatibility** todta hai. Bina default value ke field hatana **forward compatibility** todta hai.
- Null allow karne ke liye, aapko **union type** use karna hoga: `union { null, long, string } field;` — Avro mein Thrift/Protobuf jaisa `optional`/`required` markers nahi hain.
- Field ka datatype badalna possible hai agar Avro type convert kar sake.
- Field ka naam badalna: reader's schema mein **aliases** use karo (backward compatible hai lekin forward compatible nahi).

**Reader Ko Writer's Schema Kaise Pata Chalti Hai?**  
- **Badi file jismein bahut saare records ho** – File ke beginning mein writer's schema ek baar include kar do (Avro object container files).
- **Database jismein individually written records ho** – Har record ke beginning mein version number include karo; database mein schema versions ki list rakho. Reader us version ke liye writer's schema fetch karta hai. (Espresso aise kaam karta hai.)
- **Network par records bhejna** – Connection setup par schema version negotiate karo, aur connection ke lifetime ke liye use karo (Avro RPC protocol).

**Dynamically Generated Schemas**  
Avro ka Thrift/Protobuf par key advantage: koi tag numbers nahi → dynamically generated schemas ke liye zyada friendly hai.

Example: relational database ko binary file mein dump karo. Relational schema se Avro schema generate karo (har table → record, har column → field, column name → field name).  
Agar database schema change ho, to bas naya Avro schema generate karo. Data export process ko schema changes par dhyan dene ki zaroorat nahi.  
Thrift/Protobuf ke saath, har baar database schema change hone par field tags manually assign karne padte.

**Code Generation**  
- Thrift aur Protobuf code generation par rely karte hain (statically typed languages jaise Java, C++, C# mein useful).
- Dynamically typed languages (JavaScript, Ruby, Python) mein code generation kam useful hai aur aksar naapasand kiya jaata hai.
- Avro optional code generation provide karta hai. Bina iske, aap simply Avro object container file khol sakte ho aur data dekh sakte ho (file self-describing hoti hai).

### 1.5 The Merits of Schemas  
Schemas par based binary encodings (Protocol Buffers, Thrift, Avro) ke paas acche properties hain:

- **"Binary JSON" variants se zyada compact** – Encoded data se field names omit kar sakte hain.
- **Schema valuable documentation hai** – Decoding ke liye required hai, isliye guaranteed up to date hota hai (manually maintained docs ke opposite).
- **Schemas ka database** forward aur backward compatibility deployment se pehle check karne deta hai.
- **Schema se code generation** statically typed languages mein compile-time type checking enable karta hai.
- **Schema evolution** wahi flexibility deta hai jo schemaless/schema-on-read JSON databases dete hain, saath mein data ke baare mein better guarantees aur better tooling.

---

Section 2: Modes of Dataflow  
Jab bhi aap kisi doosre process ko data bhejna chahte ho jiske saath aap memory share nahi karte, aapko use bytes ke sequence mein encode karna padta hai. Teen common ways jisse data processes ke beech flow karta hai:

1. Via databases
2. Via service calls (REST and RPC)
3. Via asynchronous message passing

### 2.1 Dataflow Through Databases  
Jo process database mein likhta hai wo data encode karta hai; jo process database se padhta hai wo decode karta hai. Aap database mein kuch store karna apne future self ko message bhejne jaisa soch sakte ho.

- **Backward compatibility** clearly necessary hai (warna aapka future self decode nahi kar paayega jo aapne pehle likha tha).
- **Forward compatibility** bhi aksar required hoti hai — rolling upgrade mein, naya version kuch value likh sakta hai jise baad mein abhi bhi running purana version padhega.

**Preserving Unknown Fields**  
Agar aap record schema mein naya field add karte ho, to naya code us naye field ke liye value likhta hai. Purana version record padhta hai, update karta hai, aur wapas likh deta hai. Desirable behavior: purane code ko naye field ko intact rakhna chahiye, chahe wo interpret na kar sake.
<img width="2880" height="1565" alt="image" src="https://github.com/user-attachments/assets/f54c9d49-222d-4454-8c3b-a08c07e5e695" />

**Figure 4-7: When an older version of the application updates data previously written by a newer version, data may be lost if you're not careful**  
(Image mein dikhaya gaya hai ki purana code JSON parse karta hai to naye field kho jaata hai.)

Agar aap database value ko model objects mein decode karte ho aur baad mein reencode karte ho, to unknown field translation mein kho sakta hai. Aapko application level par is baat se aware rehna hoga.

**Data Outlives Code**  
- Ek hi database ke andar, aapke paas values ho sakti hain jo paanch millisecond pehle likhi gayi aur values jo paanch saal pehle likhi gayi.
- Jab aap naya application version deploy karte ho, aap purane code ko minutes mein replace kar dete ho. Lekin paanch saal purana data original encoding mein hi rahega jab tak explicitly rewrite na kiya jaaye. Ise "data outlives code" kehte hain.
- Data ko naye schema mein rewrite (migrate) karna large datasets par expensive hai, isliye zyadatar databases isse avoid karte hain. Zyadatar relational databases simple schema changes allow karte hain (e.g., null default ke saath column add karna) bina existing data rewrite kiye — database purani rows padhte waqt missing columns ke liye nulls fill kar deta hai.
- LinkedIn ka Espresso Avro use karta hai storage ke liye, Avro ke schema evolution rules ka fayda uthata hai.

**Archival Storage**  
Database snapshots (backup ke liye ya data warehouse mein load karne ke liye) typically latest schema use karke encode kiye jaate hain, chahe source mein mixture of schema versions ho.  
Kyunki dump ek baar mein likha jaata hai aur immutable hai, Avro object container files jaise formats acche fit hain.  
Yeh bhi ek accha mauka hai analytics-friendly column-oriented format jaise **Parquet** mein encode karne ka.

### 2.2 Dataflow Through Services: REST and RPC  
Sabse common arrangement: **clients** aur **servers**. Server network par API expose karta hai (ek **service**), aur clients requests karne ke liye connect karte hain.

- Server khud kisi doosre service ka client ho sakta hai (e.g., web app server database ka client hai).
- Yeh approach ek badi application ko functionality ke area ke hisaab se chhoti services mein tod deta hai — traditionally **service-oriented architecture (SOA)**, recently **microservices architecture** ke roop mein refined.
- Services application-specific API expose karte hain (databases jaise arbitrary queries nahi) — encapsulation aur fine-grained restrictions provide karte hain ki clients kya kar sakte hain.
- Key design goal: services ko **independently deployable aur evolvable** banana. Har service ek team own karti hai, frequently naye versions release karti hai bina doosri teams ke saath coordinate kiye.

**Web Services**  
Jab HTTP underlying protocol ho. Kai contexts mein use hota hai:

- Client application (mobile app, JavaScript web app) public internet par requests karta hai.
- Ek service se doosri service same organization/datacenter ke andar (middleware).
- Ek service se doosri service jo different organization own karti hai (public APIs, OAuth, credit card processing).

**REST vs SOAP**  
- **REST** – Ek design philosophy jo HTTP principles par build karti hai. Simple data formats, resources identify karne ke liye URLs, HTTP features for cache control, authentication, content type negotiation par jor deti hai. Microservices ke saath associated hai. REST principles ke according design ki gayi API **RESTful** kehlati hai.
- **SOAP** – Network API requests ke liye XML-based protocol. HTTP se independent hona aim karta hai. Related standards ki sprawling multitude (WS-*) ke saath aata hai. API **WSDL** (Web Services Description Language) se describe hoti hai — code generation enable karta hai lekin human-readable nahi hai. Users heavily tool support aur IDEs par rely karte hain.
- SOAP zyadatar chhoti companies mein favor kho chuka hai lekin ab bhi bahut saari badi enterprises mein use hota hai.
- RESTful APIs simpler approaches prefer karte hain jismein kam code generation ho. Definition formats jaise **OpenAPI** (Swagger) RESTful APIs describe kar sakte hain aur documentation produce kar sakte hain.

**The Problems with Remote Procedure Calls (RPCs)**  
RPC remote network service ko request local function call jaisa dikhane ki koshish karta hai (**location transparency**). Yeh fundamentally flawed hai:

- **Unpredictable** – Network request lost ho sakti hai, ya remote machine slow/unavailable ho sakti hai. Aapko network problems anticipate karni padti hain aur retry karna padta hai.
- **Timeouts** – Network request bina result ke return ho sakti hai. Aapko pata nahi ki request pahunchi ya nahi.
- **Idempotence needed** – Retry karne se action multiple times perform ho sakta hai (request pahunch gayi lekin response lost ho gaya). Deduplication mechanisms ki zaroorat hai.
- **Variable latency** – Network request function call se bahut slow hota hai, aur latency wildly variable hoti hai.
- **Encoding overhead** – Parameters bytes mein encode karne padte hain. Bade objects ke saath problematic.
- **Cross-language type translation** – RPC framework ko languages ke beech datatypes translate karne padte hain (e.g., JavaScript ka numbers > 2^53 ke saath problem).

REST ki appeal ka ek hissa yeh hai ki yeh chhupane ki koshish nahi karta ki yeh network protocol hai.

**Current Directions for RPC**  
Problems ke bawajood, RPC jaane wala nahi hai. Naye generation ke frameworks:

- **gRPC** – Protocol Buffers use karta hai.
- **Finagle** – Thrift use karta hai.
- **Rest.li** – JSON over HTTP use karta hai.
- Thrift aur Avro ke saath RPC support included hai.

Yeh frameworks zyada explicit hain ki remote requests local calls se alag hain:

- **Futures (promises)** – Asynchronous actions encapsulate karte hain jo fail ho sakte hain. Multiple services ko parallel requests simplify karte hain.
- **Streams (gRPC)** – Ek call time ke saath requests aur responses ki series par based hoti hai.
- **Service discovery** – Client ko find karne deta hai ki particular service kis IP address aur port number par hai.

**REST vs RPC Trade-offs**  
- Custom RPC with binary encoding JSON over REST se better performance achieve kar sakta hai.
- RESTful APIs ke significant advantages hain: experimentation aur debugging ke liye accha (curl, web browser), sabhi mainstream languages support karte hain, vast ecosystem of tools (servers, caches, load balancers, proxies, firewalls, monitoring).
- REST public APIs ke liye predominant hai. RPC mainly same organization/datacenter ke andar services ke beech requests ke liye hai.

**Data Encoding and Evolution for RPC**  
- Assume karo servers pehle update hote hain, clients baad mein → requests par **backward compatibility** aur responses par **forward compatibility** chahiye.
- Compatibility properties used encoding format se inherit hoti hain (Thrift, gRPC/Protobuf, Avro RPC, SOAP/XML schemas, RESTful/JSON).
- Service compatibility across organizational boundaries zyada mushkil hai — provider ke paas aksar clients par control nahi hota aur upgrades force nahi kar sakta. Shayad API ke multiple versions side by side maintain karne padein.
- API versioning approaches: URL mein version number, HTTP `Accept` header, ya per client API keys ke through stored (e.g., Stripe ka approach).

### 2.3 Message-Passing Dataflow  
**Asynchronous message-passing systems** RPC aur databases ke beech kahin hain:

- **RPC ki tarah:** Client ka request (message) doosre process tak low latency mein deliver hota hai.
- **Database ki tarah:** Message ek intermediary ke through jaata hai jise **message broker** (message queue / message-oriented middleware) kehte hain, jo message temporarily store karta hai.

**Advantages of a Message Broker**  
- Agar recipient unavailable ya overloaded ho to buffer ka kaam karta hai → reliability improve karta hai.
- Crashed process ko automatically messages redeliver kar sakta hai → message loss prevent karta hai.
- Sender ko recipient ka IP address aur port jaanne ki zaroorat nahi hoti (cloud deployments mein useful jahan VMs aate-jaate hain).
- Ek message ko kai recipients ko bhejne deta hai.
- Logically sender ko recipient se decouple karta hai (sender messages publish karta hai,不在乎 kaun consume karta hai).

Communication usually **one-way** hoti hai (sender reply expect nahi karta) aur **asynchronous** (sender delivery ka wait nahi karta).

**Message Brokers**  
- Past: Commercial enterprise software (TIBCO, IBM WebSphere, webMethods).
- Ab: Open source — RabbitMQ, ActiveMQ, HornetQ, NATS, Apache Kafka.

Ek process ek named **queue** ya **topic** mein message bhejta hai; broker use ek ya zyada **consumers/subscribers** tak deliver karta hai.  
Topic one-way dataflow provide karta hai. Consumer doosre topic mein publish kar sakta hai (chaining) ya reply queue mein (request/response pattern).

Message brokers kisi particular data model ko enforce nahi karte — message bas bytes ka sequence hai kuch metadata ke saath. Koi bhi encoding format use karo.  
Agar encoding backward aur forward compatible hai, to publishers aur consumers independently change aur deploy kiye ja sakte hain.

**Distributed Actor Frameworks**  
**Actor model** concurrency ke liye ek programming model hai:

- Logic **actors** mein encapsulated hota hai (threads, race conditions, locking, deadlock se directly deal karne ki jagah).
- Har actor ke paas kuch local state hota hai (shared nahi), doosre actors ke saath asynchronous messages bhej/kar communicate karta hai.
- Message delivery guaranteed nahi hai (error scenarios mein messages lost ho sakte hain).
- Har actor ek time mein sirf ek message process karta hai → threads ke baare mein sochne ki zaroorat nahi.

Distributed actor frameworks mein, same message-passing mechanism multiple nodes par kaam karta hai. Messages transparently encode hote hain, network par bheje jaate hain, aur doosri side decode hote hain.

Location transparency actor model mein RPC se better kaam karta hai, kyunki actor model already assume karta hai ki messages lost ho sakte hain.

Distributed actor framework message broker aur actor programming model ko single framework mein integrate karta hai. Rolling upgrades ke liye ab bhi forward aur backward compatibility chahiye.

**Three Popular Distributed Actor Frameworks**  
- **Akka** – Default mein Java ki built-in serialization use karta hai (no forward/backward compatibility). Rolling upgrades ke liye Protocol Buffers se replace kar sakte ho.
- **Orleans** – Custom encoding format use karta hai jo default mein rolling upgrades support nahi karta. Naya version deploy karne ke liye: naya cluster set up karo, traffic move karo, purana band karo. Custom serialization plug-ins use kar sakte ho.
- **Erlang OTP** – Surprisingly record schemas change karna mushkil hai despite many high-availability features. Rolling upgrades possible hain lekin careful planning chahiye. Experimental maps datatype (Erlang R17, 2014) shayad ise aasan bana de.

---

Summary  
**Key Takeaways**  

**Encoding Formats:**  
- **Language-specific encodings** – Single language tak restricted, aksar compatibility provide nahi karte. Avoid karo.
- **Textual formats (JSON, XML, CSV)** – Widespread, compatibility usage par depend karti hai. Datatypes (numbers, binary strings) ke baare mein vague. Optional schema languages.
- **Binary schema-driven formats (Thrift, Protocol Buffers, Avro)** – Compact, efficient, clearly defined forward aur backward compatibility semantics. Schemas documentation aur code generation ke liye useful hain. Downside: data human-readable hone se pehle decode karna padta hai.

**Encoding Size Comparison:**  

| Format | Bytes |
|--------|-------|
| JSON (textual, no whitespace) | 81 |
| MessagePack (binary JSON) | 66 |
| Thrift BinaryProtocol | 59 |
| Thrift CompactProtocol | 34 |
| Protocol Buffers | 33 |
| Avro | 32 |

**Modes of Dataflow:**  
- **Databases** – Writer encode karta hai, reader decode karta hai. Data code se zyada jeeta hai. Unknown fields preserve karo.
- **RPC and REST APIs** – Client request encode karta hai, server decode karta hai aur response encode karta hai, client response decode karta hai. REST public APIs ke liye predominant; RPC internal services ke liye.
- **Asynchronous message passing (message brokers ya actors)** – Nodes encoded messages bhejkar communicate karte hain. One-way, asynchronous, decoupled.

Care ke saath, backward/forward compatibility aur rolling upgrades kaafi achievable hain.
