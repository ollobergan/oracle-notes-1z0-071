# Oracle 1Z0-071 — 5-Bob Konspekti
## Single-Row Funksiyalar bilan Natijani Moslashtirish (Using Single-Row Functions to Customize Output)

Konspekt faqat imtihonda tekshiriladigan bilimlarga qaratilgan: qoidalar, istisno holatlar (edge cases) va sintaktik tuzoqlar.

---

## 1-Qism: Single-Row (Scalar) Funksiyalar — Umumiy Tushuncha

**Single-row (skalyar) funksiya** qayta ishlangan **har bir qator uchun aynan bitta** natija qaytaradi.
*   Kiruvchi parametr(lar) qabul qiladi; ba'zilari parametrsiz ishlaydi (`SYSDATE`, `USER`).
*   Ifoda (expression) ishlatilishi mumkin bo'lgan **har qanday joyda** chaqiriladi: `SELECT` ro'yxati, `WHERE`, `ORDER BY`, `GROUP BY` / `HAVING`, hamda `INSERT ... VALUES`, `UPDATE ... SET`, `DELETE ... WHERE`.

### Funksiyalarning uch toifasi (imtihon uchun muhim farq)
| Toifa | Nechta qator → nechta natija | Misollar |
| :--- | :--- | :--- |
| **Single-row (scalar)** | N qator → N natija | `UPPER`, `ROUND`, `SUBSTR`, `ADD_MONTHS` |
| **Multiple-row (aggregate)** | N qator → **1** natija | `SUM`, `AVG`, `COUNT`, `MAX`, `MIN` |
| **Analytic** | N qator → N natija, lekin **guruh/oyna** doirasida | `LAG`, `LEAD`, `RANK`, `STDDEV` |

### DUAL jadvali
Jadvalga bog'lanmagan holda bitta qiymat olish uchun (`SYSDATE`, hisob-kitob, funksiyani sinash):
*   `SYS` ga tegishli, ma'lumotlar lug'atining (Data Dictionary) bir qismi.
*   Bitta ustun (`DUMMY VARCHAR2(1)`), bitta qator (qiymati `'X'`).
*   Har doim aynan bitta qator qaytaradi.

---

## 2-Qism: Matnli Funksiyalar (Character Functions)

### 2.1. Registr (case) funksiyalari
*   **`LOWER(str)`** — hammasini kichik harfga.
*   **`UPPER(str)`** — hammasini katta harfga.
*   **`INITCAP(str)`** — har so'zning birinchi harfi katta, qolgani kichik.
    *   **Tuzoq:** so'zlarni nafaqat bo'shliq, balki **har qanday harf-raqam bo'lmagan belgi** (`-`, `_`, `/`, `.`) orqali ajratadi.
    *   `INITCAP('oracle-database-12c')` → `Oracle-Database-12C`

### 2.2. To'ldirish — LPAD / RPAD
```sql
LPAD(string, padded_length [, pad_string])   -- chapdan to'ldiradi
RPAD(string, padded_length [, pad_string])   -- o'ngdan to'ldiradi
```
*   `pad_string` ko'rsatilmasa — standart holatda **bo'sh joy (space)**.
*   **Tuzoq (truncation):** agar `padded_length` dastlabki satrdan **qisqa** bo'lsa, xatolik emas — satr o'ngdan `padded_length` gacha **qirqiladi**.
    *   `LPAD('Database', 4, '*')` → `Data`

### 2.3. Kesish — LTRIM / RTRIM / TRIM
*   **`LTRIM(s1 [, s2])` / `RTRIM(s1 [, s2])`** — `s1` ning chap/o'ng tomonidan `s2` dagi belgilarni olib tashlaydi.
    *   `s2` — bu so'z emas, **belgilar to'plami**; mos kelgan belgilarni birma-bir tozalaydi, birinchi mos kelmagan belgida to'xtaydi.
    *   `s2` berilmasa — bo'sh joylarni tozalaydi.
    *   `LTRIM('xyxba', 'xy')` → `ba`
*   **`TRIM([LEADING | TRAILING | BOTH] trim_char FROM source)`** — boshidan/oxiridan/ikkalasidan tozalaydi (standart `BOTH`).
    *   **Cheklov:** `trim_char` faqat **bitta belgi** bo'ladi. Ko'p belgi (`'xy'`) berilsa — **xatolik**.
    *   `TRIM(BOTH '*' FROM '**ORACLE**')` → `ORACLE`

### 2.4. Birlashtirish — CONCAT va ||
*   **`CONCAT(s1, s2)`** — **faqat 2 ta** parametr. `CONCAT(s1, s2, s3)` → **sintaktik xatolik**.
*   Uch yoki undan ko'pini birlashtirish: `CONCAT(CONCAT(s1, s2), s3)` yoki **`||`** operatori (tavsiya etiladi).

### 2.5. Qidirish va kesib olish — INSTR / SUBSTR
```sql
INSTR(source, search [, start_position [, occurrence]])   -- POZITSIYANI (son) qaytaradi
SUBSTR(source, start_position [, length])                 -- BO'LAKni (matn) qaytaradi
```
**INSTR:**
*   `start_position` standart 1; **manfiy** bo'lsa oxiridan chapga qidiradi, lekin qaytgan indeks har doim chapdan (1-indexed) hisoblanadi.
*   `occurrence` standart 1 (nechanchi uchrashuv).
*   Topilmasa → **0** qaytaradi.
*   `INSTR('Mississippi', 'is', 1, 2)` → `5`

**SUBSTR:**
*   `start_position` = **0** bo'lsa, **1** deb olinadi; **manfiy** bo'lsa oxiridan sanaladi.
*   `length` berilmasa — satr oxirigacha.
*   **Tuzoq:** `length` **0 yoki manfiy** bo'lsa — xatolik emas, **`NULL`** qaytadi.
*   `SUBSTR('Oracle Certified Associate', 8, 9)` → `Certified`

### 2.6. LENGTH / REPLACE / TRANSLATE
*   **`LENGTH(str)`** — belgilar sonini qaytaradi. `str` **NULL** bo'lsa → natija **`NULL`** (0 emas!).
*   **`REPLACE(str, search [, replace])`** — `search` ning barcha uchrashuvini `replace` ga almashtiradi. `replace` berilmasa — `search` **o'chirib tashlanadi**.
    *   `REPLACE('JACK AND JUE', 'J', 'BL')` → `BLACK AND BLUE`
*   **`TRANSLATE(str, from, to)`** — `from` dagi **har bir belgini** `to` dagi mos pozitsiyadagi belgiga **belgi-ba-belgi** almashtiradi (so'z emas, belgilar jadvali).
    *   **REPLACE vs TRANSLATE farqi (klassik tuzoq):** REPLACE satrni butun bo'lak sifatida almashtiradi; TRANSLATE har bir belgini alohida xaritalaydi. `from` `to` dan uzun bo'lsa, ortiqcha belgilar **o'chiriladi**.
    *   `TRANSLATE('1a2b3c', '123', 'xyz')` → `xaybzc`

### 2.7. SOUNDEX
Talaffuzga asoslangan fonetik kod qaytaradi (o'xshash eshitiladigan so'zlarni topish uchun).
*   **Tuzoq:** ustunni to'g'ridan-to'g'ri fonetik kod bilan solishtirish **noto'g'ri** — har ikki tomon ham `SOUNDEX` ga o'tkazilishi shart:
    ```sql
    WHERE SOUNDEX(lastname) = SOUNDEX('Franklin');   -- TO'G'RI
    ```

---

## 3-Qism: Sonli Funksiyalar (Number Functions)

### 3.1. ROUND va TRUNC (sonlar uchun)
*   **`ROUND(n [, decimals])`** — matematik yaxlitlaydi.
*   **`TRUNC(n [, decimals])`** — kasr qismni shunchaki qirqadi (yaxlitlamaydi).
*   `decimals` berilmasa — standart **0** (butun songacha).
*   **Manfiy `decimals`** — verguldan **chapga** (birlar, o'nlar, yuzlar...) ta'sir qiladi:

| Ifoda | Natija |
| :--- | :--- |
| `ROUND(45.926, 2)` | `45.93` |
| `TRUNC(45.926, 2)` | `45.92` |
| `ROUND(45.926, -1)` | `50` |
| `TRUNC(45.926, -1)` | `40` |
| `ROUND(45.926, -3)` | `0` (yuzlar xonasigacha: 45 < 500) |

### 3.2. MOD va REMAINDER (asosiy arxitektura farqi!)
Ikkalasi ham bo'lish qoldig'ini beradi, lekin ichki yaxlitlash har xil:
*   **`MOD(n1, n2)`** = `n1 - n2 * FLOOR(n1/n2)` → **FLOOR** ishlatadi.
*   **`REMAINDER(n1, n2)`** = `n1 - n2 * ROUND(n1/n2)` → **ROUND** (eng yaqin butun; 0.5 bo'lsa juft songa) ishlatadi.

| Ifoda | Hisoblash | Natija |
| :--- | :--- | :--- |
| `MOD(5, 3)` | 5 − 3·FLOOR(1.66)=3·1 | **`2`** |
| `REMAINDER(5, 3)` | 5 − 3·ROUND(1.66)=3·2 | **`-1`** |
| `MOD(1.5, 1)` | 1.5 − 1·1 | **`0.5`** |
| `REMAINDER(1.5, 1)` | 1.5 − 1·2 (0.5→juft 2) | **`-0.5`** |

*   `MOD(n1, 0)` → **`n1`** ni qaytaradi (0 ga bo'lish xatosi emas).

### 3.3. Boshqa muhim sonli funksiyalar
*   **`CEIL(n)`** — yuqoriga eng yaqin butun songa: `CEIL(45.1)` → `46`, `CEIL(-45.9)` → `-45`.
*   **`FLOOR(n)`** — pastga eng yaqin butun songa: `FLOOR(45.9)` → `45`, `FLOOR(-45.1)` → `-46`.
*   **`ABS(n)`** — modul (mutlaq qiymat): `ABS(-15)` → `15`.
*   **`SIGN(n)`** — ishorasi: manfiy → `-1`, nol → `0`, musbat → `1`.
*   **`POWER(m, n)`** — `m` ning `n`-darajasi: `POWER(2, 3)` → `8`.
*   **`SQRT(n)`** — kvadrat ildiz. Manfiy sondan → **xatolik**.

---

## 4-Qism: Sana Funksiyalari (Date Functions)

`DATE` turi asr, yil, oy, kun, soat, daqiqa, soniyani saqlaydi.

### 4.1. Joriy sana/vaqt funksiyalari
*   **`SYSDATE`** — serverning joriy sanasi va vaqti (DATE).
*   **`SYSTIMESTAMP`** — server vaqti + kasr soniyalar + vaqt mintaqasi (TIMESTAMP WITH TIME ZONE).
*   **`CURRENT_DATE` / `CURRENT_TIMESTAMP`** — sessiya vaqt mintaqasidagi vaqt.

### 4.2. Sana arifmetikasi
*   **`Sana + son`** / **`Sana − son`** → **kun** qo'shadi/ayiradi.
*   **`Sana1 − Sana2`** → ikki sana orasidagi **kunlar soni** (kasr bilan).
*   **`Sana1 + Sana2`** → **xatolik** (sanalarni qo'shib bo'lmaydi).
*   Soat qo'shish uchun: `Sana + N/24`.

### 4.3. ADD_MONTHS
`ADD_MONTHS(date, n)` — `n` oy qo'shadi (manfiy bo'lsa ayiradi).
*   **Edge case:** boshlang'ich sana oyning **oxirgi kuni** bo'lsa, natija ham maqsad oyining **oxirgi kuni** bo'ladi.
    *   `ADD_MONTHS('31-JAN-2024', 1)` → `29-FEB-2024` (kabisa yili)

### 4.4. MONTHS_BETWEEN
`MONTHS_BETWEEN(date1, date2)` — oylardagi farq (`date1 − date2` mantig'i).
*   `date1 < date2` bo'lsa natija **manfiy**.
*   **Butun son** chiqishi: ikkala sana ham oyning bir xil kunida, **yoki** ikkalasi ham o'z oylarining oxirgi kunida bo'lsa. Aks holda kasr (31 kunlik oy asosida).
*   `MONTHS_BETWEEN('01-APR-2024', '01-JUN-2024')` → `-2`

### 4.5. LAST_DAY / NEXT_DAY
*   **`LAST_DAY(date)`** — shu sana oyining **oxirgi kunini** qaytaradi: `LAST_DAY('15-FEB-2024')` → `29-FEB-2024`.
*   **`NEXT_DAY(date, 'kun_nomi')`** — berilgan sanadan **keyingi** birinchi shu nomli hafta kunini qaytaradi (sananing o'zini qamramaydi): `NEXT_DAY('01-JAN-2024', 'MONDAY')`.

### 4.6. EXTRACT
`EXTRACT(qism FROM date)` — sanadan bitta qismni **son** sifatida ajratadi:
```sql
EXTRACT(YEAR FROM SYSDATE)    -- yil
EXTRACT(MONTH FROM SYSDATE)   -- oy (1-12)
EXTRACT(DAY FROM SYSDATE)     -- kun
```
*   `HOUR`, `MINUTE`, `SECOND` faqat TIMESTAMP turlariga qo'llanadi (oddiy DATE dan ularni ajratib bo'lmaydi).

### 4.7. Sanalarni ROUND / TRUNC qilish
Format maskasi (format mask) asosida:
*   **`TRUNC(date, 'YEAR')`** → joriy yil 1-Yanvar (vaqt 00:00:00).
*   **`TRUNC(date, 'MONTH')`** → oy 1-kuni.
*   **`TRUNC(date)`** (maska yo'q) → vaqt qismini 00:00:00 ga tushiradi (kun saqlanadi).
*   **`ROUND(date, 'MONTH')`** → 1–15 kun bo'lsa shu oy 1-kuni; **16-kun va undan keyin** keyingi oy 1-kuni.
*   **`ROUND(date, 'YEAR')`** → yilning birinchi yarmida bo'lsa shu yil, ikkinchi yarmida keyingi yil 1-Yanvari.

---

## 5-Qism: Analitik Funksiyalar (Analytic Functions)

Skalyar va aggregate o'rtasidagi tur: **har bir qator uchun alohida natija** qaytaradi, lekin hisobni **qatorlar guruhi (window/partition)** doirasida bajaradi.

### 5.1. Sintaksis — OVER majburiy
```sql
FUNC(...) OVER (
    [PARTITION BY ustun]     -- mantiqiy guruhlarga ajratadi
    [ORDER BY ustun]         -- guruh ichidagi tartib
    [windowing_clause]
)
```
*   **Tuzoq:** `OVER` ichidagi `ORDER BY` faqat analitik oynaning tartibini belgilaydi — so'rov oxiridagi global `ORDER BY` dan **mustaqil** ishlaydi.

### 5.2. LAG va LEAD
Joriy qatordan oldingi (`LAG`) yoki keyingi (`LEAD`) qator qiymatini o'qiydi:
```sql
LAG(expr [, offset [, default]])
LEAD(expr [, offset [, default]])
```
*   `offset` standart **1**.
*   `default` — qator mavjud bo'lmasa qaytadigan qiymat; berilmasa **`NULL`**.

### 5.3. Boshqalar
*   **`STDDEV(expr)`** — standart og'ish.
*   **`PERCENTILE_CONT(p)`** — `p` persentilga mos qiymat (chiziqli interpolatsiya), `p` ∈ [0, 1].

---

## 6-Qism: Funksiyalarni Ichma-Ich Joylashtirish (Nesting)

Bir funksiya natijasini ikkinchisiga parametr sifatida berish mumkin — cheklanmagan darajada.
*   **Bajarilish tartibi:** har doim **eng ichkaridan tashqariga**.
*   Ichki funksiya qaytargan tur tashqi funksiya kutgan turga **mos** bo'lishi shart; aralash turlarda avtomatik (implicit) konversiyaga ishonmay, aniq (`TO_CHAR`, `TO_DATE`) konversiya tavsiya etiladi.

**Klassik misol — SUBSTR + INSTR:**
```sql
-- Vergulyadan keyingi 2 belgini (shtat kodi) olish:
SUBSTR(address2, INSTR(address2, ', ') + 2, 2)
```
Avval `INSTR` vergul pozitsiyasini topadi → songa 2 qo'shiladi → `SUBSTR` shu nuqtadan 2 belgi kesadi.

---

## 7-Qism: Imtihon Tuzoqlari — Tezkor Takrorlash

*   **`CONCAT`** faqat **2** parametr; 3 ta → sintaktik xato. Ko'pi uchun `||`.
*   **`TRIM`** `trim_char` faqat **1 belgi**; ko'p belgi → xato (`LTRIM`/`RTRIM` esa belgilar to'plamini qabul qiladi).
*   **`SUBSTR`** `length` ≤ 0 → **`NULL`** (xato emas); `start_position` = 0 → 1 deb olinadi.
*   **`INSTR`** topilmasa → **0**; manfiy `start` → oxiridan qidiradi, indeks baribir chapdan.
*   **`LPAD`/`RPAD`** `padded_length` qisqa bo'lsa → satr **qirqiladi**.
*   **`LENGTH(NULL)`** → **`NULL`** (0 emas).
*   **`INITCAP`** so'zlarni har qanday non-alfanumerik belgi bo'yicha ajratadi.
*   **`REPLACE`** = bo'lak almashtirish; **`TRANSLATE`** = belgi-ba-belgi xaritalash.
*   **`ROUND`/`TRUNC`** manfiy `decimals` → verguldan chapga; qiymat uzunligidan katta manfiy → **0**.
*   **`MOD`** = FLOOR asosida; **`REMAINDER`** = ROUND asosida → ishoralar/natijalar farq qiladi. `MOD(n, 0)` → `n`.
*   **`CEIL`** yuqoriga, **`FLOOR`** pastga (manfiy sonlarda yo'nalishga e'tibor).
*   **Sana + Sana** → xato; **Sana − Sana** → kunlar soni.
*   **`ADD_MONTHS`** / **`LAST_DAY`** — oy oxiri qoidasi.
*   **`MONTHS_BETWEEN(d1, d2)`**: `d1 < d2` → manfiy.
*   **`ROUND(date,'MONTH')`** — 16-kundan keyingi oyga o'tadi.
*   **`SOUNDEX`** — har ikki tomonni ham `SOUNDEX` ga o'tkazish shart.
*   **Analitik funksiya** har doim **`OVER`** bilan; `OVER` ichidagi `ORDER BY` global `ORDER BY` dan mustaqil.
*   **Nesting** — ichkaridan tashqariga bajariladi; turlar mos kelishi shart.
