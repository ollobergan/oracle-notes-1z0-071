# Oracle 1Z0-071 — 12-Bob Konspekti
## Obyektlarni Data Dictionary Ko'rinishlari Bilan Boshqarish (Managing Objects with Data Dictionary Views)

Konspekt faqat imtihonda tekshiriladigan bilimlarga qaratilgan: ko'rinishlar ro'yxati, prefikslar qoidasi, cheklov kodlari, `COMMENT` va istisno holatlar (edge cases). Qo'shimcha ma'lumotlar Oracle **19c** bo'yicha.

---

## 1-Qism: Data Dictionary — Asosiy Tushuncha

* **Data Dictionary** — "ma'lumot haqida ma'lumot" (metadata) saqlaydigan, Oracle o'zi boshqaradigan tizim jadvallari va ko'rinishlari majmuasi.
* **DDL avtomatik yangilaydi:** `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME`, `GRANT`, `REVOKE` bajarilishi bilan Oracle dictionary'ni real vaqtda yangilaydi.
* **DML metadata'ni o'zgartirmaydi:** `INSERT`/`UPDATE`/`DELETE` faqat foydalanuvchi ma'lumotini o'zgartiradi, jadval strukturasini emas — demak dictionary metadata'siga ta'sir qilmaydi.
* **Egasi `SYS`:** barcha baza jadvallari (base tables, masalan `TAB$`, `OBJ$`, `COL$`) va ularning ustiga qurilgan ko'rinishlar `SYS` ga tegishli. Baza jadvallari shifrlangan/qiyin formatda — ularga to'g'ridan-to'g'ri murojaat qilinmaydi.
* **Faqat o'qish (READ-ONLY):** foydalanuvchi dictionary'ni ko'rinishlar orqali **faqat `SELECT`** qiladi. `SYS` baza jadvallariga qo'lda `INSERT`/`UPDATE`/`DELETE` qilish man etiladi (baza butunligini buzadi).
* **Yagona yozuv yo'li:** foydalanuvchi dictionary'ga bilvosita ma'lumot yozishi mumkin bo'lgan **yagona buyruq — `COMMENT`** (6-Qism).

---

## 2-Qism: Prefikslar — `USER_` / `ALL_` / `DBA_`

Static dictionary ko'rinishlari uch oilaga bo'linadi. Qamrov kengayib boradi: **`USER_` ⊆ `ALL_` ⊆ `DBA_`**.

| Xususiyat | `USER_` | `ALL_` | `DBA_` |
| :--- | :--- | :--- | :--- |
| Nimani ko'rsatadi? | Faqat **o'zimga tegishli** (owned) obyektlar | O'zimniki **+ huquq berilgan** barcha obyektlar | Bazadagi **BARCHA** obyektlar |
| **`OWNER` ustuni** | **YO'Q** | **BOR** | **BOR** |
| Kerakli huquq | Har qanday foydalanuvchi | Har qanday foydalanuvchi | `DBA` roli yoki `SELECT ANY DICTIONARY` |
| Misol | `USER_TABLES` | `ALL_TABLES` | `DBA_TABLES` |

* **`USER_` da `OWNER` yo'q** — chunki egasi doim joriy foydalanuvchining o'zi. `OWNER` ustunini so'rasangiz → `ORA-00904: invalid identifier`. Bu eng ko'p takrorlanadigan tuzoqlardan biri.
* `USER_` ko'rinishlari joriy sessiyani **`USER` funksiyasi** (`SELECT USER FROM DUAL`) qiymati bo'yicha yashirin filtrlaydi.
* **`ALL_` o'z obyektlaringni ham ko'rsatadi:** `ALL_TABLES` da o'zingning jadvallaring ham chiqadi (ularning `OWNER` i — sening nomingdir).

---

## 3-Qism: Dinamik Unumdorlik Ko'rinishlari (`V$` / `GV$`)

* **Nima:** ma'lumotlar bazasining **xotiradagi (SGA)** real-vaqt holatini — sessiyalar, unumdorlik, instance, fayllar — ko'rsatadigan ko'rinishlar. Diskdagi jadvaldan emas, xotira strukturalaridan (`X$` fixed tables) o'qiydi; instance ishga tushganda to'ldiriladi.
* **Nomlanishi:** asl obyekt `V_$...` / `GV_$...` ko'rinishlari; foydalanuvchi ularga **`V$...` / `GV$...` ommaviy sinonimlari** orqali murojaat qiladi (`V$DATABASE`, `V$SESSION`, `V$DATAFILE`, ...). `GV$` — RAC (bir nechta instance) uchun global variant.
* **Read consistency kafolatlanmaydi (EXAM):** `V$` xotirada doimiy o'zgarib turgani uchun Oracle ular bo'yicha o'qish izchilligini **kafolatlamaydi**. Bitta so'rovda bir nechta `V$` ni `JOIN` qilsangiz, qiymatlar bir-biriga mos kelmasligi (nomuvofiq "surat") mumkin — texnik jihatdan taqiqlanmagan, ammo natijaga ishonib bo'lmaydi.

---

## 4-Qism: Imtihonda Uchraydigan Asosiy Ko'rinishlar

> Barcha ko'rinishlar `USER_` / `ALL_` / `DBA_` variantlarida mavjud (quyida qisqalik uchun `USER_` berilgan). Qavsdagi nom — ommaviy sinonim.

**Umumiy / katalog:**
* **`DICTIONARY` (`DICT`)** — barcha dictionary ko'rinishlarining "mundarijasi". Ustunlari: `TABLE_NAME`, `COMMENTS`. Kerakli ko'rinishni topish uchun shu yerdan qidiriladi.
* **`USER_CATALOG` (`CAT`)** — qisqa xulosa, faqat **2 ustun**: `TABLE_NAME`, `TABLE_TYPE`. `TABLE_TYPE` qiymatlari: `TABLE`, `VIEW`, `SYNONYM`, `SEQUENCE` (ba'zan `CLUSTER`).
* **`USER_OBJECTS` (`OBJ`)** — foydalanuvchining **barcha** obyektlari. Ustunlari: `OBJECT_NAME`, `OBJECT_TYPE`, `OBJECT_ID`, `CREATED`, `LAST_DDL_TIME`, `STATUS` (`VALID`/`INVALID`), `GENERATED` (`Y`/`N` — nom tizim tomonidan berilganmi).

**Jadval va ustunlar:**
* **`USER_TABLES` (`TABS`)** — jadval sozlamalari: `TABLE_NAME`, `TABLESPACE_NAME`, `NUM_ROWS`, `STATUS`, ... (`OWNER` yo'q).
* **`USER_TAB_COLUMNS` (`COLS`)** — ustunlar tafsiloti: `TABLE_NAME`, `COLUMN_NAME`, `DATA_TYPE`, `DATA_LENGTH`, `DATA_PRECISION`, `DATA_SCALE`, `NULLABLE` (`Y`/`N`), `COLUMN_ID`.

**Cheklovlar (constraints):**
* **`USER_CONSTRAINTS`** — `CONSTRAINT_NAME`, `CONSTRAINT_TYPE`, `TABLE_NAME`, `R_CONSTRAINT_NAME`, `SEARCH_CONDITION`, `STATUS`.
  * `R_CONSTRAINT_NAME` — Foreign Key ishora qilayotgan ota (parent) PK/Unique cheklov nomi.
  * `SEARCH_CONDITION` — Check sharti matni (`LONG` turidagi ustun).
* **`USER_CONS_COLUMNS`** — cheklov qaysi ustun(lar)ga tegishli: `CONSTRAINT_NAME`, `TABLE_NAME`, `COLUMN_NAME`, `POSITION`.

### `CONSTRAINT_TYPE` kodlari (EXAM CRITICAL)

| Kod | Ma'nosi |
| :--- | :--- |
| **`P`** | Primary Key |
| **`R`** | **Foreign Key** / referential integrity (**`F` yoki `FK` EMAS!**) |
| **`U`** | Unique |
| **`C`** | Check **yoki** NOT NULL |
| **`V`** | View ustidagi `WITH CHECK OPTION` |
| **`O`** | View ustidagi `WITH READ ONLY` |

* **NOT NULL — bu `C`:** `NOT NULL` ichki tarzda Check cheklov sifatida saqlanadi; `SEARCH_CONDITION` da `"COL" IS NOT NULL` ko'rinadi.
* **Tizim bergan nomlar:** cheklovni (yoki indeksni) nomlamagan bo'lsangiz, Oracle `SYS_Cnnnnnn` ko'rinishidagi nom beradi (`GENERATED = 'Y'`).

**View, indeks, ketma-ketlik, sinonim:**
* **`USER_VIEWS`** — view ta'rifi. `TEXT` ustuni — view'ning `SELECT` matni (**`LONG` turi**); `TEXT_LENGTH` — uzunligi.
* **`USER_INDEXES` (`IND`)** — `INDEX_NAME`, `INDEX_TYPE`, `TABLE_NAME`, `UNIQUENESS` (`UNIQUE`/`NONUNIQUE`), `VISIBILITY` (`VISIBLE`/`INVISIBLE`), `STATUS`.
* **`USER_IND_COLUMNS`** — indeks ustunlari va tartibi: `INDEX_NAME`, `TABLE_NAME`, `COLUMN_NAME`, `COLUMN_POSITION`.
* **`USER_SEQUENCES` (`SEQ`)** — `SEQUENCE_NAME`, `MIN_VALUE`, `MAX_VALUE`, `INCREMENT_BY`, `CYCLE_FLAG`, `CACHE_SIZE`, `LAST_NUMBER`.
* **`USER_SYNONYMS`** — faqat **shaxsiy (private) sinonimlar**. Ommaviy (public) sinonimlar bu yerda **KO'RINMAYDI** — ular `ALL_SYNONYMS` / `DBA_SYNONYMS` da (egasi `PUBLIC`).

**Izohlar (comments):**
* **`USER_TAB_COMMENTS`** — jadval/view izohlari: `TABLE_NAME`, `TABLE_TYPE`, `COMMENTS`.
* **`USER_COL_COMMENTS`** — ustun izohlari: `TABLE_NAME`, `COLUMN_NAME`, `COMMENTS`.

**Huquqlar va rollar:**
* **`USER_SYS_PRIVS`** — foydalanuvchiga to'g'ridan-to'g'ri berilgan tizim huquqlari (`PRIVILEGE`, `ADMIN_OPTION`).
* **`USER_TAB_PRIVS`** — obyekt (jadval) huquqlari (`GRANTEE`, `TABLE_NAME`, `PRIVILEGE`, `GRANTABLE`).
* **`USER_ROLE_PRIVS`** — berilgan rollar (`GRANTED_ROLE`, `ADMIN_OPTION`, `DEFAULT_ROLE`).
* **`SESSION_PRIVS`** — joriy sessiyada aktiv barcha tizim huquqlari (to'g'ridan + rollar orqali). Prefikssiz nom.

---

## 5-Qism: `COMMENT` — Izohlar Bilan Ishlash

Foydalanuvchining dictionary'ga yozish imkoni beradigan yagona buyruq.

* **Sintaksis:**
  * Jadval: `COMMENT ON TABLE table_name IS 'matn';`
  * Ustun: `COMMENT ON COLUMN table_name.column_name IS 'matn';`
* **Faqat `TABLE` va `COLUMN` ga** qo'llanadi (shuningdek materialized view kabi ba'zi obyektlarga). **`INDEX` yoki `SEQUENCE` ga `COMMENT` — xatolik** (sintaksis xatosi, odatda `ORA-00903`).
* **Izohni o'chirish:** `DROP COMMENT` buyrug'i **YO'Q**. Izoh bo'sh satr bilan yangilanadi (natijada `COMMENTS` qiymati `NULL` bo'ladi):
  ```sql
  COMMENT ON COLUMN employees.salary IS '';
  ```
* **O'qish:** `USER_TAB_COMMENTS` (jadval) va `USER_COL_COMMENTS` (ustun) orqali.

```sql
-- Qo'shish
COMMENT ON TABLE  employees        IS 'Barcha xodimlar ro''yxati';
COMMENT ON COLUMN employees.salary IS 'Oylik maosh (USD)';

-- O'qish
SELECT column_name, comments
FROM   user_col_comments
WHERE  table_name = 'EMPLOYEES';
```

---

## 6-Qism: Amaliy SQL Misollar

```sql
-- Kerakli ko'rinishni DICT dan qidirish
SELECT table_name, comments
FROM   dictionary
WHERE  LOWER(table_name) LIKE '%index%';

-- INVALID bo'lib qolgan view'larni topish (ota jadval o'zgargach)
SELECT object_name, status, last_ddl_time
FROM   user_objects
WHERE  object_type = 'VIEW'
AND    status = 'INVALID';

-- Ma'lum ustun qaysi jadvallarda bor?
SELECT table_name, data_type, data_length
FROM   user_tab_columns
WHERE  column_name = 'EMPLOYEE_ID';

-- Jadval cheklovlari + FK qaysi PK ga bog'langani
SELECT constraint_name, constraint_type, r_constraint_name, status
FROM   user_constraints
WHERE  table_name = 'INVOICES';
```

---

## 7-Qism: Imtihon Tuzoqlari — Tezkor Takrorlash

* **`USER_` da `OWNER` ustuni YO'Q** → `SELECT owner ... FROM user_tables` = `ORA-00904`. `OWNER` faqat `ALL_`/`DBA_` da.
* **Foreign Key kodi — `R`** (referential), `F`/`FK` emas.
* **NOT NULL — `CONSTRAINT_TYPE = 'C'`** (Check sifatida saqlanadi).
* **`USER_SYNONYMS` public sinonimlarni ko'rsatmaydi** → `CREATE PUBLIC SYNONYM` dan keyin bu yerda 0 qator; ular `ALL_SYNONYMS`/`DBA_SYNONYMS` da.
* **Nomlar katta harfda:** dictionary'da obyekt nomlari standart holda UPPERCASE saqlanadi → `WHERE table_name = 'employees'` = 0 qator (`'EMPLOYEES'` yozish kerak; `""` bilan yaratilmagan bo'lsa).
* **DDL dictionary'ni yangilaydi; DML yangilamaydi.** `SYS` baza jadvallariga qo'lda DML man etiladi.
* **`COMMENT` faqat `TABLE`/`COLUMN` uchun;** indeks/ketma-ketlikka emas. O'chirish = `IS ''` (`DROP COMMENT` yo'q).
* **`V$` — read consistency kafolatlanmaydi** (xotiradagi ma'lumot); bir nechta `V$` `JOIN` natijasiga ishonib bo'lmaydi.
* **`USER_CATALOG` / `CAT` — 2 ustun** (`TABLE_NAME`, `TABLE_TYPE`).
* **`USER_VIEWS.TEXT` va `USER_CONSTRAINTS.SEARCH_CONDITION` — `LONG` turidagi** ustunlar (filtrlash/`WHERE` da cheklangan).
* **Nomlanmagan cheklov/indeks** → tizim `SYS_Cnnnnnn` nom beradi (`GENERATED = 'Y'`).
