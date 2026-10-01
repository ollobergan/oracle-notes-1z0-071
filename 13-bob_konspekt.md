# 13-BOB: MANIPULATING LARGE DATA SETS (KATTA MA'LUMOTLAR TO'PLAMLARI BILAN ISHLASH)

Bu bob 1Z0-071 imtihonining uchta obyektini qamrab oladi:
1. **Subquery yordamida ma'lumotlarni boshqarish** (INSERT / UPDATE / DELETE ichida subquery).
2. **Multitable INSERT** xususiyatlari (`INSERT ALL`, `INSERT FIRST`, pivot).
3. **MERGE** (UPSERT) operatori — bitta so'rovda `INSERT` + `UPDATE` + ixtiyoriy `DELETE`.

---

# 1-BO'LIM: SUBQUERY YORDAMIDA MA'LUMOTLARNI BOSHQARISH

DML (`INSERT`, `UPDATE`, `DELETE`) so'rovlarida subquery uch joyda ishlashi mumkin:
- **Manba (source)** sifatida — `INSERT ... SELECT`;
- **Nishon (target)** sifatida — `INSERT INTO (subquery) ...`;
- **Qiymat yoki filtr** sifatida — `SET ustun = (SELECT ...)`, `WHERE ... (SELECT ...)`.

## 1.1. INSERT + Subquery (manba sifatida)
`VALUES` o'rniga `SELECT` yoziladi va subquery qaytargan **barcha qatorlar** bir yo'la kiritiladi. `VALUES` bu shaklda **ishlatilmaydi**.

```sql
INSERT INTO emp_archive (employee_id, last_name, hire_date)
SELECT employee_id, last_name, hire_date
FROM   employees
WHERE  hire_date < DATE '2010-01-01';
```
- `SELECT` ustunlari soni va ma'lumot turlari nishon ustunlar bilan mos kelishi shart.

## 1.2. INSERT INTO (subquery) — subquery orqali kiritish va WITH CHECK OPTION
Nishon o'rnida jadval emas, subquery (yoki view) ko'rsatiladi. `WITH CHECK OPTION` subquery'ning `WHERE` shartiga **mos kelmaydigan** qator kiritilishini taqiqlaydi.

```sql
INSERT INTO (SELECT department_id, department_name, location_id
             FROM departments
             WHERE location_id = 1700
             WITH CHECK OPTION)
VALUES (300, 'IT Support', 2500);
```
- Yuqoridagi qator `location_id = 2500` bo'lgani uchun `WHERE location_id = 1700` shartiga tushmaydi va **`ORA-01402: view WITH CHECK OPTION where-clause violation`** xatosini beradi.
- `WITH CHECK OPTION`siz bo'lsa, qator kiritiladi (lekin keyin shu subquery orqali ko'rinmaydi).

## 1.3. UPDATE + Subquery
Subquery `SET` qismida (skalyar yoki ko'p ustunli), `WHERE` qismida yoki ikkalasida ham bo'lishi mumkin.

**Skalyar subquery `SET`da** — subquery aniq bitta qiymat qaytarishi shart (ko'p qator qaytsa `ORA-01427` beradi):
```sql
UPDATE employees
SET    salary = (SELECT MAX(salary) FROM employees)
WHERE  employee_id = 100;
```

**Ko'p ustunni bitta subquery bilan yangilash** — qavs ichidagi ustunlar soni subquery ustunlari soniga teng:
```sql
UPDATE employees
SET    (job_id, salary) = (SELECT job_id, salary
                           FROM   employees
                           WHERE  employee_id = 205)
WHERE  employee_id = 206;
```

**Correlated (bog'langan) UPDATE** — ichki subquery tashqi qatorning ustuniga murojaat qiladi:
```sql
UPDATE employees e
SET    salary = (SELECT AVG(salary)
                 FROM   employees
                 WHERE  department_id = e.department_id);
```

## 1.4. DELETE + Subquery
`WHERE` ichidagi subquery qaysi qatorlarni o'chirishni aniqlaydi; correlated bo'lishi ham mumkin:
```sql
DELETE FROM employees e
WHERE  salary < (SELECT AVG(salary)
                 FROM   employees
                 WHERE  department_id = e.department_id);
```

---

# 2-BO'LIM: MULTITABLE INSERT

## 2.1. Asosiy tushuncha
Oddiy `INSERT ... SELECT` bitta manbadan faqat **bitta** jadvalga yozadi. Multitable INSERT esa manba subquery'sini **bir marta o'qib (single pass)**, natijani bir nechta jadvalga (yoki bitta jadvalga bir necha marta) taqsimlaydi. Ikki turi bor:
- **Unconditional (shartsiz):** har bir qator shart tekshirilmasdan barcha `INTO`larga yoziladi.
- **Conditional (shartli):** har bir qator `WHEN` shartlari orqali filtrlanadi.

## 2.2. Sintaksis

**Shartsiz:**
```sql
INSERT ALL
  INTO t1 (c1, c2) VALUES (expr1, expr2)
  INTO t2 (c1, c2) VALUES (expr3, expr4)
SELECT ... FROM ...;
```
- `ALL` kalit so'zi va oxiridagi subquery (`SELECT`) **majburiy**. `VALUES`siz, faqat subquery bilan ishlamaydi.

**Shartli:**
```sql
INSERT [ALL | FIRST]
  WHEN cond1 THEN
    INTO t1 VALUES (...)
  WHEN cond2 THEN
    INTO t2 VALUES (...)
  [ELSE
    INTO t3 VALUES (...)]
SELECT ... FROM ...;
```

## 2.3. `INSERT ALL` vs `INSERT FIRST` (shartli)
- **`INSERT ALL`** (standart): har bir qator uchun **barcha** `WHEN` shartlari tekshiriladi. Bir nechta `WHEN` TRUE bo'lsa, bitta qator **bir nechta jadvalga** tushadi.
- **`INSERT FIRST`**: `WHEN`lar yuqoridan pastga tekshiriladi; **birinchi TRUE** bo'lgani bajariladi, qolgan `WHEN`lar **o'tkazib yuboriladi**.
- **`ELSE`**: hech bir `WHEN` TRUE bo'lmagan qator uchun ishlaydi (`ALL`da ham, `FIRST`da ham).

## 2.4. Pivoting INSERT (ustunlarni qatorga aylantirish)
Bitta keng (ustunli) qatorni bir nechta tor qatorga yoyish uchun `INSERT ALL` ishlatiladi:
```sql
INSERT ALL
  INTO sales_rows (emp_id, period, amount) VALUES (emp_id, 'Q1', q1)
  INTO sales_rows (emp_id, period, amount) VALUES (emp_id, 'Q2', q2)
  INTO sales_rows (emp_id, period, amount) VALUES (emp_id, 'Q3', q3)
  INTO sales_rows (emp_id, period, amount) VALUES (emp_id, 'Q4', q4)
SELECT emp_id, q1, q2, q3, q4 FROM sales_grid;
```

## 2.5. Cheklovlar va imtihon tuzoqlari

1. **Faqat jadvallar:** nishon faqat oddiy **jadval** bo'ladi — **view va materialized view mumkin emas**, remote (uzoq) jadval ham mumkin emas.

2. **Table alias TANILMAYDI (`ORA-00904`):** `WHEN` / `INTO` / `VALUES` ichida subquery'ning jadval taxallusiga (`e.salary`) murojaat qilib bo'lmaydi.
   - *Yechim:* subquery `SELECT` ro'yxatida ustunga **column alias** berib, faqat shu nomga murojaat qiling:
     ```sql
     INSERT FIRST
       WHEN emp_sal >= 10000 THEN INTO high_sal VALUES (eid, emp_sal)
       ELSE INTO low_sal VALUES (eid, emp_sal)
     SELECT employee_id AS eid, salary AS emp_sal FROM employees;
     ```

3. **Sequence cheklovi (Oracle 19c):** rasmiy hujjatga ko'ra **multitable insert'ning hech bir qismida sequence ishlatib bo'lmaydi**. Multitable insert bitta SQL so'rovi hisoblanadi — shuning uchun `NEXTVAL`ga birinchi murojaat yangi raqam beradi, so'rovdagi **qolgan barcha murojaatlar esa aynan shu bir xil raqamni** qaytaradi (shu sabab bir nechta `INTO`da PK dublikat xavfi bor). Imtihon uchun qoida: multitable insert + sequence = ishonchsiz/taqiqlangan deb qarang.

4. **Atomiklik (all-or-nothing):** multitable insert bitta DML so'rovi. Biror `INTO`da cheklov buzilsa (masalan `ORA-00001`, `ORA-02290`), **barcha jadvallarga kiritilgan barcha qatorlar to'liq rollback** qilinadi.

---

# 3-BO'LIM: MERGE OPERATORI (UPSERT)

## 3.1. Asosiy tushuncha
`MERGE` manba (`USING`) va nishon (`INTO`) jadvallarini `ON` sharti bo'yicha solishtiradi:
- **mos kelgan** qatorlarni `UPDATE` (va ixtiyoriy `DELETE`) qiladi;
- **mos kelmagan** qatorlarni `INSERT` qiladi.

Afzalligi: alohida `UPDATE` + `INSERT` o'rniga bazaga **bir marta o'tish (single pass)**.

## 3.2. Sintaksis
```sql
MERGE INTO target t
USING {jadval | view | subquery} s
ON (t.id = s.id)
WHEN MATCHED THEN
  UPDATE SET t.col1 = s.col1, t.col2 = s.col2
  [DELETE WHERE delete_cond]
WHEN NOT MATCHED THEN
  INSERT (col1, col2)
  VALUES (s.col1, s.col2)
  [WHERE insert_cond];
```

## 3.3. Qoidalar
1. **`INTO` — bitta nishon jadval** (yoki updatable view). Majburiy.
2. **`USING`** — manba (jadval, view yoki inline subquery). Majburiy.
3. **`ON (shart)`** — moslashtiruvchi shart. Majburiy; qavs ichida yoziladi.
4. **`WHEN MATCHED` va `WHEN NOT MATCHED` — ikkalasi ham ixtiyoriy**, lekin kamida bittasi bo'lishi kerak (faqat `INSERT` yoki faqat `UPDATE` qilib ham ishlatsa bo'ladi).
5. `UPDATE SET`da `UPDATE table` yoki `INSERT`da `INSERT INTO` kalit so'zlari **yozilmaydi**.
6. **`WHEN NOT MATCHED` → INSERT** faqat **manba (source)** ustunlariga murojaat qila oladi (mos nishon qatori yo'q).

## 3.4. Imtihon tuzoqlari
1. **`ON` ustunini yangilab bo'lmaydi (`ORA-38104`):** `ON (t.id = s.id)` bo'lsa, `UPDATE SET t.id = ...` xato beradi — *Columns referenced in the ON Clause cannot be updated*.

2. **Barqaror qatorlar yo'q (`ORA-30926`):** manba (`USING`) jadvalida nishonning **bitta** qatoriga **bir nechta** manba qatori mos kelsa (dublikat join), Oracle `ORA-30926: unable to get a stable set of rows in the source tables` beradi.

3. **`DELETE WHERE` mexanizmi:** `UPDATE` bloki ichidagi `DELETE` **faqat shu MERGE o'tishida `UPDATE` qilingan** va `DELETE WHERE` shartiga tushgan qatorlarni o'chiradi. `INSERT` orqali yangi qo'shilgan yoki umuman `UPDATE` qilinmagan qatorlar, garchi shartga mos kelsa ham, **o'chirilmaydi**.

## 3.5. Misol
```sql
MERGE INTO invoices i
USING new_orders o
ON (i.cust_po = o.po_num)
WHEN MATCHED THEN
  UPDATE SET i.notes = o.sales_rep, i.inv_date = SYSDATE
  DELETE WHERE i.inv_date < DATE '2020-01-01'
WHEN NOT MATCHED THEN
  INSERT (i.inv_id, i.cust_po, i.inv_date, i.notes)
  VALUES (seq_inv.NEXTVAL, o.po_num, SYSDATE, o.sales_rep)
  WHERE o.po_num IS NOT NULL;
```
*(Eslatma: multitable insert'dan farqli o'laroq, `MERGE`da sequence ishlatish mumkin.)*

---

# 4-BO'LIM: TAQQOSLASH JADVALLARI

### `INSERT ALL` vs `INSERT FIRST`
| Xususiyat | `INSERT ALL` | `INSERT FIRST` |
| :--- | :--- | :--- |
| `WHEN` baholash | Har bir qator uchun **barchasi** tekshiriladi | **Birinchi TRUE**da to'xtaydi |
| Bitta qator → ko'p jadval | Ha (bir nechta `WHEN` TRUE bo'lsa) | Yo'q (faqat birinchi TRUE blok) |
| Standart rejim | Ha (`ALL` tushsa ham) | Yo'q (`FIRST` aniq yoziladi) |
| Tipik qo'llanish | Arxivlash, pivot | O'zaro eksklyuziv tasniflash |

### An'anaviy DML vs `MERGE`
| Xususiyat | `UPDATE` + `INSERT` alohida | `MERGE` |
| :--- | :--- | :--- |
| Bazaga o'tish | Kamida 2 marta | 1 marta (single pass) |
| Birlashuv | Har biri alohida so'rov | `INSERT`+`UPDATE`+`DELETE` bitta so'rovda |
| `ON` ustunini yangilash | Cheklov yo'q | Mumkin emas (`ORA-38104`) |

---

# 5-BO'LIM: ORA XATOLARI — TEZKOR MA'LUMOTNOMA
| Xato | Sabab |
| :--- | :--- |
| `ORA-01402` | `WITH CHECK OPTION` — kiritilgan qator subquery `WHERE` shartiga tushmaydi |
| `ORA-01427` | `SET`dagi skalyar subquery bitta qator o'rniga ko'p qator qaytardi |
| `ORA-00904` | Multitable insert'da `WHEN`/`INTO`/`VALUES` ichida table alias ishlatildi (column alias kerak) |
| `ORA-38104` | `MERGE`da `ON` shartidagi ustun `UPDATE SET` ichida yangilandi |
| `ORA-30926` | `MERGE` manbasida nishonning bir qatoriga ko'p qator mos keldi (dublikat) |
