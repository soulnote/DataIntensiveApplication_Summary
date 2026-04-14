
Data models shayad software develop karne ka sabse important hissa hote hain, kyunki inka bahut gahra asar hota hai — sirf is baat par nahi ki software kaise likha jaata hai, balki is baat par bhi ki hum us problem ke baare mein kaise sochte hain jise hum solve kar rahe hain.

Zyadatar applications ek data model ko doosre ke upar layer karke banayi jaati hain. Har layer ke liye, important sawaal yeh hai: use next-lower layer ke terms mein kaise represent kiya jaata hai?

- **Application developer** – Real world ko (log, organizations, goods, actions) objects ya data structures aur APIs ke terms mein model karta hai. Yeh structures aksar application-specific hote hain.
- **General-purpose data model** – Jab un data structures ko store karte hain, to unhe JSON/XML documents, relational database ki tables, ya ek graph model ke roop mein express karte hain.
- **Database engineers** – Decide karte hain ki us JSON/relational/graph data ko memory mein, disk par, ya network par bytes ke terms mein kaise represent kiya jaaye.
- **Hardware engineers** – Bytes ko electrical currents, pulses of light, magnetic fields, etc. ke terms mein represent karte hain.

Har layer apne neeche ki layers ki complexity ko ek clean data model provide karke chhupa deti hai. Yeh abstractions alag-alag groups of people ko effectively saath kaam karne dete hain.

Har data model assumptions embody karta hai ki use kaise use kiya jaayega. Kuch types of usage easy hote hain aur kuch supported nahi hote; kuch operations fast hote hain aur kuch perform badly karte hain.

---

Section 1: Relational Model Versus Document Model  
Aaj ka sabse mashhoor data model SQL hai, jo 1970 mein Edgar Codd dwara propose kiye gaye relational model par based hai: data ko **relations** (SQL mein tables) mein organize kiya jaata hai, jahan har relation **tuples** (SQL mein rows) ka ek unordered collection hota hai.

Relational model ek theoretical proposal tha, aur bahut logon ko shak tha ki kya ise efficiently implement kiya ja sakta hai. Lekin 1980s ke mid tak, RDBMSes aur SQL tools of choice ban gaye.
Relational databases ki dominance lagbhag 25–30 saal tak rahi — computing history mein ek eternity ke barabar.
Iske roots 1960s–70s ke mainframes par business data processing mein hain: transaction processing (sales, banking, airline reservations) aur batch processing (invoicing, payroll, reporting).
Relational model ka goal tha implementation detail ko ek cleaner interface ke peeche chhupana.
Saalon mein, competitors ubhare: network model aur hierarchical model (1970s–80s), object databases (late 1980s–early 1990s), XML databases (early 2000s) — har ek ne hype generate kiya lekin kabhi tike nahi.
Relational databases ne apne original scope se kahin zyada broad variety of use cases mein remarkably acchi tarah generalize kiya (web publishing, social networking, ecommerce, games, SaaS, etc.).

### 1.1 The Birth of NoSQL  
NoSQL relational model ki dominance ko khatam karne ki latest koshish hai. Yeh naam originally 2009 mein ek meetup ke liye ek catchy Twitter hashtag tha jo open source, distributed, nonrelational databases par tha. Baad mein ise retroactively "Not Only SQL" ke roop mein reinterpret kiya gaya.

**NoSQL Adoption ke Peeche Driving Forces**  
1. Relational databases se zyada scalability ki zaroorat — bahut bade datasets ya bahut high write throughput.
2. Commercial database products ke mukable free aur open source software ke liye widespread preference.
3. Specialized query operations jo relational model mein acchi tarah supported nahi hain.
4. Relational schemas ki restrictiveness se frustration, aur ek zyada dynamic aur expressive data model ki ichha.

Alag-alag applications ki alag-alag requirements hoti hain. Aisa lagta hai ki relational databases nonrelational datastores ke saath milkar use hote rahenge — ek idea jise **polyglot persistence** kehte hain.

### 1.2 The Object-Relational Mismatch  
Aaj kal zyadatar application development object-oriented programming languages mein hota hai, jisse ek common criticism ubharta hai: agar data relational tables mein store hota hai, to application code ke objects aur database model of tables, rows, and columns ke beech ek awkward translation layer ki zaroorat hoti hai. Is disconnect ko **impedance mismatch** kehte hain.

ORM frameworks jaise ActiveRecord aur Hibernate boilerplate code kam karte hain lekin dono models ke beech ke differences ko poora nahi chhupa sakte.

**LinkedIn Profile Example**  
Sochiye ki ek résumé (LinkedIn profile) ko relational schema mein kaise express kiya ja sakta hai:
<img width="2880" height="2607" alt="image" src="https://github.com/user-attachments/assets/0fb901dc-5ea8-4969-847d-6db027a041e2" />

Figure 2-1: Representing a LinkedIn profile using a relational schema  
Fields jaise `first_name` aur `last_name` har user ke liye exactly ek baar aate hain → `users` table par columns.
Lekin zyadatar logon ke paas ek se zyada job hoti hai, varying education, aur multiple contact info items → one-to-many relationship.

**One-to-Many Relationships Represent Karne Ke Tareeke**  
- **Traditional SQL (normalized)** – Positions, education, aur contact info ko alag tables mein rakho jinka foreign key reference `users` table ki taraf ho.
- **Structured datatypes / XML / JSON columns** – Baad ke SQL standards ne ek hi row ke andar multi-valued data ke liye support add kiya (Oracle, IBM DB2, MS SQL Server, PostgreSQL, MySQL supported hai).
- **JSON/XML ko text column mein encode karo** – Application khud structure ko interpret kare. Database ko us column ke andar query karne nahi diya ja sakta.

**JSON Representation**  
Ek self-contained document jaise résumé ke liye, JSON kaafi appropriate hai. Document-oriented databases jaise MongoDB, RethinkDB, CouchDB, aur Espresso is model ko support karte hain.

```json
{
  "user_id": 251,
  "first_name": "Bill",
  "last_name": "Gates",
  "summary": "Co-chair of the Bill & Melinda Gates... Active blogger.",
  "region_id": "us:91",
  "industry_id": 131,
  "positions": [
    {"job_title": "Co-chair", "organization": "Bill & Melinda Gates Foundation"},
    {"job_title": "Co-founder, Chairman", "organization": "Microsoft"}
  ],
  "education": [
    {"school_name": "Harvard University", "start": 1973, "end": 1975},
    {"school_name": "Lakeside School, Seattle", "start": null, "end": null}
  ],
  "contact_info": {
    "blog": "http://thegatesnotes.com",
    "twitter": "http://twitter.com/BillGates"
  }
}
```

- JSON application code aur storage layer ke beech impedance mismatch ko kam karta hai.
- JSON ka **locality** multi-table schema se better hota hai — saari relevant information ek jagah hoti hai, ek query kaafi hoti hai (versus multiple queries ya messy multi-way joins).
- One-to-many relationships data mein ek tree structure imply karte hain, aur JSON ise explicit banata hai.
<img width="2880" height="1268" alt="image" src="https://github.com/user-attachments/assets/e1f05a77-f6a4-409b-84ff-c53829c326ea" />

Figure 2-2: One-to-many relationships forming a tree structure

### 1.3 Many-to-One aur Many-to-Many Relationships  
JSON example mein, `region_id` aur `industry_id` IDs ke roop mein diye gaye hain, plain-text strings nahi. Kyun?

**IDs Use Karne Ke Fayde (Standardized Lists)**  
- Consistent style aur spelling across profiles.
- Ambiguity avoid karna (e.g., ek hi naam ke kai cities).
- Update karna easy — naam sirf ek jagah stored hota hai.
- Localization support — standardized lists translate ki ja sakti hain.
- Better search — e.g., Washington mein philanthropists ki search Seattle profiles ko match kar sakti hai.

**Normalization**  
Jab aap ID use karte ho, to human-meaningful information sirf ek jagah stored hoti hai, aur baaki sab cheezein usi ID se refer karti hain.
Jab aap text directly store karte ho, to aap human-meaningful information ko har record mein duplicate kar rahe ho.
ID ka fayda: iska humans ke liye koi meaning nahi hota, isliye kabhi change karne ki zaroorat nahi padti. Jo cheez humans ke liye meaningful ho, use change karna pad sakta hai, aur agar duplicate hai to saari copies update karni padti hain → inconsistencies ka risk.
Aise duplication ko hatana hi databases mein **normalization** ka key idea hai.

**The Problem with Document Databases**  
Data normalize karne ke liye **many-to-one relationships** ki zaroorat hoti hai (bahut log ek region mein rehte hain, bahut log ek industry mein kaam karte hain), jo document model mein nicely fit nahi hote.
Relational databases mein, ID se rows refer karna aur join karna easy hai. Document databases mein, joins aksar weak hote hain ya supported hi nahi hote.
Agar database joins support nahi karta, to aapko application code mein multiple queries karke joins emulate karne padte hain.

**Data Time Ke Saath Zyada Interconnected Ho Jaata Hai**  
Agar initial version join-free document model mein fit bhi ho jaata hai, to jaise jaise features add hote hain, data zyada interconnected hota jaata hai:

- **Organizations aur schools as entities** – Sirf strings nahi, balki entities ke references jinke apne web pages, logos, news feeds hote hain.
<img width="1009" height="834" alt="image" src="https://github.com/user-attachments/assets/7b9bfa2b-0e8d-4711-8940-6ae9c089e1a1" />

  - Figure 2-3: Company name sirf string nahi, balki company entity ka link hai.
- **Recommendations** – Ek user doosre user ke liye recommendation likhta hai. Recommendation ko author ke profile ko reference karna chahiye (taaki agar wo apna photo update kare to har jagah reflect ho).
<img width="1012" height="694" alt="image" src="https://github.com/user-attachments/assets/8b6ba01d-5ed8-4624-bf65-1b76bc56fbce" />

  - Figure 2-4: Extending résumés with many-to-many relationships

### 1.4 Are Document Databases Repeating History?  
Many-to-many relationships ke baare mein debate NoSQL se kahin zyada purani hai — yeh earliest computerized database systems tak jaati hai.

**The Hierarchical Model (IMS)**  
1970s mein business data processing ke liye sabse popular database IBM ka **Information Management System (IMS)** tha, jo originally Apollo space program mein stock-keeping ke liye develop kiya gaya tha (pehli baar commercially 1968 mein release hua, aaj bhi IBM mainframes par use hota hai).
IMS **hierarchical model** use karta tha: saara data records ke andar nested records ka tree — remarkably similar to JSON used by document databases.
One-to-many relationships ke liye accha kaam karta tha, lekin many-to-many relationships ko mushkil bana deta tha aur joins support nahi karta tha.
Developers ko decide karna padta tha ki data duplicate karein (denormalize) ya manually references resolve karein — wahi problems jo aaj developers ko document databases ke saath face karni padti hain.

**The Network Model (CODASYL)**  
**Conference on Data Systems Languages (CODASYL)** dwara standardize kiya gaya.
Hierarchical model ka generalization: ek record ke multiple parents ho sakte the (jisse many-to-one aur many-to-many relationships allow hue).
Records ke beech links programming language ke pointers ki tarah the (disk par stored). Ek record access karne ka ek hi tareeka tha: ek root record se chain of links ke through follow karna — ise **access path** kehte the.
Query cursor ko database mein move karte hue, lists iterate karte hue, aur access paths follow karte hue perform ki jaati thi.
CODASYL committee members ne khud admit kiya ki yeh "navigating around an n-dimensional data space" jaisa tha.
Query aur update karne ka code complicated aur inflexible tha. Agar aapke paas required data tak pahunchne ka path nahi tha, to aap phas gaye. Access path change karne ke liye handwritten query code dobara likhna padta tha.

**The Relational Model**  
Saara data khullam khulla rakh diya: relation (table) bas tuples (rows) ka collection hai, aur bas itna hi.
Koi labyrinthine nested structures nahi, koi complicated access paths nahi.
Aap koi bhi ya saari rows padh sakte ho, unhe select karke jo arbitrary condition match karein.
**Query optimizer** automatically decide karta hai ki query ke kaunse parts kis order mein execute hone chahiye, aur kaunse indexes use hone chahiye — effectively "access path", lekin automatically banaaya gaya, developer ke dwara nahi.
Agar aap naye tareeke se data query karna chahte ho, to bas naya index declare karo — queries automatically most appropriate indexes use karengi.
**Key insight**: query optimizer ek baar build karo, aur saari applications ko fayda hota hai. General-purpose optimizer long run mein handcoded access paths se jeet jaata hai.

**Comparison to Document Databases**  
Document databases ek aspect mein hierarchical model ki taraf laut gaye: nested records ko parent record ke andar store karna rather than alag table mein.
Lekin many-to-one aur many-to-many relationships ke liye, relational aur document databases fundamentally different nahi hain: dono ek unique identifier (foreign key vs document reference) use karte hain, jo read time par join ya follow-up queries se resolve hota hai.
Document databases ne CODASYL ka raasta nahi apnaya.

### 1.5 Relational Versus Document Databases Today  
Main arguments:

| Document Model | Relational Model |
|----------------|------------------|
| **Strengths** | Schema flexibility, better performance due to locality, closer to application data structures | Better support for joins, many-to-one aur many-to-many relationships |

**Kaunsa Data Model Simpler Application Code Ki Taraf Le Jaata Hai?**  
- Agar data document-like structure hai (tree of one-to-many relationships, poori tree ek saath load hoti hai) → document model use karo.
- Relational technique of **shredding** (document-like structure ko multiple tables mein todna) cumbersome schemas aur complicated application code ki taraf le ja sakti hai.
- Document model ki limitation: aap directly nested item ke andar refer nahi kar sakte — aapko kuch aisa kehna padta hai jaise "user 251 ki positions list ka doosra item" (bilkul hierarchical model ke access path jaisa).
- Agar aapki application bahut saari many-to-many relationships use karti hai, to document model kam appealing ho jaata hai. Joins ko application code mein emulate kiya ja sakta hai, lekin wo complexity application mein move kar deta hai aur usually database ke andar join se slower hota hai.
- **Highly interconnected data** ke liye: document model awkward hai, relational model acceptable hai, graph models sabse natural hain.

**Schema Flexibility in the Document Model**  
Document databases ko kabhi kabhi **schemaless** kaha jaata hai, lekin yeh misleading hai — jo code data padhta hai wo usually kuch structure assume karta hai (implicit schema).
Zayada accurate terms:
- **Schema-on-read** – Structure implicit hoti hai, sirf tab interpret hoti hai jab data padha jaata hai (jaise dynamic type checking).
- **Schema-on-write** – Schema explicit hoti hai, database ensure karta hai ki saara likha gaya data uske conform kare (jaise static type checking).

**Example: Changing Data Format**  
- **Document database (schema-on-read):**
  ```javascript
  if (user && user.name && !user.first_name) {
      // Documents written before Dec 8, 2013 don't have first_name
      user.first_name = user.name.split(" ")[0];
  }
  ```
- **Relational database (schema-on-write):**
  ```sql
  ALTER TABLE users ADD COLUMN first_name text;
  UPDATE users SET first_name = split_part(name, ' ', 1);      -- PostgreSQL
  UPDATE users SET first_name = substring_index(name, ' ', 1);  -- MySQL
  ```
Schema changes ki reputation buri hai ki wo slow hote hain, lekin zyadatar relational databases `ALTER TABLE` ko kuch milliseconds mein execute kar dete hain. **MySQL ek notable exception hai** — yeh poori table copy karta hai, jiska matlab minutes ya hours of downtime ho sakta hai (tools jaise `pt-online-schema-change` aur `gh-ost` is problem ko work around karne ke liye available hain).

**Jab Schema-on-Read Advantageous Hota Hai**  
- Collection mein items sabki same structure nahi hoti (heterogeneous data):
  - Bahut saare different types of objects, practical nahi ki har type ko apni table mein daala jaaye.
  - Structure external systems determine karte hain jinko aap control nahi karte, jo kabhi bhi change ho sakte hain.

**Data Locality for Queries**  
Document usually ek single continuous string ki tarah store hota hai (JSON, XML, ya binary variant jaise MongoDB ka BSON).
Agar aapki application ko aksar poori document ki zaroorat hoti hai, to is storage locality ka performance advantage hota hai (versus multiple tables ke across multiple index lookups).
Locality advantage sirf tab apply hota hai jab aapko document ke bade hisse ek saath chahiye hote hain. Database typically poori document load karta hai, chahe aap sirf chhota sa hissa access kar rahe ho.
Updates par, poori document usually dobara likhni padti hai — sirf modifications jo encoded size change nahi karte, unhe in-place perform kiya ja sakta hai.
**Recommendation**: documents ko fairly chhota rakho aur aise writes avoid karo jo document size badhaate hain.

**Locality sirf document model tak seemit nahi hai:**
- Google's Spanner — schema allow karta hai declare karne ki table ki rows parent table ke andar interleaved (nested) honi chahiyein.
- Oracle — multi-table index cluster tables.
- Bigtable model (Cassandra, HBase) — column-family concept for managing locality.

**Convergence of Document and Relational Databases**  
- Zyadatar relational databases mid-2000s se XML support karte hain, aur recently JSON (PostgreSQL 9.3+, MySQL 5.7+, IBM DB2 10.5+).
- Document side par, RethinkDB relational-like joins support karta hai; kuch MongoDB drivers automatically database references resolve karte hain (client-side join).
- Relational aur document databases time ke saath zyada similar ho rahe hain — yeh data models ek doosre ke complement hain.
- Relational aur document models ka hybrid future mein databases ke liye ek accha raasta hai.

---

Section 2: Query Languages for Data  
Jab relational model introduce hua tha, to usne data query karne ka ek naya tareeka diya: SQL ek **declarative query language** hai, jabki IMS aur CODASYL database ko **imperative code** se query karte the.

### 2.1 Declarative vs Imperative  

**Imperative Example (JavaScript)**
```javascript
function getSharks() {
    var sharks = [];
    for (var i = 0; i < animals.length; i++) {
        if (animals[i].family === "Sharks") {
            sharks.push(animals[i]);
        }
    }
    return sharks;
}
```

**Relational Algebra**
```
sharks = σ_family="Sharks" (animals)
```

**Declarative (SQL)**
```sql
SELECT * FROM animals WHERE family = 'Sharks';
```

**Declarative Kyun Better Hai**  
- Zyada concise aur imperative APIs ke muqable kaam karne mein aasan.
- Database engine ki implementation details chhupa leta hai — database performance improvements introduce kar sakta hai bina queries change kiye.
- **Order independence** — SQL kisi bhi particular ordering ki guarantee nahi deta, isliye database data rearrange karne ke liye free hai (e.g., disk space reclaim) bina queries tode. Imperative code ordering par rely kar sakta hai.
- **Parallel execution ke liye better** — Declarative languages sirf results ka pattern specify karte hain, algorithm nahi. Database parallel implementation use karne ke liye free hai. Imperative code instructions ek particular order mein specify karta hai, jisse parallelize karna bahut mushkil ho jaata hai.

### 2.2 Declarative Queries on the Web  
Declarative languages ke fayde sirf databases tak seemit nahi hain. CSS vs imperative JavaScript for styling ko dekho:

**Declarative (CSS)**
```css
li.selected > p {
    background-color: blue;
}
```

**Imperative (JavaScript DOM)**
```javascript
var liElements = document.getElementsByTagName("li");
for (var i = 0; i < liElements.length; i++) {
    if (liElements[i].className === "selected") {
        var children = liElements[i].childNodes;
        for (var j = 0; j < children.length; j++) {
            var child = children[j];
            if (child.nodeType === Node.ELEMENT_NODE && child.tagName === "P") {
                child.setAttribute("style", "background-color: blue");
            }
        }
    }
}
```

**Imperative approach ke problems:**
- Agar `selected` class remove ho jaaye, to blue color remove nahi hoga (chahe code dobara run karein) — CSS automatically handle karta hai.
- Agar aap better performance ke liye naya API use karna chahein, to aapko code dobara likhna padega. Browser vendors CSS/XPath performance improve kar sakte hain bina compatibility tode.
- Isi tarah, databases mein declarative query languages jaise SQL, imperative query APIs (jaise IMS aur CODASYL use karte the) se kahin zyada better sabit hue.

### 2.3 MapReduce Querying  
**MapReduce** ek programming model hai jo bulk mein large amounts of data ko bahut saari machines par process karne ke liye hai, Google dwara popularized. Kuch NoSQL datastores (MongoDB, CouchDB) iska limited form support karte hain.

MapReduce na to fully declarative hai aur na hi fully imperative — kahin beech mein.
Functional programming ke **map** (collect) aur **reduce** (fold/inject) functions par based.

**Example: Counting Shark Sightings Per Month**

**PostgreSQL (declarative):**
```sql
SELECT date_trunc('month', observation_timestamp) AS observation_month,
       sum(num_animals) AS total_animals
FROM observations
WHERE family = 'Sharks'
GROUP BY observation_month;
```

**MongoDB MapReduce:**
```javascript
db.observations.mapReduce(
    function map() {
        var year  = this.observationTimestamp.getFullYear();
        var month = this.observationTimestamp.getMonth() + 1;
        emit(year + "-" + month, this.numAnimals);
    },
    function reduce(key, values) {
        return Array.sum(values);
    },
    {
        query: { family: "Sharks" },
        out: "monthlySharkReport"
    }
);
```

- `map` aur `reduce` functions **pure functions** hone chahiye (no side effects, no additional database queries) — isse database unhe kahin bhi, kisi bhi order mein run kar sakta hai, aur failure par rerun kar sakta hai.
- MapReduce distributed execution ke liye ek fairly low-level programming model hai. Higher-level query languages jaise SQL ko MapReduce operations ke pipeline ki tarah implement kiya ja sakta hai.

**Usability Problem**  
- Do carefully coordinated JavaScript functions likhna aksar ek single query likhne se zyada mushkil hota hai.
- Declarative query language query optimizer ko performance improve karne ke zyada opportunities deti hai.

**MongoDB Aggregation Pipeline**  
Inhi reasons ki wajah se, MongoDB 2.2 ne **aggregation pipeline** introduce kiya — ek declarative query language:

```javascript
db.observations.aggregate([
    { $match: { family: "Sharks" } },
    { $group: {
        _id: {
            year:  { $year:  "$observationTimestamp" },
            month: { $month: "$observationTimestamp" }
        },
        totalAnimals: { $sum: "$numAnimals" }
    } }
]);
```

- Expressiveness mein SQL ke subset ke similar, lekin JSON-based syntax use karta hai.
- **Moral**: NoSQL system accidentally SQL reinvent kar sakta hai, chahe disguise mein hi sahi.

---

Section 3: Graph-Like Data Models  
Agar aapki application mein mostly one-to-many relationships hain (tree-structured data) ya records ke beech koi relationships nahi hain, to document model appropriate hai. Lekin agar many-to-many relationships bahut common hain, to data ko **graph** ki tarah model karna zyada natural ho jaata hai.

Graph do types ke objects se banta hai:
- **Vertices** (nodes ya entities)
- **Edges** (relationships ya arcs)

**Typical Examples**  
- **Social graphs** – Vertices log hain, edges indicate karte hain ki kaun kise jaanta hai.
- **Web graph** – Vertices web pages hain, edges HTML links hain.
- **Road ya rail networks** – Vertices junctions hain, edges roads ya railway lines hain.

Graphs par kaam karne wale mashhoor algorithms: car navigation systems shortest path dhundhte hain, PageRank search rankings ke liye web page popularity determine karta hai.

Graphs sirf homogeneous data tak seemit nahi hain. Utna hi powerful use hai completely different types of objects ko ek hi datastore mein store karna. For example, Facebook ek single graph maintain karta hai jismein vertices log, locations, events, checkins, aur comments ko represent karte hain; edges friendships, checkin locations, comments on posts, event attendance, etc.
<img width="2880" height="1731" alt="image" src="https://github.com/user-attachments/assets/d4850b69-03c0-4feb-9d03-0d134b63fd5b" />

Figure 2-5: Example of graph-structured data (boxes represent vertices, arrows represent edges)  
Example mein do log dikhaye gaye hain: Lucy from Idaho aur Alain from Beaune, France. Wo married hain aur London mein rehte hain. Yeh cheezein traditional relational schema mein express karna mushkil hota hai: different countries mein different kinds of regional structures (France mein départements aur régions, US mein counties aur states), varying granularity of data (Lucy ka residence city hai, birthplace state hai).

**Graphs evolvability ke liye acche hote hain**: jaise aap features add karte ho, graph easily extend ho sakta hai changes accommodate karne ke liye.

### 3.1 Property Graphs  
**Property graph model** (Neo4j, Titan, InfiniteGraph dwara implement) mein har vertex yeh cheezein rakhta hai:
- Ek unique identifier
- Outgoing edges ka set
- Incoming edges ka set
- Properties ka collection (key-value pairs)

Har edge yeh cheezein rakhta hai:
- Ek unique identifier
- Vertex jahan se edge start hoti hai (**tail vertex**)
- Vertex jahan par edge end hoti hai (**head vertex**)
- Ek label jo relationship ke type ko describe kare
- Properties ka collection (key-value pairs)

**Relational Representation of a Property Graph**
```sql
CREATE TABLE vertices (
    vertex_id   integer PRIMARY KEY,
    properties  json
);

CREATE TABLE edges (
    edge_id     integer PRIMARY KEY,
    tail_vertex integer REFERENCES vertices (vertex_id),
    head_vertex integer REFERENCES vertices (vertex_id),
    label       text,
    properties  json
);

CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
```

**Important Aspects**  
- Koi bhi vertex kisi bhi doosre vertex ke saath edge rakh sakta hai — koi schema restrict nahi karta ki kaunsi cheezein associate ho sakti hain.
- Kisi bhi vertex se, aap efficiently uske incoming aur outgoing edges dhundh sakte ho, aur is tarah graph ko aage aur peeche traverse kar sakte ho (isliye `tail_vertex` aur `head_vertex` dono par indexes).
- Different kinds of relationships ke liye alag labels use karke, aap ek hi graph mein kai tarah ki information store kar sakte ho, saath hi clean data model maintain karte hue.

### 3.2 The Cypher Query Language  
**Cypher** property graphs ke liye ek declarative query language hai, Neo4j graph database ke liye create ki gayi. (Naam *The Matrix* ke ek character se liya gaya, cryptography se related nahi hai.)

**Data Insert Karna**
```cypher
CREATE
  (NAmerica:Location {name:'North America', type:'continent'}),
  (USA:Location      {name:'United States', type:'country'}),
  (Idaho:Location    {name:'Idaho',         type:'state'}),
  (Lucy:Person       {name:'Lucy'}),
  (Idaho) -[:WITHIN]->  (USA)  -[:WITHIN]-> (NAmerica),
  (Lucy)  -[:BORN_IN]-> (Idaho)
```
Arrow notation: `(Idaho) -[:WITHIN]-> (USA)` ek edge create karta hai jiska label `WITHIN` hai, jahan Idaho tail node hai aur USA head node.

**Query Karna: US se Europe Emigrate Karne Wale Log Dhundho**
```cypher
MATCH
  (person) -[:BORN_IN]->  () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
  (person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'})
RETURN person.name
```
- `:WITHIN*0..` ka matlab hai "`WITHIN` edge ko zero ya zyada baar follow karo" (jaise regular expression mein `*`).
- Query optimizer automatically decide karta hai sabse efficient execution strategy — aap saare logon ko scan karke start kar sakte ho, ya Location vertices se start karke backward kaam kar sakte ho.

### 3.3 Graph Queries in SQL  
Graph data ko relational database mein represent kiya ja sakta hai, lekin SQL se query karna mushkil hai kyunki joins ki number fix nahi hoti (aapko variable number of edges traverse karni pad sakti hai).

SQL:1999 se, ise **recursive common table expressions** (`WITH RECURSIVE`) se express kiya ja sakta hai. Wahi emigration query SQL mein:

```sql
WITH RECURSIVE
  in_usa(vertex_id) AS (
      SELECT vertex_id FROM vertices WHERE properties->>'name' = 'United States'
    UNION
      SELECT edges.tail_vertex FROM edges
        JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
        WHERE edges.label = 'within'
  ),
  in_europe(vertex_id) AS (
      SELECT vertex_id FROM vertices WHERE properties->>'name' = 'Europe'
    UNION
      SELECT edges.tail_vertex FROM edges
        JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
        WHERE edges.label = 'within'
  ),
  born_in_usa(vertex_id) AS (
    SELECT edges.tail_vertex FROM edges
      JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
      WHERE edges.label = 'born_in'
  ),
  lives_in_europe(vertex_id) AS (
    SELECT edges.tail_vertex FROM edges
      JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
      WHERE edges.label = 'lives_in'
  )
SELECT vertices.properties->>'name'
FROM vertices
JOIN born_in_usa     ON vertices.vertex_id = born_in_usa.vertex_id
JOIN lives_in_europe ON vertices.vertex_id = lives_in_europe.vertex_id;
```

Agar wahi query Cypher mein 4 lines mein likhi ja sakti hai lekin SQL mein 29 lines lagti hain, to yeh dikhata hai ki alag data models alag use cases satisfy karne ke liye design kiye gaye hain. Apne application ke liye suitable data model chuno.

### 3.4 Triple-Stores aur SPARQL  
**Triple-store model** mostly property graph model ke equivalent hai, bas alag words use karta hai same ideas ke liye.

Triple-store mein, saari information three-part statements ke roop mein store hoti hai: **(subject, predicate, object)**.

- Example: `(Jim, likes, bananas)` — Jim subject hai, `likes` predicate hai, `bananas` object hai.
- Subject vertex ke equivalent hai. Object do cheezon mein se ek hota hai:
  1. Ek primitive value (string, number) — predicate aur object subject vertex ki property ke key-value pair ke equivalent hote hain. E.g., `(lucy, age, 33)` → vertex `lucy` with `{"age": 33}`.
  2. Doosra vertex — predicate ek edge hai, subject tail vertex hai, object head vertex hai. E.g., `(lucy, marriedTo, alain)`.

**Turtle Format (Example)**
```turtle
@prefix : <urn:example:>.
_:lucy     a :Person;   :name "Lucy";          :bornIn _:idaho.
_:idaho    a :Location; :name "Idaho";         :type "state";   :within _:usa.
_:usa      a :Location; :name "United States"; :type "country"; :within _:namerica.
_:namerica a :Location; :name "North America"; :type "continent".
```

**The Semantic Web and RDF**  
- **Semantic web idea**: websites machine-readable data publish karein (sirf humans ke liye text/pictures nahi), jisse alag websites ka data combine karke "web of data" banaya ja sake.
- **Resource Description Framework (RDF)** is mechanism ke liye intended tha.
- Semantic web 2000s ke early mein overhyped tha aur practice mein realize nahi hua, lekin triples ek accha internal data model ho sakta hai applications ke liye, regardless.
- RDF subjects, predicates, aur objects ke liye **URIs** use karta hai taaki different sources se data combine karte waqt naming conflicts na ho.

**The SPARQL Query Language**  
**SPARQL** (pronounced "sparkle") triple-stores ke liye ek query language hai jo RDF data model use karta hai. Yeh Cypher se pehle aaya, aur Cypher ki pattern matching SPARQL se li gayi hai.

```sparql
PREFIX : <urn:example:>
SELECT ?personName WHERE {
  ?person :name ?personName.
  ?person :bornIn  / :within* / :name "United States".
  ?person :livesIn / :within* / :name "Europe".
}
```

**Comparison with Cypher:**
```
(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (location)   # Cypher
?person :bornIn / :within* ?location.                    # SPARQL
```

SPARQL applications ke internal use ke liye ek powerful tool hai, chahe semantic web kabhi na ho.

### 3.5 Graph Databases vs. the Network Model (CODASYL)  
Pehli nazar mein, CODASYL ka network model graph databases jaisa lagta hai. Lekin important differences hain:

| CODASYL | Graph Databases |
|---------|-----------------|
| **Schema** | Schema specify karta tha kaunse record types kis ke andar nested ho sakte hain | Koi bhi vertex kisi bhi doosre vertex ke saath edge rakh sakta hai — koi restrictions nahi |
| **Access** | Record tak pahunchne ka ek hi tareeka: access path traverse karna | Kisi bhi vertex ko directly unique ID se refer kar sakte ho, ya index use kar sakte ho |
| **Ordering** | Record ke children ek ordered set the | Vertices aur edges unordered hote hain (sirf query time par sort kiya ja sakta hai) |
| **Query style** | Saari queries imperative thi, likhna mushkil, schema changes se easily break ho jaati thi | High-level declarative languages support karte hain (Cypher, SPARQL) |

### 3.6 The Foundation: Datalog  
**Datalog** SPARQL ya Cypher se kahin purani language hai, 1980s mein academics dwara extensively study ki gayi. Yeh foundation provide karta hai jispar baad ki query languages build hui hain. Datomic aur Cascalog (Hadoop mein large datasets query karne ke liye Datalog implementation) mein use hota hai.

Datalog ka data model triple-store model jaisa hai, lekin `(subject, predicate, object)` ki jagah hum likhte hain `predicate(subject, object)`.

**Data as Datalog Facts**
```datalog
name(namerica, 'North America').
type(namerica, continent).
name(usa, 'United States').
type(usa, country).
within(usa, namerica).
name(idaho, 'Idaho').
type(idaho, state).
within(idaho, usa).
name(lucy, 'Lucy').
born_in(lucy, idaho).
```

**Query Using Rules**
```datalog
within_recursive(Location, Name) :- name(Location, Name).     /* Rule 1 */
within_recursive(Location, Name) :- within(Location, Via),     /* Rule 2 */
                                    within_recursive(Via, Name).

migrated(Name, BornIn, LivingIn) :- name(Person, Name),       /* Rule 3 */
                                    born_in(Person, BornLoc),
                                    within_recursive(BornLoc, BornIn),
                                    lives_in(Person, LivingLoc),
                                    within_recursive(LivingLoc, LivingIn).

?- migrated(Who, 'United States', 'Europe').
/* Who = 'Lucy'. */
```
<img width="2880" height="922" alt="image" src="https://github.com/user-attachments/assets/e81e5c99-c89c-4501-a123-491d6cd0701c" />

Figure 2-6: Determining that Idaho is in North America, using the Datalog rules

**Rules Kaise Kaam Karte Hain**  
- Hum rules define karte hain jo database ko naye predicates ke baare mein batate hain (`within_recursive` aur `migrated`). Yeh data se ya doosre rules se derive hote hain.
- Rules doosre rules ko refer kar sakte hain, bilkul jaise functions doosre functions ko call karte hain ya recursively khud ko call karte hain.
- Ek rule apply hota hai agar system `:-` operator ke righthand side ke saare predicates ke liye match dhundh sake.

Rule 1 aur Rule 2 ke repeated application se, `within_recursive` hume bata sakta hai ki kisi bhi named location ke andar kaunse locations aate hain.

**Step-by-step Application**
1. `name(namerica, 'North America')` exists → Rule 1 generates `within_recursive(namerica, 'North America')`.
2. `within(usa, namerica)` exists + previous step → Rule 2 generates `within_recursive(usa, 'North America')`.
3. `within(idaho, usa)` exists + previous step → Rule 2 generates `within_recursive(idaho, 'North America')`.

Datalog approach alag tarah ki soch maangta hai, lekin bahut powerful hai kyunki rules combine aur reuse kiye ja sakte hain different queries mein. Simple one-off queries ke liye kam convenient hai, lekin complex data ke saath better cope karta hai.

---

Summary  
**Key Takeaways**  
- Historically, data ek bade tree (hierarchical model) ki tarah shuru hua, lekin wo many-to-many relationships ke liye accha nahi tha, isliye relational model invent hua.
- Recently, NoSQL datastores do directions mein diverge hue:
  - **Document databases** – Un use cases ko target karte hain jahan data self-contained documents mein aata hai aur documents ke beech relationships rare hote hain.
  - **Graph databases** – Un use cases ko target karte hain jahan kuch bhi potentially kisi bhi cheez se related ho sakta hai.
- Teenon models (document, relational, graph) aaj widely use hote hain, har ek apne respective domain mein accha hai. Ek model ko doosre mein emulate kiya ja sakta hai, lekin result aksar awkward hota hai — isliye humare paas alag-alag purposes ke liye alag systems hain.
- Document aur graph databases typically schema enforce nahi karte, jisse changing requirements ke saath adapt karna aasan hota hai. Lekin application phir bhi structure assume karta hai — yeh sawaal hai ki schema explicit hai (write par enforce) ya implicit (read par handle).
- Har data model apni query language ke saath aata hai: SQL, MapReduce, MongoDB aggregation pipeline, Cypher, SPARQL, aur Datalog. CSS aur XSL/XPath ke interesting parallels hain.
- Doosre specialized data models bhi exist karte hain: genome databases (GenBank), particle physics (LHC), full-text search indexes.
