# Oracle 1Z0-071 — 10-Bob Konspekti
## Schema Ob'ektlarini Boshqarish (Managing Schema Objects)

Konspekt faqat imtihonda tekshiriladigan bilimlarga qaratilgan: qoidalar, istisno holatlar (edge cases) va sintaktik tuzoqlar. Qo'shimcha ma'lumotlar Oracle **19c** bo'yicha berilgan.

---

## 1-Qism: Schema Ob'ektlari va Namespace'lar

**Schema (sxema)** — bitta foydalanuvchi hisobiga tegishli ob'ektlar to'plami; sxema nomi foydalanuvchi nomi bilan bir xil.

**Schema ob'ektlari** (sxemaga tegishli): Table, View, Index, Sequence, Synonym (private), PL/SQL (procedure, function, package), Trigger va h.k.
**Nonschema ob'ektlari** (baza darajasida): User, Role, Tablespace, Public synonym, Directory, Profile.

### Namespace qoidasi (EXAM CRITICAL)

Bir sxema ichida ba'zi ob'ektlar **umumiy namespace**ni baham ko'radi — ular bir xil nomga ega bo'la olmaydi:

| Umumiy namespace (bir xil nom MUMKIN EMAS) | Alohida namespace (bir xil nom MUMKIN) |
| :--- | :--- |
| Table, View, Sequence, Private synonym, PL/SQL (procedure/function/package) | **Index** — alohida |
| | **Constraint** — alohida |

* Bir sxemada `EMPLOYEES` nomli Table va `EMPLOYEES` nomli View **bo'lmaydi** → `ORA-00955: name is already used by an existing object`.
* Lekin `VENDORS` nomli Table, `VENDORS` nomli Index va `VENDORS` nomli Constraint **bir vaqtda bo'la oladi** (har biri boshqa namespace'da).

### Identifikator (nom) qoidalari

* 1–30 bayt (Oracle **12.2+ va 19c**: **128 baytgacha**).
* Harf bilan boshlanadi; keyin harf, raqam, `_`, `$`, `#` bo'lishi mumkin.
* Reserved so'z bo'lmasligi kerak (masalan `SELECT`, `NUMBER` — jadval nomi sifatida xato).
* **Quoted identifier** (`"My Table"`) — bo'shliq, maxsus belgi va kичik/katta harf sezgirligini saqlaydi, lekin keyin har doim qo'shtirnoq bilan chaqirish kerak.

---

## 2-Qism: View'lar (10.02)

**View** — bazada nom bilan saqlangan `SELECT` so'rovi. O'zida ma'lumot saqlamaydi; asos jadvallardan (base tables) ma'lumotni dinamik oladi.

| Tur | Ta'rifi |
| :--- | :--- |
| **Simple view** | 1 jadval, `GROUP BY`/agregat/`DISTINCT` yo'q → odatda to'liq DML qilinadi |
| **Complex view** | Join, `GROUP BY`, agregat, `DISTINCT`, set operator → DML cheklangan |
| **Inline view** | `FROM` dagi nomsiz subquery (9-bobga qarang) |

### Sintaksis

```sql
CREATE [OR REPLACE] [FORCE | NOFORCE] VIEW view_name [(alias_list)]
AS subquery
[WITH CHECK OPTION [CONSTRAINT constr_name]]
[WITH READ ONLY   [CONSTRAINT constr_name]];
```

* **`OR REPLACE`** — mavjud view'ni ogohlantirishsiz qayta yozadi. (`CREATE TABLE`da `OR REPLACE` **yo'q**!)
* **`FORCE`** — asos jadval mavjud bo'lmasa yoki huquq bo'lmasa ham view'ni yaratadi; view `INVALID` holatda bo'ladi.
* **`NOFORCE`** (default) — asos jadval/huquq bo'lmasa, xato beradi.
* `ORDER BY` view ichida **ruxsat etiladi**.

### View orqali DML qilish qoidalari

View orqali `INSERT`/`UPDATE`/`DELETE` qilish uchun view **updatable** bo'lishi kerak. Quyidagilar bo'lsa DML (yoki ba'zi ustunlar) **taqiqlanadi**:

| Element | Natija |
| :--- | :--- |
| `GROUP BY`, agregat funksiya, `DISTINCT` | DML umuman mumkin emas |
| Set operator (`UNION`, `MINUS`, `INTERSECT`) | DML mumkin emas |
| `ROWNUM` pseudocolumn | DML mumkin emas |
| Hisoblangan ustun (`salary*12`) | o'sha ustunni `UPDATE`/`INSERT` qilib bo'lmaydi |
| Join (bir nechta jadval) | faqat **key-preserved** jadval ustunlariga DML mumkin |

* **`INSERT` muhim shart:** asos jadvalning `NOT NULL` (va `DEFAULT`siz) ustunlari view select-ro'yxatida bo'lishi shart; aks holda → `ORA-01400: cannot insert NULL`.
* **Key-preserved table:** join-view'da PK/UNIQUE'i natijada ham noyob qoladigan jadval. DML faqat shunga tegadi.

### WITH CHECK OPTION va WITH READ ONLY

* **`WITH CHECK OPTION`** — view orqali `INSERT`/`UPDATE` qilinayotgan qator view'ning `WHERE` shartiga mos bo'lishini majbur qiladi. Mos kelmasa → `ORA-01402: view WITH CHECK OPTION where-clause violation`.
* **`WITH READ ONLY`** — view orqali har qanday DML'ni taqiqlaydi → `ORA-42399: cannot perform a DML operation on a read-only view`.

### View va Invisible ustunlar (EXAM TRAP)

* `CREATE VIEW v AS SELECT * FROM t;` — asos jadvaldagi **INVISIBLE ustun view'ga KIRMAYDI** (`*` invisible ustunni olmaydi).
* `CREATE VIEW v AS SELECT invis_col, ... FROM t;` — invisible ustunni **nomi bilan** yozsangiz, u view'da **oddiy VISIBLE ustunga aylanadi**.

### Boshqa

* **`ALTER VIEW v COMPILE;`** — asos jadval o'zgarib view `INVALID` bo'lsa, qayta kompilyatsiya qiladi.
* **`DROP VIEW v;`** — view'ni o'chiradi; asos jadvalga ta'sir qilmaydi.

> **⚠️ Asos jadval o'zgarsa:** asos jadval `DROP` qilinsa yoki ishlatilayotgan ustun o'chirilsa, view **avtomatik o'chmaydi** — `INVALID` holatga tushadi va chaqirilganda xato beradi:
> ```sql
> DROP TABLE employees;
> SELECT * FROM emp_view;   -- ORA-04063: view "EMP_VIEW" has errors
> ```
> Jadval qayta yaratilsa, view keyingi chaqiruvda avtomatik (yoki `ALTER VIEW ... COMPILE` bilan) qayta kompilyatsiya bo'ladi.

---

## 3-Qism: Sequence'lar (10.03)

**Sequence** — noyob, ketma-ket raqamlar generatori. Hech qaysi jadvalga bog'lanmagan; bir nechta jadval foydalanishi mumkin.

### Sintaksis va default qiymatlar

```sql
CREATE SEQUENCE seq_name
  [START WITH n]        -- default: 1 (o'suvchi), -1 (kamayuvchi)
  [INCREMENT BY n]      -- default: 1 (manfiy bo'lsa kamayuvchi)
  [MAXVALUE n | NOMAXVALUE]   -- default: NOMAXVALUE
  [MINVALUE n | NOMINVALUE]   -- default: NOMINVALUE
  [CYCLE | NOCYCLE]           -- default: NOCYCLE
  [CACHE n | NOCACHE];        -- default: CACHE 20
```

* **`NOCYCLE`** (default) + limitga yetsa → `ORA-08004: sequence exceeds MAXVALUE and cannot be instantiated`.
* **`CYCLE`** — limitga yetgач `MINVALUE`/`MAXVALUE`dan qayta boshlaydi (noyoblik yo'qoladi).

### NEXTVAL va CURRVAL

* **`NEXTVAL`** — generatorni oshiradi va yangi qiymatni qaytaradi.
* **`CURRVAL`** — joriy sessiyadagi oxirgi `NEXTVAL` qiymatini qaytaradi.
* **EXAM CRITICAL:** yangi sessiyada `CURRVAL`ni chaqirishdan oldin **kamida bir marta `NEXTVAL`** ishlatilgan bo'lishi shart. Aks holda → `ORA-08002: sequence CURRVAL is not yet defined in this session`.

### Qayerda ishlatiladi / ishlatilmaydi

| RUXSAT (ALLOWED) | TAQIQLANGAN (NOT ALLOWED) |
| :--- | :--- |
| `SELECT` select-ro'yxatida | `WHERE` sharti |
| `INSERT ... VALUES` | `GROUP BY`, `HAVING`, `ORDER BY` |
| `UPDATE ... SET` | `DISTINCT` bilan |
| Ustun **`DEFAULT`** qiymatida *(12c+/19c)* | Set operator (`UNION` ...) tarkibidagi `SELECT` |
| | Subquery, view, CHECK constraint ifodasida |

> **19c tuzatish:** Oracle **12c+** dan boshlab `seq.NEXTVAL` va `CURRVAL` ni **ustun `DEFAULT` qiymati** sifatida ishlatish mumkin:
> ```sql
> CREATE TABLE orders (id NUMBER DEFAULT ord_seq.NEXTVAL PRIMARY KEY, ...);
> ```
> (Eski 11g va undan oldin bu mumkin emas edi.)

### Sequence Gaps (raqamlar uzilishi) — normal holat

Uzilish paydo bo'ladi, agar:
1. `NEXTVAL` olingan DML **`ROLLBACK`** qilinsa (sequence orqaga qaytmaydi).
2. DML xato bersa (masalan CHECK constraint buzilsa) — `NEXTVAL` baribir oshgan bo'ladi.
3. Tizim o'chib, `CACHE`dagi raqamlar yo'qolsa.

### ALTER / DROP

* **EXAM CRITICAL:** `ALTER SEQUENCE` bilan **`START WITH` ni o'zgartirib BO'LMAYDI** → `ORA-02283: cannot alter starting sequence number`. O'zgartirish uchun `DROP` + qayta `CREATE`.
* `INCREMENT BY`, `MAXVALUE`, `CACHE` va boshqalarni `ALTER` qilish mumkin.

> **`USER_SEQUENCES.LAST_NUMBER`** — joriy qiymat emas, **diskda saqlangan** keyingi raqam. `CACHE` tufayli u xotiradagi haqiqiy `NEXTVAL`dan **oldinda** turadi (cache'dagi butun blok oldindan ajratiladi):
> ```sql
> SELECT sequence_name, last_number, cache_size FROM user_sequences;
> ```

### IDENTITY ustunlari (12c+/19c) — sequence'ga muqobil

> Oracle 12c+ da avtomatik raqamlash uchun ustun darajasida **IDENTITY** ishlatiladi (ichida yashirin sequence yaratadi):
> ```sql
> CREATE TABLE t (
>   id   NUMBER GENERATED ALWAYS AS IDENTITY,          -- har doim avtomatik
>   -- yoki: GENERATED BY DEFAULT AS IDENTITY          -- qo'lda ham kiritsa bo'ladi
>   -- yoki: GENERATED BY DEFAULT ON NULL AS IDENTITY  -- NULL berilsagina avtomatik
>   name VARCHAR2(30)
> );
> ```
> * **`GENERATED ALWAYS`** — ustunga qo'lda qiymat kiritib bo'lmaydi → `ORA-32795: cannot insert into a generated always identity column`.
> * **`BY DEFAULT`** — qo'lda qiymat kiritsa bo'ladi; berilmasa avtomatik.
> * Jadvalda faqat **bitta** IDENTITY ustun bo'lishi mumkin; u bilvosita `NOT NULL`.

---

## 4-Qism: Index'lar (10.04)

**Index** — qidiruvni (`WHERE`, `ORDER BY`, join) tezlashtiruvchi presort qilingan alohida ob'ekt; qator manzili `ROWID`ni saqlaydi.

| Tur | Izoh |
| :--- | :--- |
| **B-Tree** (default) | ko'pchilik holat; yuqori selectivity uchun |
| **Bitmap** | kam noyob (low-cardinality) ustunlar uchun |
| **Unique** | qiymat noyobligini ta'minlaydi |
| **Composite** | bir nechta ustun; birinchi (leading) ustun eng muhim |
| **Function-based** | ifoda bo'yicha: `CREATE INDEX i ON t(UPPER(name))` |

### Composite va DESC index (ustun tartibi muhim)

```sql
CREATE INDEX ix_emp ON employees (department_id, salary DESC);
```

* **Composite index'da leading (birinchi) ustun hal qiluvchi:** optimizer index'dan faqat so'rovda **leading ustun** (`department_id`) qatnashganda samarali foydalanadi. Faqat `salary` bo'yicha qidiruv bu index'ni to'liq ishlata olmaydi.
* **`DESC`** — ustunni kamayuvchi tartibda saqlaydi; `ORDER BY ... DESC` so'rovlarini tezlashtiradi. (DESC bo'lmasa, default **ASC**.)
* **Function-based tuzoq:** `WHERE UPPER(name)='TOM'` so'rovi oddiy `name` index'ini **ishlatmaydi** — aynan `UPPER(name)` ustidagi function-based index kerak.

### Avtomatik (implicit) index

* **`PRIMARY KEY`** yoki **`UNIQUE`** constraint yaratilsa, agar mos index bo'lmasa, Oracle avtomatik **unique index** yaratadi (masalan `SYS_C009931`).
* `DROP TABLE` qilinsa, jadvalning barcha indexlari **avtomatik o'chadi**.

### Invisible Index (EXAM CRITICAL)

```sql
CREATE INDEX ix ON t(col) INVISIBLE;
ALTER INDEX ix VISIBLE;
ALTER INDEX ix INVISIBLE;
```

* Invisible index **optimizer uchun ko'rinmaydi** (execution plan'ga kirmaydi).
* **LEKIN DML (`INSERT`/`UPDATE`/`DELETE`) vaqtida to'liq yangilanadi** — resurs sarflaydi.
* Maqsad: index'ni o'chirishdan oldin ta'sirini xavfsiz test qilish.
* `USER_INDEXES.VISIBILITY` = `VISIBLE` / `INVISIBLE`.

### Bir xil ustunlarda bir nechta index (12c+)

* Bir xil ustun(lar) to'plamida bir nechta index bo'lishi mumkin, agar turlari/atributlari farqlansa (masalan B-Tree va Bitmap, yoki unique va non-unique).
* **QAT'IY QOIDA:** ulardan **faqat BITTAsi `VISIBLE`** bo'la oladi; qolganlari `INVISIBLE` bo'lishi shart.

### DROP

```sql
DROP INDEX ix;
```
PK/UNIQUE constraint tomonidan ishlatilayotgan index'ni to'g'ridan-to'g'ri o'chirib bo'lmaydi — avval constraint'ni drop/disable qilish kerak.

---

## 5-Qism: Flashback Amaliyotlari (10.05)

Flashback — ma'lumotni backup tiklamasdan o'tmishdagi holatiga tez qaytarish.

### A) Flashback Query — faqat o'qish (SELECT)

Jadval ma'lumotini o'tmishdagi holatda **ko'rish** (jadvalni o'zgartirmaydi):

```sql
-- Vaqt yoki SCN bo'yicha:
SELECT * FROM employees AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '15' MINUTE);
SELECT * FROM employees AS OF SCN 5896167;

-- Versiyalar oralig'i (Flashback Version Query):
SELECT versions_starttime, versions_operation, salary
FROM   employees VERSIONS BETWEEN TIMESTAMP t1 AND t2
WHERE  employee_id = 101;
```
* Pseudocolumn'lar: `VERSIONS_STARTTIME`, `VERSIONS_ENDTIME`, `VERSIONS_XID`, `VERSIONS_OPERATION` (`I`/`U`/`D`).
* Undo tablespace'dagi ma'lumotga bog'liq (`UNDO_RETENTION`).

### B) Flashback Drop — `TO BEFORE DROP`

O'chirilgan (`DROP TABLE`) jadvalni **Recycle Bin**'dan qaytaradi:

```sql
FLASHBACK TABLE emp TO BEFORE DROP [RENAME TO new_name];
```
* Tiklanadi: ma'lumot, B-Tree indexlar, triggerlar, grantlar, oddiy constraint'lar.
* **EXAM CRITICAL:** **Foreign Key constraint'lar TIKLANMAYDI** — qo'lda qayta yaratish kerak.
* Index va constraint'lar `BIN$...` tizim nomi bilan tiklanadi (qo'lda qayta nomlanadi).
* **LIFO:** bir xil nomli jadval bir necha marta drop qilingan bo'lsa, oxirgisi qaytariladi.
* **`PURGE`** qilingan ob'ektni tiklab bo'lmaydi → `ORA-38305: object not in RECYCLE BIN`.
  * `PURGE TABLE t;` — bitta jadvalni butunlay o'chiradi.
  * `PURGE RECYCLEBIN;` — recycle bin'ni tozalaydi.
* **`DROP TABLE t PURGE;`** — recycle bin'ga qo'ymasdan to'g'ridan-to'g'ri o'chiradi.

### C) Flashback Table — `TO TIMESTAMP / SCN / RESTORE POINT`

Mavjud jadvalni o'tmishdagi holatiga **qaytaradi** (noto'g'ri DML'ni bekor qilish):

```sql
ALTER TABLE emp ENABLE ROW MOVEMENT;   -- MAJBURIY old shart
FLASHBACK TABLE emp TO TIMESTAMP (SYSTIMESTAMP - INTERVAL '10' MINUTE);
FLASHBACK TABLE emp TO SCN 5896167;
FLASHBACK TABLE emp TO RESTORE POINT rp_good;
```
* **EXAM CRITICAL:** oldindan **`ALTER TABLE ... ENABLE ROW MOVEMENT`** bajarilgan bo'lishi shart; aks holda → `ORA-08189: cannot flashback the table because row movement is not enabled`.
* (`TO BEFORE DROP` uchun row movement **talab qilinmaydi**.)
* Flashback Table **implicit COMMIT** bajaradi.
* Oraliqda jadvalda DDL (ustun drop, tur o'zgarishi) bo'lgan bo'lsa — bajarilmaydi.

### Taqqoslash

| | Flashback Drop (`TO BEFORE DROP`) | Flashback Table (`TO TIMESTAMP/SCN`) |
| :--- | :--- | :--- |
| Maqsad | drop qilingan jadvalni tiklash | mavjud jadvaldagi DML'ni bekor qilish |
| Jadval holati | bazada yo'q (Recycle Bin'da) | bazada mavjud |
| `ENABLE ROW MOVEMENT` | **kerak emas** | **MAJBURIY** |
| Foreign Key | **tiklanmaydi** | saqlanadi |
| `PURGE` qilingan bo'lsa | ishlamaydi | undo'ga bog'liq |

---

## 6-Qism: Synonym'lar (10.06)

**Synonym** — bir ob'ektga (table, view, sequence, synonym, PL/SQL) qo'yilgan **taxallus (alias)**. Uzun yoki boshqa sxemadagi nomni qisqartiradi; o'zida ma'lumot saqlamaydi.

### Sintaksis

```sql
CREATE [OR REPLACE] [PUBLIC] SYNONYM syn_name FOR [schema.]object;

-- Private synonym (faqat yaratgan sxemaga tegishli):
CREATE SYNONYM emp FOR hr.employees;

-- Public synonym (butun baza foydalanuvchilariga ko'rinadi):
CREATE PUBLIC SYNONYM dept FOR hr.departments;
```

* **Private synonym** — schema ob'ekti; Table/View/Sequence bilan **bitta namespace**ni baham ko'radi (1-Qim), shuning uchun o'sha sxemada bir xil nomli table bilan birga bo'la olmaydi.
* **Public synonym** — nonschema ob'ekti (baza darajasida); `PUBLIC` sxemasiga tegishli.
* **`OR REPLACE`** — mavjud synonym'ni qayta yozadi (`CREATE TABLE`da yo'q, lekin synonym/view/sequence'da bor).
* Synonym yaratish **asos ob'ekt mavjudligini talab qilmaydi** va huquqni tekshirmaydi; chaqirilganda tekshiriladi.

### Nom yechish tartibi (resolution) — EXAM CRITICAL

Nom chaqirilganda Oracle shu tartibda qidiradi:

1. Joriy sxemadagi **o'z ob'ekti** (table/view/...) yoki **private synonym**.
2. Topilmasa — **public synonym**.

> **⚠️ Tuzoq:** agar sxemada `ORDERS` nomli **o'z jadvali** bo'lsa va **public synonym** ham `ORDERS` bo'lsa → doim **o'z jadvali** ustunlik qiladi, public synonym "ko'milib" qoladi.
> ```sql
> CREATE PUBLIC SYNONYM orders FOR hr.orders;   -- hr.orders ga ishora
> -- Lekin scott sxemasida o'zining ORDERS jadvali bo'lsa:
> SELECT * FROM orders;   -- scott.orders dan oladi (public synonym emas!)
> ```

### DROP va INVALID holat

```sql
DROP SYNONYM emp;                 -- private
DROP PUBLIC SYNONYM dept;         -- public (PUBLIC so'zi SHART)
```

* Asos ob'ekt o'chirilsa, synonym **o'chmaydi** — lekin `INVALID` bo'ladi; chaqirilsa → `ORA-00980: synonym translation is no longer valid`.
* `DROP TABLE` synonym'ni avtomatik o'chirmaydi (view'ga o'xshab, bog'liq synonym qoladi).

---

## 7-Qism: Imtihon Tuzoqlari — Tezkor Takrorlash

* **Namespace:** bir sxemada Table va View bir xil nom **mumkin emas** (`ORA-00955`); Table, Index, Constraint esa bir xil nomli bo'la **oladi**.
* **`CREATE VIEW`** da `OR REPLACE` va `FORCE` bor; **`CREATE TABLE`** da bu kalit so'zlar **yo'q**.
* **`WITH CHECK OPTION`** — `WHERE`ga mos kelmagan DML → `ORA-01402`. **`WITH READ ONLY`** — har qanday DML → `ORA-42399`.
* **View + `SELECT *`** — invisible ustun olinmaydi; invisible ustunni **nomi bilan** yozsa, view'da visible bo'ladi.
* **View INSERT** — barcha `NOT NULL`/DEFAULT siz ustunlar bo'lishi shart, aks holda `ORA-01400`.
* **Sequence:** yangi sessiyada `CURRVAL`dan oldin `NEXTVAL` kerak → aks holda `ORA-08002`.
* **Sequence + ROLLBACK/xato DML** → raqam orqaga qaytmaydi (gap paydo bo'ladi).
* **`ALTER SEQUENCE START WITH`** → mumkin emas (`ORA-02283`); drop+create kerak.
* **Sequence `DEFAULT`da** → 12c+/19c da mumkin (eski versiyada emas).
* **IDENTITY `GENERATED ALWAYS`** → qo'lda insert → `ORA-32795`.
* **Invisible Index** → optimizer ko'rmaydi, lekin DML vaqtida yangilanadi.
* **Bir xil ustunlarda ko'p index** → faqat BITTAsi VISIBLE.
* **PK/UNIQUE** → avtomatik unique index yaratadi; `DROP TABLE` indexlarni ham o'chiradi.
* **Flashback Table `TO TIMESTAMP/SCN`** → `ENABLE ROW MOVEMENT` **MAJBURIY** (`ORA-08189`); `TO BEFORE DROP` uchun kerak emas.
* **Flashback Drop** → Foreign Key **tiklanmaydi**; `PURGE` qilingan → `ORA-38305`.
* **Flashback Query** (`AS OF`, `VERSIONS BETWEEN`) → faqat ko'rish, jadvalni o'zgartirmaydi.
* **View INVALID** → asos jadval o'chsa/o'zgarsa view o'chmaydi, `INVALID` bo'ladi (`ORA-04063`); `ALTER VIEW ... COMPILE`.
* **Composite index** → faqat **leading ustun** so'rovda bo'lsa samarali; `UPPER(col)` so'rovi oddiy index'ni ishlatmaydi (function-based kerak).
* **`USER_SEQUENCES.LAST_NUMBER`** → joriy qiymat emas, cache tufayli diskdagi keyingi (oldinda turuvchi) raqam.
* **Synonym resolution** → avval o'z ob'ekt/private synonym, keyin public synonym; o'z jadvali bir xil nomli public synonym'dan **ustun**.
* **`DROP PUBLIC SYNONYM`** → `PUBLIC` so'zi shart; asos ob'ekt o'chsa synonym `INVALID` (`ORA-00980`).
