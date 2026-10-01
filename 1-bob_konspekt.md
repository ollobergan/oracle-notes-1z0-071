# 1-Bob: Oracle va SQL — Konspekt (imtihon yadrosi)

> Faqat 1Z0-071 da sinaladigan bilim. Strategiya va ortiqcha gap yo'q.

---

## 1. ERD → Relatsion model

ERD = **mantiqiy model** (logical). RDBMS = uning **jismoniy amalga oshirilishi** (physical).

| ERD tushunchasi | Jadvaldagi muqobili |
| :--- | :--- |
| Entity (sub'ekt) | Table (jadval) |
| Attribute (atribut) | Column (ustun) |
| Instance / record | Row (qator) |

**Munosabat turlari:** 1:1, 1:N, N:M.

**N:M qoidasi (muhim):** Relatsion bazada ko'pga-ko'p munosabat **saqlanmaydi**. U doim oraliq **junction (intersection) jadval** orqali ikkita 1:N munosabatga bo'linadi:

```
[SHIPS] (1) ──< (N) [ROSTER] (N) >── (1) [EMPLOYEES]
```

---

## 2. Kalitlar (Keys)

- **Primary Key (PK):** qatorni yagona (unique) aniqlaydi.
  - PK = **`NOT NULL` + `UNIQUE`** birikmasi.
  - Bitta jadvalda faqat **bitta** PK bo'ladi (bir yoki bir nechta ustundan — composite).
- **Foreign Key (FK):** boshqa (yoki shu) jadvalning **PK yoki UNIQUE** ustuniga ishora qiladi.
  - **Referential integrity** — FK qiymati ota jadvalda mavjud bo'lishini (yoki NULL bo'lishini) kafolatlaydi.

---

## 3. Normalizatsiya (qisqa)

- **1NF:** takrorlanuvchi guruhlar yo'q, har katakda bitta atomar qiymat.
- **2NF:** 1NF + kalit bo'lmagan ustun composite PK ning **bir qismiga** emas, to'liq PK ga bog'liq.
- **3NF:** 2NF + **tranzitiv bog'liqlik** yo'q (kalit bo'lmagan ustun boshqa kalit bo'lmagan ustunga bog'liq emas).

> OLTP tizimlar odatda 3NF da bo'ladi. BCNF/4NF/5NF — imtihon uchun shart emas, nomini bilsang yetarli.

---

## 4. Relatsion baza va SQL

- **RDBMS** — ma'lumotning **persistent** (barqaror) saqlanishi: dastur tugasa ham ma'lumot qoladi.
- **SQL** — baza bilan muloqot qiluvchi **deklarativ** til. SQL bazaning o'zi emas, interfeysi.
- SQL ni interfeyslardan (SQL Developer, SQL*Plus) yoki dasturlash tillaridan (Java, PHP, C#...) yuborish mumkin.

---

## 5. SQL buyruqlarining 6 turi

| Tur | Vazifasi | Misollar |
| :--- | :--- | :--- |
| **DDL** | Obyekt strukturasi | `CREATE`, `ALTER`, `DROP`, `RENAME`, `TRUNCATE`, `GRANT`, `REVOKE`, `COMMENT`, `FLASHBACK`, `PURGE` |
| **DML** | Obyekt ichidagi ma'lumot | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE` |
| **TCL** | Tranzaksiyani boshqarish | `COMMIT`, `ROLLBACK`, `SAVEPOINT` |
| **Session Control** | Sessiyani sozlash | `ALTER SESSION`, `SET ROLE` |
| **System Control** | Instance ni sozlash | `ALTER SYSTEM` |
| **Embedded SQL** | SQL ni 3GL tillarga joylash | — |

**⚠ Imtihon tuzog'i:** `ALTER SESSION` va `ALTER SYSTEM` — DDL **EMAS** (nomida ALTER bor-u, lekin Session/System Control).

---

## 6. DDL va Implicit Commit (kritik)

Har qanday **DDL buyrug'i avtomatik `COMMIT` beradi** (implicit commit).

Ssenariy: Bitta sessiyada ochiq `INSERT`/`UPDATE` (commit qilinmagan) turganda `CREATE`/`TRUNCATE` kabi DDL bajarilsa — avvalgi barcha DML o'zgarishlari **qaytarib bo'lmas holda saqlanadi**.

---

## 7. TCL

- **`COMMIT`** — DML o'zgarishlarini doimiy saqlaydi.
- **`ROLLBACK`** — oxirgi commitdan beri bo'lgan o'zgarishlarni bekor qiladi.
- **`SAVEPOINT`** — tranzaksiya ichida oraliq nuqta; `ROLLBACK TO savepoint_name` faqat o'shagacha qaytaradi.

---

## 8. TRUNCATE (DDL) vs DELETE (DML)

| Mezon | `TRUNCATE` | `DELETE` |
| :--- | :--- | :--- |
| Kategoriya | DDL | DML |
| Rollback | **Yo'q** (implicit commit) | **Ha** |
| `WHERE` | Yo'q (hamma qator o'chadi) | Ha (shart bilan) |
| DML triggerlar | Chaqirmaydi | Chaqiradi (`BEFORE/AFTER DELETE`) |
| Mexanizm | Segment darajasida tozalaydi (tez) | Qatorma-qator (sekinroq, undo log) |

---

## 9. SELECT so'rovi

**3 imkoniyat (capabilities):**
1. **Projection** — kerakli ustunlarni tanlash.
2. **Selection** — `WHERE` bilan kerakli qatorlarni filtrlash.
3. **Joining** — jadvallarni umumiy ustun orqali biriktirish.

```sql
CREATE TABLE ships (
    ship_id   NUMBER,
    ship_name VARCHAR2(20),
    capacity  NUMBER
);

INSERT INTO ships (ship_id, ship_name, capacity)
VALUES (1, 'Codd Crystal', 2052);

SELECT ship_name, capacity
FROM   ships;
```

- **`SELECT` ma'lumotni o'zgartirmaydi** — faqat o'qiydi.

---

## Imtihon xulosasi (Exam Watch)

1. **DDL → implicit commit.** Ochiq DML + DDL = avtomatik COMMIT.
2. **N:M** relatsion bazada saqlanmaydi → junction jadval → 2 ta 1:N.
3. **PK** = `NOT NULL` + `UNIQUE`; jadvalda bitta PK.
4. **FK** → boshqa jadval PK yoki UNIQUE ustuniga ishora qiladi.
5. **`TRUNCATE`** = DDL (rollback yo'q, trigger yo'q); **`DELETE`** = DML (rollback bor, trigger bor).
6. **`ALTER SESSION` / `ALTER SYSTEM`** DDL emas.

> **Tekshirish kerak:** savollar soni / vaqt / o'tish foizi — kitobdagi raqamlar eski versiyaga tegishli. Oracle rasmiy sahifasidan tasdiqlang.
