# @Oracle 1Z0-071 — 9-Bob Konspekti

## Subquery'lar Yordamida So'rovlarni Hal Qilish (Using Subqueries to Solve Queries)

Konspekt faqat imtihonda tekshiriladigan bilimlarga qaratilgan: qoidalar, istisno holatlar (edge cases) va sintaktik tuzoqlar.

---

## 1-Qism: Subquery — Umumiy Tushuncha

**Subquery (ichki so'rov)** — boshqa bir SQL buyrug'i ichida qavs ichida joylashgan `SELECT` so'rovi. Tashqi buyruq **outer / parent query** deb ataladi.

Subquery joylashishi mumkin bo'lgan joylar:


| Joy                                        | Nomi / vazifasi                             |
| ------------------------------------------ | ------------------------------------------- |
| `WHERE`                                    | filtrlash sharti (eng ko'p uchraydi)        |
| `HAVING`                                   | guruhlangan natijani filtrlash              |
| `FROM`                                     | **Inline view** (vaqtinchalik jadval)       |
| `SELECT` ustunlar ro'yxati                 | **Scalar subquery** (1 qiymat)              |
| `UPDATE … SET` / `WHERE`, `DELETE … WHERE` | ma'lumotni o'zgartirish/o'chirish           |
| `INSERT … VALUES` / `SELECT` qismi         | qiymat yoki manba                           |
| `WITH … AS (…)`                            | **Subquery factoring** (nomlangan subquery) |
| `CREATE TABLE … AS SELECT`, `CREATE VIEW`  | DDL da manba                                |


Qat'iy qoidalar:

1. Subquery **har doim qavs ichida** bo'ladi: `(SELECT …)`.
2. Subquery qaytargan qiymat turi uni solishtirayotgan ustun/operator turiga **mos (compatible)** bo'lishi shart.
3. **Nesterlash chuqurligi**: `WHERE`/`HAVING` ichida 255 darajagacha; `FROM` (inline view) ichida cheklov yo'q.
4. `ORDER BY`: an'anaviy `WHERE`/`HAVING` subquery ichida mantiqsiz, lekin inline view ichida yoki `FETCH`/`OFFSET` bilan **ruxsat etiladi**.

---



## 2-Qism: Subquery Turlari (Tasnif)


| Tur                 | Ta'rifi                                                                  |
| ------------------- | ------------------------------------------------------------------------ |
| **Single-row**      | aniq **1 qator** qaytaradi                                               |
| **Multiple-row**    | **0, 1 yoki ko'p** qator qaytaradi                                       |
| **Multiple-column** | bir vaqtda **bir nechta ustun** qaytaradi                                |
| **Scalar**          | aniq **1 qator, 1 ustun** (bitta qiymat) qaytaradi                       |
| **Correlated**      | tashqi so'rov ustuniga murojaat qiladi, har qator uchun qayta bajariladi |
| **Inline view**     | `FROM` dagi subquery (vaqtinchalik jadval o'rnida)                       |


Bu turlar bir-birini istisno qilmaydi (masalan, correlated subquery ham scalar bo'lishi mumkin).

---



## 3-Qism: Single-Row Subquery

Tashqi so'rovga **bitta qiymat** qaytaradi va **single-row operatorlar** bilan ishlatiladi:

`=` , `<>` / `!=` / `^=` (teng emas) , `>` , `>=` , `<` , `<=`

```sql
-- Al Smith ishlaydigan kemadagi boshqa xodimlar
SELECT employee_id, last_name, first_name
FROM   employees
WHERE  ship_id = (SELECT ship_id
                  FROM   employees
                  WHERE  last_name = 'Smith' AND first_name = 'Al')
  AND NOT (last_name = 'Smith' AND first_name = 'Al');
```

Edge case'lar (EXAM CRITICAL):

- **>1 qator qaytsa** → `ORA-01427: single-row subquery returns more than one row`. Bu **run-time** (bajarilish paytidagi) xato, kompilyatsiya xatosi emas.
- **0 qator qaytsa** → subquery `NULL` qaytaradi. `WHERE ship_id = NULL` har doim `UNKNOWN`, shuning uchun so'rov **xatosiz, lekin 0 qator** qaytaradi.

---



## 4-Qism: Multiple-Row Subquery

0, 1 yoki ko'p qator qaytaradi; **multiple-row operatorlar** talab qiladi:

- `IN` — to'plamdagi biror qiymatga teng bo'lsa `TRUE`.
- `NOT IN` — to'plamdagi hech bir qiymatga teng bo'lmasa `TRUE`.
- `ANY` **/** `SOME` — to'plamdagi **kamida bitta** qiymat shartni qanoatlantirsa `TRUE`.
  - `> ANY` → **minimum**dan katta; `< ANY` → **maximum**dan kichik; `= ANY` ⇔ `IN`.
- `ALL` — to'plamdagi **barcha** qiymatlar shartni qanoatlantirsa `TRUE`.
  - `> ALL` → **maximum**dan katta; `< ALL` → **minimum**dan kichik.

> **Operator ekvivalentliklari:** `= ANY` ⇔ `IN`; `<> ALL` (`!= ALL`) ⇔ `NOT IN`. (`<> ANY` esa `NOT IN` EMAS — u deyarli doim `TRUE`, chunki har qanday qiymat to'plamdagi kamida bittasidan farq qiladi.)

```sql
-- Familiyasi 'Smith' bo'lgan xodim ishlaydigan kemalardagi barcha xodimlar
SELECT ship_id, last_name
FROM   employees
WHERE  ship_id IN (SELECT ship_id FROM employees WHERE last_name = 'Smith');

-- 'Luxury' kategoriyasidagi barcha mahsulotdan qimmatroq mahsulotlar
SELECT *
FROM   products
WHERE  price > ALL (SELECT price FROM products WHERE category = 'Luxury');
```

> **⚠️** `NOT IN` **+** `NULL` **tuzog'i (eng ko'p tushadigan savol):** agar subquery to'plamida **bitta bo'lsa ham** `NULL` bo'lsa, butun `NOT IN` sharti `UNKNOWN` ga aylanadi va so'rov **0 qator** qaytaradi.
> Sababi: `x NOT IN (1, 2, NULL)` ⇔ `x != 1 AND x != 2 AND x != NULL`; oxirgi shart doim `UNKNOWN`.
> Yechim: subquery ichida `WHERE col IS NOT NULL`, yoki `NOT EXISTS` ishlatish.

> **⚠️** `ANY`**/**`ALL` **+ bo'sh to'plam:** subquery **0 qator** qaytarsa, `> ALL` doim `TRUE`, `> ANY` doim `FALSE`.
>
> Operatorni **oddiy jumla** sifatida o'qing: `> ALL` = "hammasidan katta", `> ANY` = "kamida bittasidan katta".
>
> **`ALL` + bo'sh to'plam → doim `TRUE`:** "to'plamdagi hammasidan katta bo'lsin" degan shart. To'plamda hech kim yo'q bo'lsa, sizdan katta bo'lishi kerak bo'lgan birorta ham raqib yo'q — shartni buzadigan ("men sendan kichik emasman" deydigan) qiymat ham yo'q. Shuning uchun shart **avtomatik bajariladi**. Matematikada bu **"vacuous truth"** (bo'sh haqiqat) deyiladi: bo'sh to'plam ustidagi "barcha ... uchun" shart doim `TRUE`.
> ```sql
> SELECT * FROM products
> WHERE price > ALL (SELECT price FROM products WHERE category = 'YO_QNARSA');
> -- categoriya bo'sh → BARCHA qator chiqadi (shart hamma uchun TRUE)
> ```
>
> **`ANY` + bo'sh to'plam → doim `FALSE`:** "to'plamdagi kamida bittasidan katta bo'lsin" degan shart. To'plamda hech kim yo'q bo'lsa, kattaroq bo'lishim mumkin bo'lgan birorta ham qiymat yo'q — demak "kamida bittasidan katta" bo'lib bo'lmaydi, chunki o'sha "bitta" ham mavjud emas.
> ```sql
> SELECT * FROM products
> WHERE price > ANY (SELECT price FROM products WHERE category = 'YO_QNARSA');
> -- categoriya bo'sh → 0 qator (shart hech kim uchun TRUE emas)
> ```
>
> **Mnemonika:** `ALL` → *all pass* (hamma o'tadi); `ANY` → *nobody passes* (hech kim o'tmaydi).
>
> **Diqqat — bu NULL holatidan boshqa:** bo'sh to'plam = **0 qator** (yuqoridagi aniq `TRUE`/`FALSE`). Ichida `NULL` bo'lgan to'plam = qator **bor**, lekin qiymati NULL → `UNKNOWN` (pastdagi tuzoq). `price > ALL ()` → `TRUE`, ammo `price > ALL (NULL)` → `UNKNOWN`.

> **⚠️** `ANY`**/**`ALL` **to'plamida** `NULL` **(**`NOT IN` **tuzog'ining ukasi):**
>
> - `ALL` **+ NULL** — `NOT IN` kabi buziladi: `x > ALL (10, 20, NULL)` ichidagi `x > NULL` → `UNKNOWN`, butun `ALL` sharti `TRUE` bo'la olmaydi → qatorlar yo'qoladi.
> - `ANY` **+ NULL** — chidamliroq: bitta qiymat `TRUE` bersa yetarli, NULL e'tiborsiz qoladi. `x > ANY (10, 20, NULL)` — agar `x > 10` yoki `x > 20` bo'lsa `TRUE`.
> - Xavfsizlik uchun subquery ichida `WHERE col IS NOT NULL` qo'yish tavsiya etiladi.

---



## 5-Qism: Multiple-Column Subquery

Subquery bir nechta ustun qaytaradi; tashqi tomonda ustunlar **qavs ichida juftlanadi**.

```sql
SELECT employee_id, manager_id, department_id
FROM   employees
WHERE  (manager_id, department_id) IN
       (SELECT manager_id, department_id
        FROM   employees
        WHERE  employee_id IN (101, 102));
```

- Chapdagi ustunlar **soni va tartibi** subquery `SELECT` ro'yxatiga, **turlari** esa mos kelishi shart; aks holda xato.

> **⚠️ Pairwise vs Nonpairwise (juftlangan vs juftlanmagan) — imtihon tuzog'i:**
> Yuqoridagi `(manager_id, department_id) IN (SELECT …)` — **pairwise**: ustunlar **birga, bitta juftlik** sifatida solishtiriladi (aniq `manager_id`+`department_id` kombinatsiyasi mos kelishi kerak).
> **Nonpairwise** esa har ustunni **alohida** subquery bilan solishtiradi:
>
> ```sql
> SELECT employee_id, manager_id, department_id
> FROM   employees
> WHERE  manager_id    IN (SELECT manager_id    FROM employees WHERE employee_id IN (101,102))
>   AND  department_id IN (SELECT department_id FROM employees WHERE employee_id IN (101,102));
> ```
>
> Bu ikkisi **boshqa-boshqa natija** beradi: pairwise — faqat aynan mos juftliklar; nonpairwise — ikki to'plamning **kesishmasi** (kross-kombinatsiyalar ham tushadi). Imtihonda "qaysi biri ko'proq/kamroq qator qaytaradi?" tipidagi savol bo'ladi.

---



## 6-Qism: Scalar Subquery

Aniq **1 qator, 1 ustun** qaytaradi va **ifoda (expression)** ishlatilishi mumkin bo'lgan deyarli har joyda turadi: `SELECT` ro'yxati, `WHERE`, `ORDER BY`, `VALUES`, funksiya argumenti va h.k.

```sql
-- Har bir xodim yonida uning bo'limidagi o'rtacha maosh
SELECT last_name, salary,
       (SELECT AVG(salary) FROM employees x
        WHERE  x.department_id = e.department_id) AS dept_avg
FROM   employees e;
```

- Agar scalar subquery **0 qator** qaytarsa → qiymat `NULL`.
- **>1 qator** qaytarsa → `ORA-01427`.

---



## 7-Qism: Correlated Subquery

Ichki so'rov **tashqi so'rov ustuniga murojaat qiladi** va tashqi so'rovning **har bir qatori uchun qayta bajariladi**.

Bajarilish tartibi: tashqidan 1 qator olinadi → ichki so'rov shu qator qiymati bilan bajariladi → natija filtrga ishlatiladi → keyingi qatorga o'tiladi.

```sql
-- Har bir xona uslubida o'rtacha maydondan katta kabinalar
SELECT A.ship_cabin_id, A.room_style, A.sq_ft
FROM   ship_cabins A
WHERE  A.sq_ft > (SELECT AVG(sq_ft)
                  FROM   ship_cabins B
                  WHERE  B.room_style = A.room_style);
```


|                      | Non-correlated           | Correlated                    |
| -------------------- | ------------------------ | ----------------------------- |
| Mustaqil bajarilishi | Ha (alohida run bo'ladi) | Yo'q (tashqi aliasga bog'liq) |
| Bajarilish soni      | 1 marta                  | har qator uchun               |


> **⚠️** Correlated subquery'ni **alohida ajratib** run qilsangiz `ORA-00904: invalid identifier` chiqadi — tashqi jadval aliasi topilmaydi.



### UPDATE va DELETE da correlated subquery

```sql
-- SET va WHERE da birga: har portga biriktirilgan kema sonini yozish
UPDATE ports P
SET    capacity = (SELECT COUNT(*) FROM ships S WHERE S.home_port_id = P.port_id)
WHERE  EXISTS   (SELECT 1 FROM ships S WHERE S.home_port_id = P.port_id);

-- DELETE: har xona turi/stili bo'yicha eng kichik balkonli kabinani o'chirish
DELETE FROM ship_cabins S1
WHERE  S1.balcony_sq_ft = (SELECT MIN(balcony_sq_ft)
                           FROM   ship_cabins S2
                           WHERE  S1.room_type  = S2.room_type
                             AND  S1.room_style = S2.room_style);
```

> `SET col = (subquery)` da subquery **0 qator** qaytarsa, ustunga `NULL` yoziladi — buning oldini olish uchun yuqoridagi kabi `WHERE EXISTS` bilan cheklash kerak.

---



## 8-Qism: EXISTS va NOT EXISTS (Semijoin)

- `EXISTS` — subquery **kamida 1 qator** qaytarsa `TRUE`.
- `NOT EXISTS` — subquery **0 qator** qaytarsa `TRUE`.
- Odatda **correlated** holda ishlatiladi; qiymatni emas, **qator mavjudligini** tekshiradi.

```sql
-- Kamida bitta kema biriktirilgan portlar
SELECT port_id, port_name
FROM   ports P1
WHERE  EXISTS (SELECT 1 FROM ships S1 WHERE P1.port_id = S1.home_port_id);

-- Hech qanday kema biriktirilmagan portlar
SELECT port_id, port_name
FROM   ports P1
WHERE  NOT EXISTS (SELECT 1 FROM ships S1 WHERE P1.port_id = S1.home_port_id);
```

Muhim nuqtalar:

- `SELECT` ro'yxatiga nima yozilishi (`*`, `1`, `'X'`, `NULL`) **ahamiyatsiz** — optimizer faqat qator borligini tekshiradi.
- **NULL xavfsiz:** `NOT EXISTS` to'plamda `NULL` bo'lsa ham `NOT IN` kabi buzilmaydi (4-Qism tuzog'iga alternativa).


|                   | `IN`                      | `EXISTS`                   |
| ----------------- | ------------------------- | -------------------------- |
| Nimani tekshiradi | qiymat to'plamda bormi    | subquery qator qaytaradimi |
| `SELECT` ro'yxati | ustun ko'rsatilishi shart | muhim emas                 |
| `NOT` + `NULL`    | 0 qator (tuzoq!)          | ta'sir qilmaydi            |


---



## 9-Qism: Inline View (`FROM` dagi subquery)

`FROM` da jadval o'rnida turadigan subquery. **Top-N** va guruhlangan natija ustidan qayta ishlash uchun ishlatiladi.

```sql
-- Eng qimmat 5 mahsulot (Top-N)
SELECT *
FROM   (SELECT product_name, price
        FROM   products
        ORDER BY price DESC)
WHERE  ROWNUM <= 5;
```

- Inline view ichida `ORDER BY` **ruxsat etiladi**.
- Inline view'ga taxallus berib, ustunlariga tashqaridan murojaat qilish mumkin.

> **⚠️** `ROWNUM` **TUZOQLARI (EXAM CRITICAL):**
> `ROWNUM` — qator natijaga **qo'shilgandan keyin** beriladigan psevdo-ustun. Shundan ikki muhim oqibat kelib chiqadi:
>
> **(a)** `ROWNUM = n` **(n > 1) va** `ROWNUM > n` **→ doim 0 qator.**
>
> ```sql
> SELECT * FROM products WHERE ROWNUM = 1;   -- ISHLAYDI (1 qator)
> SELECT * FROM products WHERE ROWNUM = 5;   -- 0 QATOR
> SELECT * FROM products WHERE ROWNUM > 1;   -- 0 QATOR
> ```
>
> Sababi: birinchi nomzod qatorga `ROWNUM = 1` beriladi. Agar shart `> 1` yoki `= 5` bo'lsa, u o'tolmay tashlanadi; keyingi qator yana `1` bo'lishga urinadi — hech qachon o'tmaydi. Faqat `ROWNUM = 1`, `ROWNUM <= n`, `ROWNUM < n` ishlaydi.
>
> **(b)** `ROWNUM` ****`ORDER BY` **dan OLDIN beriladi** — shuning uchun Top-N da `ORDER BY` **albatta inline view ichida** bo'lishi shart:
>
> ```sql
> -- XATO MANTIQ: tartiblanmagan dastlabki 5 qator olinadi, keyin saralanadi
> SELECT product_name, price FROM products
> WHERE ROWNUM <= 5 ORDER BY price DESC;
>
> -- TO'G'RI: avval inline view'da saralanadi, keyin ROWNUM qo'llanadi
> SELECT * FROM (SELECT product_name, price FROM products ORDER BY price DESC)
> WHERE ROWNUM <= 5;
> ```

> **Oracle 12c+ / 19c** `FETCH` **(Row Limiting Clause)** — Top-N uchun zamonaviy usul, inline view + `ROWNUM` o'rnini bosadi:
>
> ```sql
> SELECT product_name, price
> FROM   products
> ORDER BY price DESC
> FETCH FIRST 5 ROWS ONLY;        -- yoki: FETCH FIRST 10 PERCENT ROWS ONLY
> ```
>
> `WITH TIES` oxirgi o'rindagi teng qiymatlarni ham qo'shadi; `OFFSET n ROWS` bilan sahifalash qilinadi.

---



## 10-Qism: WITH Clause (Subquery Factoring)

So'rov boshida subquery bloklariga **nom berish** va ularni asosiy so'rovda bir necha bor ishlatish. O'qilishni yaxshilaydi, bir xil hisob-kitobni takrorlamaydi.

```sql
WITH
port_bookings AS (
    SELECT P.port_id, P.port_name, COUNT(S.ship_id) AS ct
    FROM   ports P JOIN ships S ON P.port_id = S.home_port_id
    GROUP BY P.port_id, P.port_name
)
SELECT port_name
FROM   port_bookings
WHERE  ct = (SELECT MAX(ct) FROM port_bookings);
```

Scope (ko'rinish) qoidalari:

- `WITH` so'rovning **eng yuqorisida** e'lon qilinadi.
- Nom **o'z ta'rifi ichida** chaqirilmaydi (recursive bundan mustasno, pastga qarang).
- Nom **undan keyingi** `WITH` bloklarida va asosiy `SELECT` da erkin ishlatiladi.

> **Oracle 19c — Recursive Subquery Factoring (recursive** `WITH`**):** ierarxik/rekursiv ma'lumot uchun (`CONNECT BY` ga ANSI muqobil). `UNION ALL` bilan **anchor** (boshlang'ich) va **recursive** qismdan iborat:
>
> ```sql
> WITH emp_tree (employee_id, manager_id, lvl) AS (
>     SELECT employee_id, manager_id, 1
>     FROM   employees
>     WHERE  manager_id IS NULL          -- anchor
>   UNION ALL
>     SELECT e.employee_id, e.manager_id, t.lvl + 1
>     FROM   employees e JOIN emp_tree t ON e.manager_id = t.employee_id  -- recursive
> )
> SELECT * FROM emp_tree;
> ```

---



## 11-Qism: Imtihon Tuzoqlari — Tezkor Takrorlash

- **Single-row operator** (`=`, `>`, …) ishlatilgan joyda subquery **>1 qator** qaytarsa → `ORA-01427` (**run-time**).
- Single-row subquery **0 qator** qaytarsa → `NULL`, so'rov xatosiz **0 qator** qaytaradi.
- `NOT IN` **to'plamida** `NULL` bo'lsa → butun so'rov **0 qator**. `NOT EXISTS` bilan hal qilinadi.
- `ANY`**/**`ALL`: `> ANY`=min'dan katta, `< ANY`=max'dan kichik, `> ALL`=max'dan katta, `< ALL`=min'dan kichik; `= ANY`⇔`IN`, `<> ALL`⇔`NOT IN`.
- `ALL` **to'plamida NULL** → `NOT IN` kabi buziladi; `ANY` NULL'ga chidamli.
- **Multiple-column**: `(c1, c2) IN (SELECT c1, c2 …)` — ustun soni, tartibi va turlari mos bo'lishi shart. **Pairwise** (juftlik) ≠ **nonpairwise** (alohida `IN`lar) — har xil natija.
- **Scalar subquery**: 0 qator → `NULL`; >1 qator → `ORA-01427`.
- **Correlated**ni alohida run qilsa → `ORA-00904`.
- `EXISTS` `SELECT` ro'yxatiga nima yozilishi muhim emas; `NOT EXISTS` NULL'ga chidamli.
- **Nesterlash**: `WHERE`/`HAVING` da 255 daraja, `FROM` (inline view) da cheklovsiz.
- `ORDER BY` subquery ichida faqat inline view'da yoki `FETCH`/`OFFSET` bilan.
- `UPDATE … SET col = (subquery)` 0 qator qaytarsa `NULL` yozadi — `WHERE EXISTS` bilan cheklang.
- **Top-N**: eski — `FROM (… ORDER BY …) WHERE ROWNUM <= n`; 19c — `FETCH FIRST n ROWS ONLY`.
- `ROWNUM`: faqat `= 1`, `<= n`, `< n` ishlaydi; `= n` (n>1) va `> n` → 0 qator. `ROWNUM` `ORDER BY` dan oldin beriladi → Top-N da `ORDER BY` inline view ichida bo'lishi shart.

