# Oracle 1Z0-071 — 7-Bob Konspekti
## Guruh Funksiyalari Bilan Umumlashtirilgan Ma'lumot (Reporting Aggregated Data Using the Group Functions)

Konspekt faqat imtihonda tekshiriladigan bilimlarga qaratilgan: qoidalar, istisno holatlar (edge cases) va sintaktik tuzoqlar.

---

## 1-Qism: Guruh Funksiyalari — Umumiy Tushuncha

SQL funksiyalari ikki toifaga bo'linadi:
*   **Single-row (skalar) funksiyalar** — har bir qatorni alohida qayta ishlab, **har qatorga bitta natija** qaytaradi (`UPPER`, `ROUND`, `NVL` …).
*   **Group (guruh) funksiyalari** — nol yoki undan ko'p qatorni **bitta guruh** sifatida qabul qilib, **guruhga bitta natija** qaytaradi (`SUM`, `COUNT` …).

Guruh funksiyalari o'z ichida ikkiga bo'linadi:
*   **Agregat (multirow)** — guruhdagi barcha qatorlarni skanerlab, yagona qiymat qaytaradi.
*   **Analitik** — guruh ichidagi "oyna" (window) bo'yicha ishlab, **har bir qatorga** alohida qiymat qaytaradi.

`COUNT`, `SUM`, `MIN`, `MAX`, `AVG` kabi ko'pchilik funksiyalar sintaksisga qarab **ham agregat, ham analitik** ishlay oladi. Istisnolar (imtihon savoli):
*   **`MEDIAN`** — faqat agregat; analitik sifatida ishlatib bo'lmaydi.
*   **`GROUPING`, `GROUPING_ID`, `GROUP_ID`** — faqat agregat.

Agregat funksiyalar nafaqat raqam, balki matn va sana turlari bilan ham ishlay oladi; kiruvchi va chiquvchi ma'lumot turi bir xil bo'lishi shart emas (masalan `COUNT` har qanday turni qabul qiladi, lekin doim `NUMBER` qaytaradi).

---

## 2-Qism: Asosiy Agregat Funksiyalar

### COUNT
| Shakli | Nimani sanaydi |
| :--- | :--- |
| `COUNT(*)` | Barcha qatorlarni — **takror va NULL qatorlar ham** kiradi |
| `COUNT(expr)` | `expr` dagi **NOT NULL** qiymatlarni |
| `COUNT(DISTINCT expr)` | `expr` dagi **takrorlanmas, NOT NULL** qiymatlarni |

### SUM va AVG
*   Faqat **raqamli** turlar (`NUMBER`, `FLOAT` …) bilan ishlaydi.
*   NULL qiymatlarni **hisobga olmaydi** (tashlab ketadi).
*   `DISTINCT` mumkin: `AVG(DISTINCT salary)` takror maoshlarni bir marta oladi. Default — `ALL`.

### MIN va MAX
*   **Raqam, matn va sana** bilan ishlaydi.
*   NULL ni hisobga olmaydi.
*   Matn uchun: `MIN` — alifboda eng oldingi, `MAX` — eng oxirgi qiymat.
*   Sana uchun: `MIN` — eng eski (earliest), `MAX` — eng yangi (latest) sana.

### STDDEV va VARIANCE
*   Raqamli ustunning standart chetlanishi va dispersiyasini hisoblaydi; NULL ni hisobga olmaydi. (Sintaksisi `SUM`/`AVG` kabi.)

### MEDIAN
*   Guruhdagi **o'rta (median)** qiymatni qaytaradi; qatorlar soni juft bo'lsa o'rtadagi ikki qiymatning chiziqli interpolyatsiyasini oladi.
*   NULL ni hisobga olmaydi. Analitik sifatida ishlatib bo'lmaydi.

---

## 3-Qism: Guruh Funksiyalari va NULL (MUHIM QOIDALAR)

1.  **`COUNT(*)` dan tashqari barcha guruh funksiyalari NULL qiymatlarni hisobga olmaydi** (ignore NULLs).
2.  Guruhda hech bo'lmaganda bitta NOT NULL qiymat bo'lsa, funksiya o'sha qiymatlar ustida ishlaydi; NULL qatorlar hisobdan chiqadi.
3.  **Bo'sh guruh yoki barcha qiymat NULL bo'lsa:**
    *   `SUM`, `AVG`, `MIN`, `MAX`, `MEDIAN` → **`NULL`**
    *   `COUNT` → **`0`**
4.  **AVG tuzog'i:** `AVG(salary)` NULL maoshli qatorlarni **bo'luvchidan ham chiqarib tashlaydi** (faqat maoshi borlar soniga bo'linadi). NULL ni `0` deb qatnashtirish kerak bo'lsa: `AVG(NVL(salary, 0))` — bunda jami barcha qatorlar soniga bo'linadi.

> **⚠️ Shuning uchun** "o'rtacha nega noto'g'ri chiqdi?" tipidagi savollarda avval NULL qatorlar borligini tekshiring: `AVG(col)` va `AVG(NVL(col,0))` har xil natija beradi.

---

## 4-Qism: Funksiyalarni Ichma-ich Joylash (Nesting)

*   **Taqiq:** bir xil darajada skalar va agregat funksiyani to'g'ridan-to'g'ri aralashtirib bo'lmaydi (agregatsiya bo'lmaganda).
*   **Ruxsat:** agregat funksiyani skalar funksiya(lar) ichiga joylash mumkin; guruh funksiyasi eng ichki (innermost) bo'lishi shart emas.
*   **Agregatni agregat ichiga joylash** (masalan `MAX(AVG(salary))`):
    1.  **Maksimal chuqurlik — 2 daraja.** `MAX(AVG(salary))` to'g'ri; `SUM(MAX(AVG(salary)))` — xato.
    2.  **`GROUP BY` majburiy.** Agregat ichma-ich bo'lishi bilanoq so'rovda `GROUP BY` bo'lishi shart (bitta guruh bo'lsa ham).
    3.  Natija doim **yagona qator** — shuning uchun yonida boshqa ustun ko'rsatib bo'lmaydi:
        ```sql
        SELECT MAX(AVG(sq_ft)) FROM ship_cabins GROUP BY room_style;            -- TO'G'RI
        SELECT room_style, MAX(AVG(sq_ft)) FROM ship_cabins GROUP BY room_style; -- XATO
        ```

---

## 5-Qism: RANK, DENSE_RANK, FIRST, LAST (agregat shakli)

Bular guruhda **gipotetik (taxminiy) qiymat** qaysi o'ringa tushishini hisoblaydi.

**Sintaksis:**
```sql
RANK(expr_list) WITHIN GROUP (ORDER BY col_list)
aggregate_func KEEP (DENSE_RANK FIRST|LAST ORDER BY col_list)
```

*   **`RANK` vs `DENSE_RANK`:** teng qiymatlarda `RANK` reytingda **bo'shliq (gap)** qoldiradi (1,2,2,4), `DENSE_RANK` bo'shliqsiz ketadi (1,2,2,3).
*   **Moslik qoidasi:** `WITHIN GROUP` ichidagi gipotetik qiymatlar soni va turi `ORDER BY` dagi ustunlar soni va turiga **aynan mos** bo'lishi shart.

> **⚠️ Tuzoq — vergul vs qo'shtirnoq:**
> ```sql
> RANK(100000)  WITHIN GROUP (ORDER BY project_cost)  -- TO'G'RI
> RANK(100,000) WITHIN GROUP (ORDER BY project_cost)  -- XATO
> ```
> `100,000` — vergul tufayli Oracle buni **ikki alohida argument** (`100` va `000`) deb oladi, `ORDER BY` da esa bitta ustun bor → argumentlar soni mos emas → xatolik.

---

## 6-Qism: GROUP BY Klauzulasi

Qatorlarni mantiqiy kichik guruhlarga ajratadi; har bir guruh yagona butunlik sifatida qaytadi.
*   Ixtiyoriy; faqat **SELECT** doirasida ishlaydi (`INSERT`/`UPDATE`/`DELETE` da yo'q).

### Asosiy qoida (ENG KO'P TUSHADIGAN TUZOQ)
**`GROUP BY` ishlatilsa, SELECT ro'yxatidagi har bir element yo agregat funksiya ichida, yo `GROUP BY` ro'yxatida bo'lishi shart.**
*   Buzilsa: `ORA-00979: not a GROUP BY expression` (yoki `ORA-00937`).
*   **Teskarisi majburiy emas:** `GROUP BY` dagi ustun SELECT da ko'rsatilishi shart emas.

| So'rov | Status | Sabab |
| :--- | :--- | :--- |
| `SELECT room_style, AVG(sq_ft) … GROUP BY room_style` | TO'G'RI | `room_style` guruhlangan, `sq_ft` agregat ichida |
| `SELECT room_style, room_type, AVG(sq_ft) … GROUP BY room_style` | XATO | `room_type` guruhlanmagan ham, agregat ichida ham emas |
| `SELECT AVG(sq_ft) … GROUP BY room_style` | TO'G'RI | `room_style` SELECT da majburiy emas |

### Qo'shimcha qoidalar
*   **Bir nechta ustun:** `GROUP BY a, b` — `a` va `b` ning har bir noyob kombinatsiyasi alohida guruh.
*   **Ustun taxallusi (alias) TAQIQLANADI:** `GROUP BY` ichida SELECT dagi alias ishlatib bo'lmaydi.
    ```sql
    SELECT room_style AS uslub, AVG(sq_ft) … GROUP BY uslub;      -- XATO
    SELECT room_style AS uslub, AVG(sq_ft) … GROUP BY room_style; -- TO'G'RI
    ```
*   **Ifodalar mumkin:** `GROUP BY TO_CHAR(start_date,'Q')` to'liq ishlaydi.
*   **LOB cheklovi:** `BLOB`, `CLOB`, `NCLOB` ustunlari bo'yicha guruhlab bo'lmaydi.
*   **ORDER BY cheklovi:** guruhli so'rovda `ORDER BY` faqat guruhlangan ustun yoki agregat natijani ko'rsata oladi; guruhlanmagan oddiy ustun bo'yicha saralash — xato.

---

## 7-Qism: HAVING Klauzulasi

Guruhlardan hosil bo'lgan natijalarni **shart asosida filtrlaydi** — mantiqan "guruhlar uchun WHERE".
*   Faqat **SELECT** da; mavjud guruhlar ustida ishlaydi (o'zi guruh yaratmaydi).
*   Kitob qoidasi: `HAVING` faqat `GROUP BY` bor so'rovda ishlatiladi.
*   `AND`/`OR`/`NOT` bilan bir nechta shart hamda ichki so'rov (subquery) qo'llash mumkin.

### WHERE va HAVING farqi
| | WHERE | HAVING |
| :--- | :--- | :--- |
| Qachon | Guruhlashdan **oldin**, alohida qatorlarni | Guruhlashdan **keyin**, guruhlarni |
| Guruh funksiyasi | **Ishlatib bo'lmaydi** (sintaktik xato) | Bevosita ishlatiladi: `HAVING SUM(capacity) > 1000` |
| Qaysi ustun | Guruhlangan/guruhlanmagan — barchasi | Faqat guruhlangan yoki agregatga o'ralgan ustun |

> **⚠️ HAVING tuzog'i — guruhlanmagan ustun:**
> ```sql
> SELECT purpose, AVG(project_cost) FROM projects
> GROUP BY purpose
> HAVING days > 3;        -- XATO: days guruhlanmagan, agregatga ham o'ralmagan
> ```
> To'g'rilash: qatorni filtrlasa → `WHERE days > 3`; guruhni filtrlasa → `HAVING AVG(days) > 3`.

---

## 8-Qism: Bandlarning Bajarilish Tartibi

**Yozilish tartibi:**
```
SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY
```

**Mantiqiy bajarilish tartibi** (nega `WHERE` da agregat ishlamasligini tushuntiradi):
```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
```
`WHERE` guruhlar tuzilishidan **oldin** bajariladi, shuning uchun o'sha paytda hali agregat qiymat mavjud emas → `WHERE SUM(...)` xato. Guruhlar bo'yicha filtr — faqat `HAVING`.

> **Sintaktik moslashuvchanlik (edge case):** `GROUP BY` va `HAVING` o'zaro **istalgan tartibda** yozilsa ham sintaktik to'g'ri — `HAVING ... GROUP BY ...` ham ishlaydi.

---

## 9-Qism: Imtihon Tuzoqlari — Tezkor Takrorlash

*   **`COUNT(*)`** — NULL va takror qatorlarni ham sanaydi; qolgan hamma guruh funksiyasi NULL ni tashlaydi.
*   Bo'sh/hammasi-NULL guruhda: `COUNT` → **0**, qolganlari → **NULL**.
*   **`AVG(col)` ≠ `AVG(NVL(col,0))`** — NULL qatorlar bo'luvchidan chiqib ketadi.
*   **`SUM`/`AVG`** faqat raqam bilan; **`MIN`/`MAX`** raqam, matn, sana bilan.
*   **`MEDIAN`** — faqat agregat (analitik emas).
*   Agregatni agregat ichiga — **maks. 2 daraja** va **`GROUP BY` majburiy**; natija yonida boshqa ustun bo'lmaydi.
*   **`RANK(...)` ichida vergulli raqam** (`100,000`) → ikki argument deb olinadi → xato.
*   **GROUP BY asosiy qoidasi:** SELECT dagi har element yo agregatda, yo `GROUP BY` da. Teskarisi majburiy emas.
*   **`GROUP BY` da alias YO'Q**; ifoda (`TO_CHAR(...)`) mumkin; LOB bo'yicha guruhlash yo'q.
*   **`WHERE` da guruh funksiyasi YO'Q** — guruh filtri faqat `HAVING`.
*   **`HAVING`** faqat guruhlangan yoki agregatga o'ralgan ustun bilan; `GROUP BY` bilan tartibi ixtiyoriy.
*   Mantiqiy tartib: **FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY**.
