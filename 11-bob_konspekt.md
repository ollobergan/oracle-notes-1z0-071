# Oracle 1Z0-071 — 11-Bob Konspekti
## Set Operatorlaridan Foydalanish (Using the Set Operators)

Konspekt faqat imtihonda tekshiriladigan bilimlarga qaratilgan: qoidalar, istisno holatlar (edge cases) va sintaktik tuzoqlar. Qo'shimcha ma'lumotlar Oracle **19c** bo'yicha berilgan.

---

## 1-Qism: Umumiy Tushuncha

**Set operator** — ikki yoki undan ortiq mustaqil `SELECT` so'rovining natijalarini **vertikal** (qatorma-qator, ustma-ust) birlashtiradi.

* `JOIN` jadvallarni **gorizontal** (ustun qo'shib) birlashtiradi; set operator **vertikal** (qator qo'shib).
* Birlashtirilayotgan so'rovlar o'rtasida PK/FK bog'liqlik **shart emas** — manbalar butunlay bog'liqsiz bo'lishi mumkin.
* Har bir `SELECT` o'zining `WHERE`, `GROUP BY`, `HAVING`, `JOIN`, funksiya va subquery'siga ega bo'lishi mumkin.

Oracle **19c** da faqat **4 ta** set operatori bor:

| Operator | Vazifasi | Takrorlar (Duplicates) | Yashirin sort | Kommutativ (A op B = B op A) |
| :--- | :--- | :--- | :--- | :--- |
| **`UNION`** | Ikkala so'rov qatorlari birlashadi | **Olib tashlanadi** | Ha | Ha |
| **`UNION ALL`** | Ikkala so'rov qatorlari birlashadi | **Saqlanadi** | **Yo'q** | Ha |
| **`INTERSECT`** | Faqat **ikkalasida ham** bor qatorlar | Olib tashlanadi | Ha | Ha |
| **`MINUS`** | 1-so'rovdan 2-so'rovda borlarini **ayiradi** | Olib tashlanadi | Ha | **Yo'q** |

* **`UNION ALL`** — eng **tez**, chunki duplicate tekshirish/sort yo'q. Takrorlar bo'lmasligini bilsangiz, `UNION` o'rniga ishlating.
* **`MINUS` kommutativ EMAS:** `A MINUS B ≠ B MINUS A`. Qolgan uchtasida so'rovlar o'rni almashsa natija (to'plam sifatida) o'zgarmaydi.

---

## 2-Qism: Qat'iy Qoidalar (EXAM CRITICAL)

1. **Ustunlar soni teng** bo'lishi shart (har bir `SELECT` dagi select-ro'yxat uzunligi bir xil).
   * Aks holda → `ORA-01789: query block has incorrect number of result columns`.

2. **Mos pozitsiyadagi ustunlar bir xil data type guruhida** bo'lishi (yoki Oracle implicit konversiya qila olishi) kerak.
   * Guruhlar: **character** (`CHAR`/`VARCHAR2`), **numeric** (`NUMBER`), **datetime** (`DATE`/`TIMESTAMP`).
   * `NUMBER` va `VARCHAR2` mos kelmaydi → `ORA-01790: expression must have same datatype as corresponding expression`.
   * Yechim: `TO_CHAR`, `TO_NUMBER`, `TO_DATE` bilan moslashtirish, yoki yetishmagan ustun o'rniga `NULL` / konstanta qo'yish.

3. **Ustun nomlari (sarlavhalari) faqat BIRINCHI `SELECT` dan olinadi.** Keyingi so'rovlardagi nom/alias e'tiborga olinmaydi.

4. **Natija ustunining uzunligi/aniqligi** — eng kattasiga tenglashadi (masalan `VARCHAR2(5)` va `VARCHAR2(10)` → natija `VARCHAR2(10)`).

5. **`NULL` lar teng deb hisoblanadi.** Takrorlarni aniqlashda (`UNION`, `INTERSECT`, `MINUS`) Oracle `NULL = NULL` deb qaraydi — ikki `NULL` qator bir xil hisoblanadi.

6. **Taqiqlangan data type'lar:** `BLOB`, `CLOB`, `NCLOB`, `BFILE`, `LONG`, `LONG RAW` va user-defined object type'lar set operatorlarida ishlatilmaydi.

7. **Sequence taqiqi:** set operatorli so'rovning `SELECT` ro'yxatida `NEXTVAL` / `CURRVAL` ishlatib bo'lmaydi.

### Misol — ustun nomi va NULL bilan to'ldirish

```sql
SELECT product_id AS id, product_name, price      -- 3 ustun, nomlar shu yerdan
FROM store_inventory
UNION
SELECT item_id, item_name, NULL                   -- NULL bilan 3-ustun o'rni to'ldirildi
FROM furnishings;
-- Natija sarlavhalari: ID, PRODUCT_NAME, PRICE
```

---

## 3-Qism: `ORDER BY` Qoidalari (EXAM CRITICAL)

* Butun zanjirda **faqat BITTA `ORDER BY`** bo'lishi mumkin va u **eng oxirida** turadi.
  * Oraliq `SELECT` ga `ORDER BY` qo'yilsa → `ORA-00933: SQL command not properly ended`.
* `ORDER BY` da ustunga 3 xil usulda murojaat qilinadi:
  * **Pozitsiya raqami** bo'yicha — `ORDER BY 1, 2` (eng xavfsiz usul).
  * **Birinchi `SELECT` dagi ustun nomi/alias'i** bo'yicha.
  * 2- yoki keyingi `SELECT` dagi nom ishlatilsa → `ORA-00904: invalid identifier`.

```sql
SELECT 'Mijoz' AS turi, last_name AS ism FROM customers
UNION ALL
SELECT 'Yetkazib beruvchi', vendor_name FROM vendors
ORDER BY ism;        -- birinchi SELECT alias'i (yoki: ORDER BY 2)
-- 'vendor_name' deb yozilsa → ORA-00904
```

---

## 4-Qism: Prioritet va Qavslar

* Oracle **19c** da barcha set operatorlari **TENG prioritetga** ega → **chapdan o'ngga / yuqoridan pastga** bajariladi.
* Bajarilish tartibini o'zgartirish uchun **qavs `()`** ishlatiladi — qavs ichidagi avval bajariladi.

```sql
(
  SELECT product FROM store_inventory
  UNION ALL
  SELECT item_name FROM furnishings
)
INTERSECT
SELECT item_name FROM furnishings WHERE item_name = 'Towel';
```

> **19c tuzatish / kelajak ogohlantirishi:** Oracle 19c hujjati `INTERSECT` ni boshqa operatorlar bilan birga ishlatganda **qavs qo'yishni tavsiya qiladi**, chunki keyingi versiya (**21c**) dan boshlab `INTERSECT` ga **yuqori prioritet** beriladi. 19c da esa u hali teng prioritetda.

---

## 5-Qism: `MINUS` vs `NOT IN` — `NULL` Tuzog'i (EXAM CRITICAL)

Subquery natijasida **bitta bo'lsa ham `NULL`** bo'lsa:

* **`NOT IN`** butun so'rov uchun **`no rows selected`** qaytaradi (`NULL` bilan taqqoslash `UNKNOWN` beradi).
* **`MINUS`** esa `NULL` ni oddiy qiymat kabi to'g'ri taqqoslaydi va ishlayveradi.

```sql
-- 2-jadvalda NULL bo'lsa — HECH NARSA qaytmaydi:
SELECT house FROM parent WHERE house NOT IN (SELECT house FROM kid);

-- MINUS esa to'g'ri ishlaydi:
SELECT house FROM parent
MINUS
SELECT house FROM kid;
```

---

## 6-Qism: Imtihon Tuzoqlari — Tezkor Takrorlash

* **4 ta operator:** `UNION`, `UNION ALL`, `INTERSECT`, `MINUS`. `SET` (`UPDATE ... SET`) — bu **set operatori EMAS**.
* **19c da `UNION DISTINCT`, `INTERSECT ALL`, `MINUS ALL` YO'Q** — faqat `UNION ALL` da `ALL` variant bor. (`EXCEPT` ham Oracle kalit so'zi emas — uning o'rniga `MINUS`.) *(`INTERSECT ALL` / `MINUS ALL` / `EXCEPT` 21c da qo'shilgan.)*
* **`UNION ALL`** — eng tez (sort/dedup yo'q), takrorlarni saqlaydi.
* **`MINUS`** — kommutativ emas: so'rovlar o'rni muhim.
* **Ustunlar soni teng emas** → `ORA-01789`. **Data type guruhlari mos emas** → `ORA-01790`.
* **Natija sarlavhalari** — faqat **birinchi `SELECT`** dan.
* **`ORDER BY`** — butun zanjir oxirida **faqat 1 ta**; oraliqda bo'lsa → `ORA-00933`. Nom bo'yicha saralashda faqat **birinchi `SELECT`** ustuni; aks holda `ORA-00904`. Pozitsiya (`ORDER BY 1`) eng ishonchli.
* **Tartib kafolati:** `ORDER BY` bo'lmasa, natija tartibi **kafolatlanmaydi** (UNION "saralangandek" ko'rinishi mumkin, lekin bunga tayanmang).
* **`NULL`** takror aniqlashda teng (`NULL = NULL`).
* **Taqiqlangan:** `BLOB`/`CLOB`/`LONG` ustunlar va `NEXTVAL`/`CURRVAL`.
* **Prioritet** 19c da teng (chapdan o'ngga); tartibni o'zgartirish → **qavs `()`**.
* **`NOT IN` + `NULL`** → `no rows`; **`MINUS` + `NULL`** → to'g'ri ishlaydi.
