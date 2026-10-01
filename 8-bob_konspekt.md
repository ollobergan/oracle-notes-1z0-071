# Oracle 1Z0-071 — 8-Bob Konspekti
## Ko'p Jadvaldan Ma'lumot Ko'rsatish (Displaying Data from Multiple Tables)

Konspekt faqat imtihonda tekshiriladigan bilimlarga qaratilgan: qoidalar, istisno holatlar (edge cases) va sintaktik tuzoqlar.

---

## 1-Qism: Join — Umumiy Tushuncha va Tasnif

Ma'lumotlar ortiqchalikni (redundancy) kamaytirish uchun normallashtirilib, turli jadvallarga bo'linadi. Bu jadvallar odatda **Primary Key → Foreign Key** munosabati orqali bog'lanadi.

**Join (birlashtirish)** — ikki yoki undan ortiq jadvalni umumiy ustun yoki mantiqiy shart asosida bog'lab, yagona natija qaytaruvchi `SELECT` so'rovi.

Imtihonda birlashtirishlar bir necha mezon bo'yicha tasniflanadi:

| Mezon | Turlari |
| :--- | :--- |
| **Natijaga ko'ra** | **Inner** (faqat mos qatorlar) / **Outer** (mos + mossiz qatorlar, NULL bilan) |
| **Shart operatoriga ko'ra** | **Equijoin** (`=`) / **Non-equijoin** (`<`, `>`, `BETWEEN`, `!=` …) |
| **Arxitekturaga ko'ra** | `NATURAL JOIN`, `JOIN…USING`, `JOIN…ON`, `CROSS JOIN`, `Self-join` |

Ikkita sintaksis oilasi mavjud:
*   **ANSI/ISO SQL** — `JOIN … ON/USING`, `NATURAL JOIN`, `CROSS JOIN`, `LEFT/RIGHT/FULL OUTER JOIN`.
*   **Oracle eski (proprietary)** — jadvallar `FROM` da vergul bilan, shart `WHERE` da; tashqi birlashma uchun `(+)`.

---

## 2-Qism: Dekart Ko'paytmasi va CROSS JOIN

**Cartesian product (dekart ko'paytmasi)** — birinchi jadvalning har bir qatori ikkinchi jadvalning har bir qatoriga bog'lanadi. Natija qatorlar soni = `m × n`.

Qachon hosil bo'ladi (imtihon tuzog'i):
*   `CROSS JOIN` **ataylab** ishlatilganda.
*   Eski sintaksisda `FROM a, b` yozilib, `WHERE` da birlashtirish sharti **tushib qolganda** (eng ko'p uchraydigan xato sabab).
*   `NATURAL JOIN` da ikki jadvalda **umumiy nomli ustun bo'lmasa** — xato bermaydi, balki jimgina dekart ko'paytma qaytaradi.

```sql
-- Aniq CROSS JOIN (ON/USING ishlatilmaydi — ishlatilsa xato)
SELECT e.last_name, d.department_name
FROM   employees e
CROSS JOIN departments d;

-- Eski sintaksisda bir xil natija (shart yo'q)
SELECT e.last_name, d.department_name
FROM   employees e, departments d;
```

> **⚠️** `CROSS JOIN` bilan `ON` yoki `USING` **ishlatib bo'lmaydi** — bu sintaktik xato.

---

## 3-Qism: Jadval Taxalluslari (Table Aliases) va Ustun Noaniqligi

Agar ikkala jadvalda ham bir xil nomli ustun bo'lsa, uni prefikssiz yozish **`ORA-00918: column ambiguously defined`** xatosini beradi. Ustunni aniqlash (qualify) kerak:
1.  To'liq jadval nomi bilan: `employees.employee_id`
2.  Jadval taxallusi bilan: `e.employee_id`

Taxalluslarga doir qoidalar:
*   Taxallus **faqat shu so'rov bajarilishi davomida** amal qiladi; bazada saqlanadigan ob'ekt emas.
*   `FROM` da jadvalga taxallus berilsa (`FROM employees e`), so'rovning qolgan qismida (`SELECT`, `ON`, `WHERE`, `ORDER BY`) **faqat taxallus** ishlatilishi shart. To'liq nomni qayta ishlatish — xato (`ORA-00904`).

```sql
SELECT employees.last_name        -- XATO: taxallus berilgan, to'liq nom mumkin emas
FROM   employees e JOIN addresses a ON e.employee_id = a.employee_id;
-- To'g'ri: e.last_name
```

---

## 4-Qism: NATURAL JOIN

Ikki jadvaldagi **barcha bir xil nomli ustunlar** bo'yicha avtomatik tenglik (`=`) birlashmasi. Shart ochiq yozilmaydi.

Qat'iy qoidalar (tez-tez tushadigan tuzoqlar):
1.  `ON` yoki `USING` bilan **birga ishlatib bo'lmaydi** — sintaktik xato.
2.  Birlashtiruvchi (umumiy) ustun so'rovning **hech yerida prefiks/taxallus bilan yozilmaydi** → `ORA-25155: column used in NATURAL join cannot have qualifier`.
3.  Agar umumiy nomli ustunlar bir nechta bo'lsa (`employee_id` va `manager_id`), `NATURAL JOIN` **hammasi bo'yicha birdaniga** birlashtiradi — barchasi bir vaqtda mos kelishi kerak.
4.  Umumiy ustunlarning **turlari mos kelmasa** — xato.
5.  Umumiy nomli ustun **umuman bo'lmasa** — dekart ko'paytma (2-Qismga qarang).

```sql
SELECT employee_id, last_name, street_address   -- employee_id prefikssiz
FROM   employees NATURAL JOIN addresses;
```

---

## 5-Qism: JOIN … USING

Umumiy nomli ustunlar bir nechta bo'lganda **aynan qaysi ustun(lar)** bo'yicha birlashishni ko'rsatadi.

Qoidalar:
1.  Ustun **har doim qavs ichida**: `USING (employee_id)`. Qavssiz — xato.
2.  `USING` dagi ustun so'rovning hech yerida **prefiks/taxallusga ega bo'lmaydi** → `ORA-25155`. (Boshqa, birlashtirmaydigan ustunlar prefiks olishi mumkin.)
3.  Bir nechta ustun vergul bilan: `USING (employee_id, office_name)`.
4.  `USING` dagi ustunlar nomi ikkala jadvalda bir xil bo'lishi shart (`NATURAL JOIN` kabi); nomlari farq qilsa — `ON` kerak.

```sql
SELECT employee_id, e.last_name, a.street_address  -- employee_id prefikssiz
FROM   employees e JOIN addresses a USING (employee_id);
```

---

## 6-Qism: JOIN … ON — eng moslashuvchan usul

`ON` istalgan shartni yozishga imkon beradi:
*   Ustun nomlari **turlicha** bo'lgan jadvallarni bog'lash (`ON s.home_port_id = p.port_id`).
*   Ustunlarga **erkin prefiks/taxallus** qo'yish (hatto majburiy, agar nom noaniq bo'lsa).
*   **Non-equijoin** (tenglikdan boshqa) shartlar.
*   Qo'shimcha shartlar: `ON … AND …`.

> **Ustun ko'rinishidagi farq (muhim):**
> `NATURAL JOIN` va `USING` da birlashtiruvchi ustun natijada **bir marta** chiqadi.
> `ON` da esa har ikki jadvalning ustuni **alohida** saqlanadi (`SELECT *` da ikki marta ko'rinadi).

---

## 7-Qism: Equijoin va Non-Equijoin

| | Equijoin | Non-Equijoin |
| :--- | :--- | :--- |
| Operator | faqat `=` | `<`, `>`, `<=`, `>=`, `BETWEEN`, `!=` … |
| Munosabat | aniq moslik | diapazon / nisbiy munosabat |
| Foreign Key | odatda bor | shart emas |
| Tipik qo'llanish | jadvallarni kalit bo'yicha bog'lash | ballni bahoga, qiymatni diapazonga moslash |

```sql
-- Non-equijoin: ball qaysi harfli bahoga tushishini topish
SELECT s.score_id, s.test_score, g.grade
FROM   scores s
JOIN   grading g ON s.test_score BETWEEN g.score_min AND g.score_max;
```

---

## 8-Qism: Outer Joins (Tashqi Birlashmalar)

Mos qatorlardan tashqari, **mos jufti bo'lmagan qatorlarni** ham qaytaradi; yetishmagan tomon ustunlari `NULL` bo'ladi.

*   **`LEFT [OUTER] JOIN`** — **chap** jadvalning barcha qatori; o'ngda mos yo'q bo'lsa, o'ng ustunlar `NULL`.
*   **`RIGHT [OUTER] JOIN`** — **o'ng** jadvalning barcha qatori; chapda mos yo'q bo'lsa, chap ustunlar `NULL`.
*   **`FULL [OUTER] JOIN`** — ikkala jadvalning barcha qatori; mossizlari `NULL` bilan.

Qoidalar:
*   `OUTER` kalit so'zi **ixtiyoriy** (`LEFT JOIN` = `LEFT OUTER JOIN`).
*   Outer join `ON`, `USING` yoki `NATURAL` bilan ishlaydi (`NATURAL LEFT JOIN` ham mumkin).
*   `CROSS JOIN` ning outer varianti yo'q.

```sql
SELECT e.last_name, d.department_name
FROM   employees e
LEFT OUTER JOIN departments d ON e.department_id = d.department_id;
-- Bo'limi yo'q xodim ham chiqadi; uning department_name = NULL
```

---

## 9-Qism: Oracle Eski Sintaksisi — `(+)` Operatori

ANSI standartidan oldingi usul; imtihonda savollar chiqadi. `(+)` belgisi **ma'lumot yetishmaydigan (NULL qaytishi kutilgan)** tomon ustuniga qo'yiladi.

```sql
-- LEFT OUTER JOIN ekvivalenti: barcha SHIP chiqadi, mos PORT bo'lmasa NULL
SELECT s.ship_name, p.port_name
FROM   ships s, ports p
WHERE  s.home_port_id = p.port_id(+);
```
> Mantiq: `(+)` **"kam" tomonga** qo'yiladi. Yuqorida `ports` tomonda yetishmovchilik bo'lishi mumkin, shuning uchun `(+)` o'sha tomonda; natijada `ships` to'liq saqlanadi (chap tashqi birlashma).

Qat'iy cheklovlar (EXAM CRITICAL):
1.  Faqat `WHERE` da ishlaydi; ANSI `JOIN … ON` bilan **aralashtirib bo'lmaydi**.
2.  **FULL OUTER JOIN yaratib bo'lmaydi** — `a.col(+) = b.col(+)` xato.
3.  Shartda **`OR`** yoki **`IN`** bilan birga ishlatib bo'lmaydi.
4.  `(+)` qatnashgan shartning ikkinchi tomoni **subquery** bo'lishi mumkin emas.

---

## 10-Qism: Self-Join (Jadvalni O'ziga Birlashtirish)

Jadvalni o'ziga bog'lab, qatorlarni o'zaro solishtirish yoki ierarxiyani (boshliq–xodim) aks ettirish.

*   `FROM` da jadval **ikki marta**, har biriga **har xil taxallus** berish **majburiy**.
*   Ierarxiyaning eng yuqori qatori (`manager_id IS NULL`) ham chiqishi uchun `LEFT OUTER JOIN` ishlatiladi.

```sql
SELECT e.last_name   AS xodim,
       m.last_name   AS boshliq
FROM   employees e
LEFT OUTER JOIN employees m ON e.manager_id = m.employee_id;
```

---

## 11-Qism: Ko'p Jadvalli Birlashma (Multitable Joins)

ANSI sintaksisda jadvallar **zanjirli** bog'lanadi; har bir `JOIN` ga o'z `ON`/`USING` sharti kerak.

```sql
SELECT p.port_name, s.ship_name, c.room_number
FROM   ports p
JOIN   ships s        ON p.port_id = s.home_port_id
JOIN   ship_cabins c  ON s.ship_id = c.ship_id;
```
*   n ta jadvalni bog'lash uchun odatda **n−1** ta birlashtirish sharti bo'ladi; shart yetmasa — dekart ko'paytma.
*   Turli join turlarini bitta so'rovda aralashtirish mumkin (`INNER` + `LEFT` …).

---

## 12-Qism: Imtihon Tuzoqlari — Tezkor Takrorlash

*   **Dekart ko'paytma** `FROM a, b` da `WHERE` sharti tushib qolsa yoki `NATURAL JOIN` da umumiy ustun bo'lmasa sodir bo'ladi.
*   **`CROSS JOIN`** bilan `ON`/`USING` — xato.
*   **Bir xil nomli ustun prefikssiz** → `ORA-00918`.
*   Taxallus berilgach **to'liq nom ishlatish** → `ORA-00904`.
*   **`NATURAL JOIN`/`USING`**: birlashtiruvchi ustun hech yerda prefiks olmaydi → `ORA-25155`.
*   **`NATURAL JOIN`** bilan `ON`/`USING` — xato; umumiy ustunlar turi mos kelmasa — xato.
*   **`USING`** ustuni doim qavs ichida; qavssiz — xato.
*   **`NATURAL`/`USING`** da birlashtiruvchi ustun natijada **bir marta**; **`ON`** da ikki marta.
*   **`OUTER`** so'zi ixtiyoriy; `(+)` **kam/NULL tomonga** qo'yiladi.
*   **`(+)`**: faqat `WHERE`; `OR`/`IN`/subquery bilan emas; FULL OUTER yasay olmaydi; ANSI `JOIN` bilan aralashmaydi.
*   **Self-join**: jadval ikki xil taxallus bilan; eng yuqori ierarxiya uchun `LEFT OUTER JOIN`.
*   **n jadval → n−1 shart**; yetmasa dekart ko'paytma.
