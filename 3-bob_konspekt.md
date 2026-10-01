# 3-Bob: Ma'lumotlarni boshqarish (Manipulating Data) — Konspekt (imtihon yadrosi)

> Chapter 3: Manipulating Data.
> Faqat 1Z0-071 da sinaladigan bilim: DML (`INSERT`, `UPDATE`, `DELETE`), `TRUNCATE` va tranzaksiya boshqaruvi (TCL).

---

## 1. TRUNCATE — jadvalni tozalash

- **Nima qiladi:** jadvaldagi **barcha** qatorlarni tez o'chiradi.
- **Kategoriya:** DML emas, **DDL**.
- **Sintaksis:** `TRUNCATE TABLE table_name;` — `TABLE` kalit so'zi **majburiy**.
- Jadval segmentini (extent) bo'shatadi, **high-water mark**ni nollaydi, indekslarni ham tozalaydi.
- DDL bo'lgani uchun **DML triggerlarni ishga tushirmaydi** (`ON DELETE` trigger ishlamaydi).
- **Implicit COMMIT**: o'zidan oldingi va keyingi holatni avtomatik commit qiladi → **ROLLBACK qilib bo'lmaydi**.
- **FK cheklovi:** boshqa jadval FK orqali bog'langan va unda qator bo'lsa → `ORA-02266` xato. Yechim: `TRUNCATE TABLE parent CASCADE;` (faqat `ON DELETE CASCADE` o'rnatilgan bo'lsa ishlaydi).

> 🎓 **Exam Watch:** `TRUNCATE personnel;` — **SYNTAX ERROR**. Doim `TRUNCATE TABLE personnel;`.

### TRUNCATE vs DELETE

| Xususiyat | `TRUNCATE` | `DELETE` |
| :--- | :--- | :--- |
| Kategoriya | DDL | DML |
| Tranzaksiya | Implicit commit, **rollback yo'q** | Rollback mumkin |
| `ON DELETE` trigger | Ishlamaydi | Har qatorda ishlaydi |
| `WHERE` | Yo'q (butun jadval) | Bor (saralab o'chirish) |
| High-water mark | Nollanadi, xotira bo'shaydi | Joyida qoladi |
| Tezlik | Juda tez (undo yo'q) | Sekinroq (har qator undo'ga) |

---

## 2. INSERT — qator qo'shish

### Ikki shakl
1. **Ustunlarsiz:** `INSERT INTO t VALUES (...)` — qiymatlar jadvaldagi ustunlarning **aniq tartibi, soni va turiga** mos bo'lishi shart.
2. **Ustunlar bilan:** `INSERT INTO t (c1, c2) VALUES (...)` — ko'rsatilmagan ustunlarga `DEFAULT` yoki `NULL` yoziladi.

### Muhim qoidalar
- **`VALUES` bilan faqat BITTA qator qo'shiladi.** Oracle'da `VALUES (...), (...)` kabi ko'p qatorli sintaksis **ishlamaydi** (bu MySQL sintaksisi).
- Ko'p qator qo'shish uchun — **subquery** ishlatiladi, bunda `VALUES` **yozilmaydi**:
  ```sql
  INSERT INTO ship_stats (ship_id, ship_name)
  SELECT ship_id, ship_name FROM ships WHERE capacity > 2000;
  ```
- **O'tkazib yuborilgan `NOT NULL` ustun** (DEFAULT yo'q) → `ORA-01400: cannot insert NULL`.
- **Ustunlarsiz shakl xavfli:** jadvalga `ALTER TABLE ... ADD` bilan yangi ustun qo'shilsa, eski `INSERT ... VALUES` kodi buziladi (`ORA-00947: not enough values`). Shuning uchun ustunlar ro'yxatini ko'rsatish tavsiya etiladi.
- **`DEFAULT` kalit so'zi:** `INSERT INTO t (id, status) VALUES (5, DEFAULT);` — ustunning default qiymatini aniq qo'yadi.
- **Subquery/view ichiga insert:** `INSERT INTO (SELECT ... FROM t WHERE ... WITH CHECK OPTION) VALUES (...)` — `WITH CHECK OPTION` shartga mos kelmaydigan qatorni qo'shishni bloklaydi.

> 🎓 **Exam Watch (SEQUENCE):** `INSERT`da `seq.NEXTVAL` ishlatilib, so'rov **cheklov buzilishi tufayli muvaffaqiyatsiz** bo'lsa ham, sekvensiya hisoblagichi **oshib ketadi** — qiymat orqaga qaytmaydi ("gap" paydo bo'ladi).

### Ko'p jadvalli INSERT (Multi-table)
- `INSERT ALL` / `INSERT FIRST` — bitta subquery natijasini **bir nechta jadvalga** qo'shadi.
- Doim **subquery talab qiladi** (`VALUES` emas, `SELECT` bilan tugaydi).
- `INSERT ALL`: har bir `WHEN` sharti alohida tekshiriladi. `INSERT FIRST`: birinchi mos kelgan `WHEN`da to'xtaydi.

```sql
INSERT ALL
  WHEN capacity > 2000 THEN INTO big_ships (id) VALUES (ship_id)
  WHEN capacity <= 2000 THEN INTO small_ships (id) VALUES (ship_id)
SELECT ship_id, capacity FROM ships;
```

---

## 3. UPDATE — qatorlarni yangilash

- **Sintaksis:** `UPDATE t SET c1 = v1, c2 = v2 WHERE ...;`
- **`WHERE` yo'q bo'lsa → BARCHA qatorlar yangilanadi.**
- **Atomarlik:** ko'p qatorli `UPDATE`da bitta qatorda cheklov (`CHECK`, `UNIQUE`, FK...) buzilsa → **butun so'rov bekor qilinadi**, 0 qator o'zgaradi.
- **Ustun qiymatini tozalash:** `SET col = NULL` — qatorni o'chirmaydi, faqat qiymatni `NULL` qiladi.
- `SET`da ustunlar istalgan tartibda; `NOT NULL` ustunlarni ko'rsatish majburiy emas (qator allaqachon mavjud).
- **`DEFAULT` kalit so'zi:** `SET col = DEFAULT` — ustunni default qiymatiga qaytaradi.

### Subquery bilan UPDATE
- **Skalyar subquery SET'da:**
  ```sql
  UPDATE ships
  SET home_port_id = (SELECT port_id FROM ports WHERE port_name = 'Chicago')
  WHERE ship_id = 12;
  ```
- **Ko'p ustunni bir subquerydan:**
  ```sql
  UPDATE ships SET (length, capacity) =
    (SELECT length, capacity FROM ship_specs WHERE ship_specs.id = ships.ship_id);
  ```
- **`WHERE`da subquery** — qaysi qatorlarni yangilashni saralaydi.

```sql
UPDATE projects SET cost = cost * 1.20 WHERE cost * 1.20 < 1000000;
```

---

## 4. DELETE — qatorlarni o'chirish

- **Sintaksis:** `DELETE [FROM] t [WHERE ...];` — `FROM` **ixtiyoriy**.
- **`WHERE` yo'q bo'lsa → barcha qatorlar o'chadi.** Lekin DML bo'lgani uchun `ROLLBACK` mumkin, `ON DELETE` triggerlar ishlaydi (`TRUNCATE`dan farqi shu).
- `DELETE` faqat **butun qatorni** o'chiradi. Alohida ustun qiymatini tozalash uchun `DELETE` emas, `UPDATE ... SET col = NULL`.
- **`WHERE`da subquery** ishlatish mumkin:
  ```sql
  DELETE FROM ships WHERE home_port_id IN (SELECT port_id FROM ports WHERE country = 'USA');
  ```

### UPDATE (SET NULL) vs DELETE

| Operatsiya | Vazifasi | Qator soni |
| :--- | :--- | :--- |
| `UPDATE t SET col = NULL` | Bitta ustun qiymatini tozalaydi | O'zgarmaydi |
| `DELETE FROM t WHERE ...` | Shartga mos qator(lar)ni o'chiradi | Kamayadi |

---

## 5. Tranzaksiyalarni boshqarish (TCL)

### Asoslar
- **Tranzaksiya:** mantiqan bir butun DML buyruqlari ketma-ketligi.
- **Read consistency:** boshqa sessiyalar **commit qilinmagan** o'zgarishlarni **ko'rmaydi**; faqat oxirgi commit holatini ko'radi (undo segmentlar orqali).
- Bir sessiya o'zgartirayotgan qatorlar **qulflanadi** (lock) — boshqa sessiya o'sha qatorni commit/rollback'gacha yangilay olmaydi.

### COMMIT turlari
- **Explicit:** `COMMIT;` yoki `COMMIT WORK;`.
- **Implicit COMMIT quyidagilarda:**
  1. **Har qanday DDL** (`CREATE`, `ALTER`, `DROP`, `GRANT`, `REVOKE`, `TRUNCATE`).
     - **Edge case:** DDL **bajarilish xatosi** (execution error) bilan tugasa ham, undan **oldingi DML**lar **commit bo'lib ketadi**. (Faqat **syntax error**da commit bo'lmaydi — buyruq umuman ishga tushmaydi.)
  2. SQL*Plus / SQL Developer'dan **normal Exit** (precompiler dasturlari esa rollback qiladi).

### ROLLBACK va SAVEPOINT
- **`ROLLBACK;`** — oxirgi commit'dan beri barcha DML'ni bekor qiladi.
- **`SAVEPOINT sp;`** — tranzaksiya ichida oraliq nuqta.
- **`ROLLBACK TO sp;`** — faqat `sp`dan keyingi o'zgarishlarni bekor qiladi, tranzaksiya ochiq qoladi.
- **SAVEPOINT qoidalari:**
  1. Bir xil nomli SAVEPOINT qayta yaratilsa — **xato bermaydi**, eskisi ustiga yoziladi.
  2. **`COMMIT` barcha SAVEPOINT'larni o'chiradi.**
  3. Mavjud bo'lmagan/o'chgan SAVEPOINT'ga `ROLLBACK TO` → `ORA-01086` xato, rollback **bajarilmaydi**.

| TCL | Vazifasi |
| :--- | :--- |
| `COMMIT` | DML'ni **doimiy** saqlaydi, barcha SAVEPOINT'ni o'chiradi |
| `ROLLBACK` | O'zgarishlarni oxirgi commit yoki SAVEPOINT'gacha qaytaradi |
| `SAVEPOINT` | Qisman rollback nuqtasini belgilaydi |

### Misol
```sql
INSERT INTO ports (port_id, port_name) VALUES (701, 'Chicago');
SAVEPOINT sp_1;

UPDATE ships SET home_port_id = 701 WHERE ship_id = 12;
SAVEPOINT sp_2;

DELETE FROM ships WHERE ship_id = 99;

ROLLBACK TO sp_2;   -- faqat DELETE bekor; INSERT va UPDATE qoladi
COMMIT;             -- INSERT va UPDATE doimiylashadi
```
