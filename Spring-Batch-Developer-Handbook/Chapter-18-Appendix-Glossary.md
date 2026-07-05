# Chapter 18 - Appendix: Glossary

## Introduction
Spring Batch documentation and architecture lo konni specific terms or buzzwords ni chala ekkuva use chestuntaru. Kotha developer ki aa terms meaning exact ga theliyakapothe framework internal ga ela work avthondi ani ardham cheskovadam kashtam. Ee glossary lo Spring Batch loni most commonly used terms and vaati definitions ni telugu lo simplified ga explain chestham.

---

## 1. Core Concepts

### Batch
Oka nirdhistamaina samayam lo (over time) accumulate ayina business transactions ni oka group ga cheyadanni "Batch" antaru.

### Batch Processing
Time tho paatu accumulate ayina pedda data entities ni or transactions ni (e.g. hourly, daily, monthly) okesaari, human intervention (manual elements) lekunda or kevalam error processing ki mathrame manual element unchela predictable ga process cheyadanne "Batch Processing" antaru.

### Batch Window
Oka batch job start ayyi, compulsory ga complete avvalsina time frame ni "Batch Window" antaru. Ee window usually vere dependent jobs start avvadaniki munde close avvali.

### Batch Application Style
Web applications, Microservices, SOA laaga "Batch" kooda oka application style. Deentlo input, validation, business transformation, processing, output, and macro-level monitoring ane standard elements untayi.

---

## 2. Job and Step Components

### Step
Idi batch processing loni main task or oka unit of work. Idi business logic ni initialize chesi, commit interval parameter base chesukoni transaction environment ni control chestundi.

### Tasklet
Oka `Step` loni business logic ni process cheyadaniki developer create chese oka component ne `Tasklet` antaru.


### Batch Job Type
Job types define the typical usage of a batch processing pattern. Common categories include Interface Processing (like parsing and dumping flat files), Forms Processing (like generating massive PDF statements), and Report Processing (aggregating huge data sets for BI reporting).

### Item
Process cheyadaniki panikoche ati chinna (smallest) complete data entity. Idi oka file loni row kavochu, database table loni record kavochu, leda XML loni oka element kavochu.

### Logical Unit of Work (LUW)
Batch job oka input source (like driving query or file) nunchi iterativ ga pani chestundi. Prathi iteration lo chese pani ni oka "unit of work" antaru.

### Commit Interval
Oka single transaction lopu enni Logical Units of Work (LUWs) ni process cheyalo (e.g. 100 items) cheppe counting number ni "Commit Interval" antaru.

---

## 3. Scalability and Patterns

### Partitioning
Oka job loni pedda data ni chinnachi parts (subsets) ga divide chesi, okko part ni okko thread ki icchi parallel ga run cheyadanni "Partitioning" antaru. Ee threads okey JVM lo undochu leda multiple JVMs ki distribute kavochu.

### Driving Query
Oka job yelaanti data meeda pani cheyalo identify chese initial SQL query ne "Driving Query" antaru. Example ki "pending" status unna ID lu anni theesukuravadam. Ah tharavata ah prathi ID okko unit of work (item) avtundi.

### Staging Table
Data ni process chesthunappudu temporary ga hold cheyadaniki vaade table ni "Staging Table" antaru.

---

## 4. Fault Tolerance and Recovery

### Restartable
Fail ayina oka job ni malli run chesinappudu, adhi paatha job identity (same JobInstance ID) ni use cheskunte, ah job ni "Restartable" job antaru.

### Rerunnable
Restartable ayyundi, daani sontha state ni adhe manage chesukune job ni "Rerunnable" antaru (e.g. driving query loni where clause lo `processedFlag != true` ani petti malli run cheyadam).

### Repeat
Batch processing lo idoka basic unit. Oka code block ni error raanantha varaku, leda pani poorthi ayye varaku malli malli pilavadaanni (repeated calling) "Repeat" antaru. Input unnantha sepu adhi repeat avthune untundi.

### Retry
Idi repeat kante koncham different. Okasari fail aina okate input ni theesukuni, konni sarlu (retry limit varaku) malli ah operation ne try cheyadanni "Retry" antaru. Database locks valla occhhe temporary transactional exceptions ki idi baga upayogapaduthundi.

### Recover
Oka exception vachinappudu daanni ela handle cheyali (so that the repeat process doesn't fail entirely) ani cheppe logic ni "Recover" antaru.

### Skip
File validations lanti scenarios lo, oka chedda (bad) input record vachinappudu job fail avvakunda daanni vadilesi munduku vellipoyetanduku vaade recovery strategy ni "Skip" antaru.

## Additional Terminology
- **Chunk**: A logical list of items that is read, optionally processed, and then passed to an ItemWriter in a single transaction.
- **Partitioning**: Splitting a large dataset into smaller bounds that multiple worker nodes can process simultaneously.
