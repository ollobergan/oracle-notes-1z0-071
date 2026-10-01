# Oracle 1Z0-071 — 6-Bob Konspekti
## Konversiya Funksiyalari va Shartli Ifodalar (Using Conversion Functions and Conditional Expressions)

Konspekt faqat imtihonda tekshiriladigan bilimlarga qaratilgan: qoidalar, istisno holatlar (edge cases) va sintaktik tuzoqlar.

---

## 1-Qism: Konversiya — Umumiy Tushuncha

Konversiya qiymatning **ma'lumot turini** (data type) o'zgartiradi, qiymatning o'zini emas. Uchta asosiy tur: **raqamli** (`NUMBER`), **matnli** (`VARCHAR2`/`CHAR`), **sana/vaqt** (`DATE`/`TIMESTAMP`).

### Implicit (yashirin) va Explicit (ochiq) konversiya
*   **Implicit** — Oracle mos kelmagan, lekin moslashtirsa bo'ladigan turlarni **avtomatik** o'zgartiradi.
*   **Explicit** — dasturchi `TO_NUMBER`, `TO_CHAR`, `TO_DATE`, `CAST` kabi funksiyalar bilan **ochiq** o'zgartiradi.

**Explicit nega afzal (imtihon savoli):**
*   Implicit konversiya qo'shimcha CPU talab qiladi (ish unumdorligi pasayadi).
*   Implicit algoritm kelgusi relizlarda o'zgarishi mumkin — kod barqaror emas.
*   **Indekslarni bekor qiladi:** indekslangan ustunda implicit konversiya bo'lsa, optimizator indeksdan foydalanmasligi mumkin.
*   Explicit kod maqsadni aniq ko'rsatadi (self-documenting).

### Implicit konversiyaning yo'nalishlari (bilish foydali)
*   `VARCHAR2`/`CHAR` ↔ `NUMBER` (agar satr haqiqiy raqam bo'lsa).
*   `VARCHAR2`/`CHAR` ↔ `DATE` (agar satr NLS format maskasiga mos bo'lsa).
*   `NUMBER` → `VARCHAR2`.

---

## 2-Qism: Explicit Konversiya Funksiyalari

### 2.1. TO_NUMBER — satrni songa
```sql
TO_NUMBER(expression [, format_model [, nls_parms]])
```
*   `expression` majburiy; `format_model` kiruvchi satrdagi valyuta, vergul, nuqta joylashuvini tushuntiradi.
*   `nls_parms` — decimal belgisi, guruh ajratgichi, valyuta simvolini lokal belgilaydi; yagona satr literal ichida yoziladi.
*   `TO_NUMBER('$17,000.23', '$99,999.99')` → `17000.23`

### 2.2. TO_CHAR — songa YOKI sanaga ishlaydi (overloaded)
`TO_CHAR` **yuklangan (overloaded)** funksiya: ham sonni, ham sanani matnga o'tkazadi.

**A) Sonni matnga:** `TO_CHAR(number [, format_model [, nls_parms]])`

**Son formatlash elementlari:**
| Element | Tavsif | Misol |
| :--- | :--- | :--- |
| `9` | Istalgan raqam (yetishmasa — bo'sh joy) | `9999` |
| `0` | Yetishmayotgan o'ringa **nol** qo'yadi | `0999` → `0150` |
| `$` | AQSh dollari belgisi | `$999` |
| `L` | Lokal valyuta belgisi (NLS) | `L999` |
| `C` | ISO xalqaro valyuta kodi | `C999` → `USD150` |
| `,` | Guruh (mingliklar) ajratgichi | `9,999` |
| `.` | O'nlik kasr ajratgichi (faqat bitta) | `99.99` |
| `MI` | Manfiy belgini **o'ngga** chiqaradi | `999MI` → `150-` |
| `PR` | Manfiy sonni `<>` qavsga oladi | `999PR` → `<150>` |
| `RN` / `rn` | Rim raqami (katta/kichik harf) | `RN` → `CL` |
| `EEEE` | Ilmiy (eksponensial) ko'rinish | `9.9EEEE` |
*   `TO_CHAR(1234.5, '$9,999.99')` → `$1,234.50`

**B) Sanani matnga:** `TO_CHAR(date [, format_model [, nls_parms]])`

**Sana formatlash elementlari:**
| Element | Tavsif | Natija |
| :--- | :--- | :--- |
| `YYYY` | 4 xonali yil | `2026` |
| `RRRR` / `RR` | Asr almashuvini hisobga oluvchi yil (pastda) | `2026` / `26` |
| `YY` | 2 xonali oddiy yil | `26` |
| `MM` | Oy raqami (01–12) | `09` |
| `MONTH` / `MON` | Oyning to'liq nomi / 3 harfli qisqartmasi | `SEPTEMBER` / `SEP` |
| `DAY` / `DY` | Hafta kunining to'liq nomi / 3 harfli qisqartmasi | `FRIDAY` / `FRI` |
| `DD` | Oydagi kun (1–31) | `04` |
| `DDD` | Yildagi kun (1–366) | `247` |
| `HH24` / `HH` (yoki `HH12`) | 24 soatlik / 12 soatlik vaqt | `19` / `07` |
| `MI` | **Daqiqa (0–59)** | `01` |
| `SS` | Soniya (0–59) | `38` |
| `AM` / `PM` | Ertalab / kechqurun belgisi | `PM` |
| `FM` | **Fill Mode** — trailing bo'shliq va leading nollarni o'chiradi | `FMDAY` → `FRIDAY` |
| `FX` | **Format Exact** — satr formatga 100% mos kelishini talab qiladi | — |
| `"matn"` | O'zgarmas matnni natijaga qo'shadi | `"of"` |

*   **Registr natijaga ta'sir qiladi:** `'MONTH'`→`SEPTEMBER`, `'Month'`→`September`, `'month'`→`september`.
*   `TO_CHAR(SYSDATE, 'FMDAY, "the" DDth "of" MONTH, YYYY')` → `FRIDAY, the 4th of SEPTEMBER, 2026`

> **⚠️ KLASSIK TUZOQ — `MM` vs `MI`:** Vaqtda daqiqani olish uchun **`MI`** ishlatiladi. `TO_CHAR(SYSDATE, 'HH24:MM:SS')` sintaktik xatosiz ishlaydi, lekin daqiqa o'rniga **oy raqamini** chiqaradi. To'g'risi: `'HH24:MI:SS'`.

### 2.3. TO_DATE — satrni sanaga
```sql
TO_DATE(char_string [, format_model [, nls_parms]])
```
*   Satr `format_model` ga har bir belgisigacha mos bo'lishi kerak; aks holda `ORA-01840` kabi xatolik.
*   `TO_DATE('2016-01-31', 'RRRR-MM-DD')` → `31-JAN-16`

### 2.4. CAST — universal tur konvertori
```sql
CAST(expression AS data_type)
```
*   Istalgan ifodani ko'rsatilgan turga o'tkazadi (`NUMBER`, `VARCHAR2(n)`, `DATE`, `TIMESTAMP`, interval turlari h.k.).
*   `TO_*` funksiyalaridan farqi — **format maskasi yo'q** (NLS standart formatiga tayanadi), lekin barcha turlar bilan ishlaydi.
*   `CAST('25-DEC-2026' AS DATE)`, `CAST(123.45 AS VARCHAR2(10))`, `CAST('2026' AS NUMBER)`.

---

## 3-Qism: RR va YY — Asr Logikasi (imtihon tuzog'i)

2 xonali yil berilganda `YY` joriy asrni oladi, `RR` esa asrni **aqlli** aniqlaydi:

| Joriy yilning oxirgi 2 raqami | Berilgan 2 xonali yil 00–49 | Berilgan 2 xonali yil 50–99 |
| :--- | :--- | :--- |
| **00–49** | **joriy** asr | **oldingi** asr |
| **50–99** | **keyingi** asr | **joriy** asr |

*   Misol (joriy yil 2026 → oxirgi 2 raqam 26, ya'ni 00–49 guruhi): `RR` bilan `'95'` → `1995`, `'15'` → `2015`.
*   `YY` bilan har doim joriy asr: `'95'` → `2095`.

---

## 4-Qism: Interval va Timestamp Konversiyalari

*   **`TO_TIMESTAMP(char, format)`** — satrni kasr soniyali `TIMESTAMP` ga o'tkazadi.
*   **`TO_YMINTERVAL('Y-M')`** — `INTERVAL YEAR TO MONTH` ga o'tkazadi.
    *   Format qat'iy `'Y-M'` bo'lishi shart: `TO_YMINTERVAL('01-02')` = 1 yil 2 oy.
    *   **Tuzoq:** defis o'rniga ikki nuqta — xato. `TO_YMINTERVAL('01:02')` → xatolik.
    *   **Tuzoq:** `TO_INTERVALYM` nomli funksiya **mavjud emas**.
*   **`TO_DSINTERVAL('D HH:MI:SS')`** — `INTERVAL DAY TO SECOND` ga o'tkazadi.

---

## 5-Qism: NULL bilan Ishlovchi Funksiyalar

### 5.1. NVL — NULL o'rnini bosuvchi
```sql
NVL(expr1, expr2)
```
*   `expr1` `NULL` emas → `expr1`; `expr1` `NULL` → `expr2`.
*   **Aynan 2** parametr. `expr1` va `expr2` turlari mos (yoki implicit o'tadigan) bo'lishi shart.
*   `SELECT salary + NVL(bonus, 0) FROM employees;`

### 5.2. NVL2 — uch argumentli variant
```sql
NVL2(expr1, expr2, expr3)
```
*   `expr1` `NULL` **emas** → `expr2`; `expr1` `NULL` → `expr3`.
*   `NVL2(bonus, 'Has bonus', 'No bonus')`
*   **E'tibor:** NVL2 da natija `expr1` ning o'zi EMAS — NULL bo'lmaganda ham `expr2` qaytadi.

### 5.3. COALESCE — birinchi NULL bo'lmagan qiymat
```sql
COALESCE(expr1, expr2, ..., exprN)
```
*   Ro'yxatdagi **birinchi `NULL` bo'lmagan** qiymatni qaytaradi; barchasi `NULL` bo'lsa → `NULL`.
*   **Kamida 2** argument; barcha argumentlar bir xil (yoki mos) turda bo'lishi kerak.
*   `COALESCE(comm, bonus, salary, 0)`
*   **NVL bilan farqi:** NVL faqat 2 argument; COALESCE ko'p argument qabul qiladi va argumentlarni **faqat kerak bo'lgunicha** baholaydi (short-circuit).

### 5.4. NULLIF — teng bo'lsa NULL
```sql
NULLIF(expr1, expr2)
```
*   `expr1` = `expr2` → **`NULL`**; teng emas → **`expr1`**.
*   Bu `CASE WHEN expr1 = expr2 THEN NULL ELSE expr1 END` ga teng.
*   **Tuzoq:** birinchi argument literal `NULL` bo'la olmaydi (xatolik beradi).
*   `NULLIF(100, 100)` → `NULL`; `NULLIF(100, 200)` → `100`.

---

## 6-Qism: Shartli Ifodalar

### 6.1. CASE — ikki shakli
**Searched CASE** (har xil shartlar, murakkab operatorlar):
```sql
CASE WHEN days <= 3 THEN 'Quick'
     WHEN days <= 10 THEN 'Medium'
     ELSE 'Long'
END
```
**Simple CASE** (bitta ifodani tenglik bo'yicha solishtiradi):
```sql
CASE status WHEN 1 THEN 'Active'
            WHEN 2 THEN 'Closed'
            ELSE 'Unknown'
END
```
*   **Har doim `END` bilan tugaydi** (yozilmasa — syntax error).
*   `THEN`/`ELSE` qaytargan barcha qiymatlar **bir xil turda** bo'lishi shart.
*   `ELSE` yo'q va hech bir shart mos kelmasa → **`NULL`**.
*   Simple CASE faqat **tenglik (`=`)** bilan ishlaydi; `>`, `<`, `IN`, `BETWEEN` kerak bo'lsa Searched CASE.

### 6.2. DECODE — Oracle'ga xos (proprietary)
```sql
DECODE(expression, search1, result1 [, search2, result2, ...] [, default])
```
*   `expression` ni `search` lar bilan navbatma-navbat solishtiradi; mos kelsa mos `result` ni qaytaradi.
*   Hech biri mos kelmasa → `default` (berilmasa → `NULL`).
*   `DECODE(capacity, 2052, 'SMALL', 2974, 'LARGE', 'UNKNOWN')`

**DECODE ning farqlari (imtihon uchun muhim):**
*   Oxirida **`END` yo'q** — oddiy funksiya kabi `)` bilan tugaydi.
*   Faqat **tenglik (`=`)** tekshiriladi (`>`, `<`, `AND` mumkin emas).
*   **NULL ni maxsus ishlaydi:** DECODE `NULL` ni `NULL` ga **teng** deb hisoblaydi (oddiy `=` da `NULL = NULL` → noma'lum). Shuning uchun `DECODE(col, NULL, 'bosh', 'tola')` ishlaydi.
*   ANSI emas — faqat Oracle'da.

---

## 7-Qism: Taqqoslash Jadvallari

### CASE va DECODE
| Xususiyat | CASE | DECODE |
| :--- | :--- | :--- |
| Standart | ANSI SQL (universal) | Oracle'ga xos |
| Tugash | `END` majburiy | `)` bilan (END yo'q) |
| Operatorlar | `=`, `>`, `<`, `IN`, `BETWEEN`, `AND`/`OR` | Faqat `=` |
| NULL solishtiruvi | `WHERE`dagi kabi (NULL ≠ NULL) | NULL = NULL deb hisoblaydi |

### NULL funksiyalari
| Funksiya | Argument | Qaytaradi |
| :--- | :--- | :--- |
| `NVL(a, b)` | 2 | `a` NULL emas → `a`, aks holda `b` |
| `NVL2(a, b, c)` | 3 | `a` NULL emas → `b`, aks holda `c` |
| `COALESCE(a, b, …)` | ≥2 | Birinchi NULL bo'lmagan qiymat |
| `NULLIF(a, b)` | 2 | `a = b` → NULL, aks holda `a` |

---

## 8-Qism: Imtihon Tuzoqlari — Tezkor Takrorlash

*   **Daqiqa** = `MI`, **oy** = `MM`. `HH24:MM:SS` — oyni chiqaradi (daqiqani emas).
*   **`FM`** — leading nol va trailing bo'shliqlarni o'chiradi; **`FX`** — satr formatga aniq mos kelishini talab qiladi.
*   Format modelidagi harf **registri** natija registrini belgilaydi (`Month` → `September`).
*   **`RR`** asrni aqlli aniqlaydi; **`YY`** har doim joriy asrni oladi.
*   **`TO_DATE`** satri format maskasiga mos kelmasa → xatolik.
*   **`CAST`** — format maskasiz universal konvertor.
*   **`TO_YMINTERVAL`** formati `'Y-M'` (defis bilan); `TO_INTERVALYM` funksiyasi **yo'q**.
*   **`NVL`** = 2 arg; **`NVL2`** = 3 arg (NULL bo'lmasa ham `expr2` qaytadi); **`COALESCE`** = ko'p arg, birinchi NULL bo'lmagani.
*   **`NULLIF(a, b)`**: teng → NULL, teng emas → `a`. Birinchi argument literal NULL bo'la olmaydi.
*   **`CASE`** har doim `END` bilan; `THEN`/`ELSE` turlari bir xil; mos yo'q + `ELSE` yo'q → NULL.
*   **Simple CASE** va **DECODE** faqat `=` bilan; `>`/`<`/`IN` kerak bo'lsa Searched CASE.
*   **`DECODE`** da `END` yo'q, u `NULL = NULL` ni teng deb biladi, faqat Oracle'da ishlaydi.
*   `NULL` ustidagi har qanday arifmetik amal → **`NULL`** (`100 * NULL` → NULL, 0 emas); shuning uchun hisob-kitobda `NVL`/`COALESCE`.
