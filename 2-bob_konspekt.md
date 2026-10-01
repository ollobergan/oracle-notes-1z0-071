# 2-Bob: DDL bilan jadval yaratish va boshqarish — Konspekt (imtihon yadrosi)

> Chapter 2: Using DDL Statements to Create and Manage Tables.
> Faqat 1Z0-071 da sinaladigan bilim.

---

## 1. Baza obyektlari (8 ta)

| Obyekt | Vazifasi |
| :--- | :--- |
| **TABLE** | Ma'lumotni ustun/qator ko'rinishida saqlaydi |
| **VIEW** | Nomlangan `SELECT` (virtual jadval), o'zi ma'lumot saqlamaydi |
| **INDEX** | Qidiruvni tezlashtiradi |
| **SEQUENCE** | Unikal raqam generatori (odatda PK uchun) |
| **SYNONYM** | Obyektga taxallus (Private / Public) |
| **CONSTRAINT** | Ma'lumot yaxlitligi qoidasi |
| **USER** | Hisob qaydnomasi va obyekt egasi |
| **ROLE** | Imtiyozlar to'plami |

---

## 2. Schema vs Nonschema obyektlar

- **Schema** = bitta `USER` ga tegishli obyektlar to'plami; nomi user nomi bilan bir xil.
- **Schema obyektlar:** `TABLE`, `VIEW`, `INDEX`, `SEQUENCE`, `CONSTRAINT`, `PRIVATE SYNONYM`.
- **Nonschema obyektlar:** `USER`, `ROLE`, `PUBLIC SYNONYM` — baza darajasida, alohida userga tegishli emas.

---

## 3. Namespace (nomlar fazosi) qoidalari

| Namespace | Obyektlar | Oqibat |
| :--- | :--- | :--- |
| Baza darajasi | `USER`, `ROLE` bitta umumiy namespace; `PUBLIC SYNONYM` alohida | Bir nomda ham USER, ham ROLE bo'lmaydi |
| Sxema asosiy | `TABLE`, `VIEW`, `SEQUENCE`, `PRIVATE SYNONYM` bitta namespace | TABLE va VIEW (yoki SEQUENCE) bir xil nomda **bo'lmaydi** |
| Alohida | `INDEX` va `CONSTRAINT` har biri alohida namespace | TABLE va INDEX/CONSTRAINT bir xil nomda bo'lishi **mumkin** |

---

## 4. Nomlash qoidalari (Naming Rules)

**Standard (tirnoqsiz) nom:**
1. Uzunligi **1–30 belgi**. (19c versiyada 1-128 belgi)
2. Birinchi belgi — **harf** (raqam yoki maxsus belgi emas).
3. Qolgani: harf, raqam, `$`, `_`, `#`. Boshqa belgi mumkin emas.
4. **Case-insensitive**, lekin bazada **UPPERCASE** saqlanadi.
5. Reserved word (`SELECT`, `DATE`, `NUMBER`...) nom bo'la olmaydi.

**Quoted (`"..."`) nom:**
- Har qanday belgidan boshlanishi, bo'sh joy va reserved word saqlashi mumkin.
- **Case-sensitive**; har safar tirnoq bilan yozilishi shart. Tavsiya etilmaydi.

---

## 5. DDL va Implicit Commit

- DDL: `CREATE`, `ALTER`, `DROP`, `RENAME`, `TRUNCATE`, `GRANT`, `REVOKE`, `FLASHBACK`, `PURGE`, `COMMENT`.
- Har DDL buyrug'idan **oldin va keyin** avtomatik `COMMIT`. DDL ni `ROLLBACK` qilib bo'lmaydi.

---

## 6. Ma'lumot turlari (Data Types)

| Tur | Qoida | Limit / Default |
| :--- | :--- | :--- |
| **`CHAR(n)`** | Fixed-length; `n` gacha bo'sh joy bilan to'ldiriladi | Default `1`, max **2000** bayt |
| **`VARCHAR2(n)`** | Variable-length; `n` **majburiy** | Max **4000** bayt (12c EXTENDED: 32767) |
| **`NUMBER(p,s)`** | `p`=precision (1–38), `s`=scale (−84…127) | `p` berilib `s` yo'q → `s=0` |
| **`DATE`** | Yil…Sekund | −4712 … 9999 yil |
| **`TIMESTAMP(n)`** | DATE + kasr sekund | `n`=0–9, default **6** |
| **`TIMESTAMP WITH [LOCAL] TIME ZONE`** | + vaqt zonasi | `n`=0–9, default 6 |
| **`INTERVAL YEAR(n) TO MONTH`** | Yil–oy oralig'i | `n` default 2 |
| **`INTERVAL DAY(n1) TO SECOND(n2)`** | Kun…sekund oralig'i | `n1` def 2, `n2` def 6 |
| **`BLOB` / `CLOB`** | Katta binar / matn | Terabayt darajasida |

**`CHAR` vs `VARCHAR2` (asosiy farq):**
- **`CHAR(n)`** — *fixed-length*. Kelgan qiymat `n` dan qisqa bo'lsa, Oracle uni **o'ng tomondan bo'sh joy (space) bilan `n` gacha to'ldiradi** va shu holda saqlaydi. Ya'ni har doim aniq `n` bayt egallaydi.
- **`VARCHAR2(n)`** — *variable-length*. Faqat **kelgan qiymat uzunligicha** saqlaydi, bo'sh joy bilan to'ldirmaydi. `n` — bu ruxsat etilgan maksimum.
- Oqibati: `CHAR` ustunidagi `'ABC'` aslida `'ABC   '` bo'lib saqlanadi; tenglashtirishda (`=`) bu farq muammo keltirib chiqarishi mumkin (blank-padded comparison semantikasi).

**LOB (`BLOB`/`CLOB`/`NCLOB`) da TAQIQLANADI:** `PRIMARY KEY`, `UNIQUE`, `DISTINCT`, `GROUP BY`, `ORDER BY`, `JOIN` sharti.

---

## 7. NUMBER(p,s) yaxlitlash va ORA-01438

- `s` musbat → verguldan keyin `s` xonagacha yaxlitlanadi.
- `s` manfiy → butun qismdagi `10^|s|` xonalari nolga yaxlitlanadi.
- Butun qism xonalari **`p − s`** dan oshsa → **ORA-01438** (qiymat juda katta).
  - `NUMBER(3,2)` ga `10.59` → xato (ruxsat: 3−2=1 butun xona, bizda 2 ta).
  - `NUMBER(5,-2)` ga `1059.34` → xato emas, `1100` bo'lib saqlanadi.

---

## 8. Constraintlar (5 tur)

| Constraint | Ma'nosi |
| :--- | :--- |
| **`NOT NULL`** | NULL kiritishni taqiqlaydi |
| **`UNIQUE`** | Qiymat unikal (lekin NULL ga ruxsat) |
| **`PRIMARY KEY`** | `NOT NULL` + `UNIQUE`; jadvalda faqat 1 ta |
| **`FOREIGN KEY`** | Boshqa/shu jadval `PK` yoki `UNIQUE` ustuniga ishora |
| **`CHECK`** | Shart `TRUE` yoki `UNKNOWN` (NULL) bo'lishini talab qiladi |

**In-line vs Out-of-line:**
- `NOT NULL` — **faqat In-line** yoziladi (Out-of-line → xato).
- Composite (ko'p ustunli) `PRIMARY KEY` / `UNIQUE` / `FOREIGN KEY` — **faqat Out-of-line**.
- Nom berilmasa Oracle `SYS_C...` prefiksli nom beradi.

---

## 9. DROP COLUMN vs SET UNUSED

| Mezon | `DROP COLUMN` | `SET UNUSED` |
| :--- | :--- | :--- |
| Tezlik | Sekin (resurs talab) | **Juda tez** (mantiqiy bayroq) |
| Disk | Darhol bo'shaydi | `DROP UNUSED COLUMNS` gacha bo'shamaydi |
| Tiklash | Imkonsiz | **Imkonsiz** |
| Ko'rinish | Tuzilmadan o'chadi | Yashirinadi (`DESC`/`SELECT` da ko'rinmaydi) |

- Jadvalda **kamida 1 ta aktiv ustun** qolishi shart — hammasini birdan DROP/UNUSED qilib bo'lmaydi.
- Keyin tozalash: `ALTER TABLE ... DROP UNUSED COLUMNS;`

---

## 10. External Table

- Metadata bazada, ma'lumot OS faylida (CSV/TXT).
- **READ-ONLY:** `INSERT`/`UPDATE`/`DELETE` taqiqlanadi.
- Ustiga `INDEX` va `CONSTRAINT` yaratib bo'lmaydi.
- `SELECT` bilan oddiy jadvaldek o'qiladi.

---

## 11. Asosiy sintaksis

```sql
-- In-line NOT NULL + DEFAULT, Out-of-line PK va CHECK
CREATE TABLE cruises (
    cruise_id    NUMBER,
    cruise_name  VARCHAR2(30) CONSTRAINT cruises_name_nn NOT NULL,
    status       VARCHAR2(10) DEFAULT 'DOCK',
    CONSTRAINT cruises_pk PRIMARY KEY (cruise_id),
    CONSTRAINT cruises_status_ck CHECK (status IN ('DOCK','SAILING','MAINT'))
);

-- Composite PK (faqat Out-of-line)
CREATE TABLE hd_tickets (
    hd_year      NUMBER(4),
    hd_ticket_no NUMBER,
    CONSTRAINT hd_pk PRIMARY KEY (hd_year, hd_ticket_no)
);

-- FOREIGN KEY (ota ustunda PK yoki UNIQUE bo'lishi shart)
CREATE TABLE ships (
    ship_id      NUMBER PRIMARY KEY,
    home_port_id NUMBER,
    CONSTRAINT ships_port_fk FOREIGN KEY (home_port_id) REFERENCES ports(port_id)
);

-- Ustunni o'chirish / yashirish
ALTER TABLE cruises DROP COLUMN captain_id CASCADE CONSTRAINTS;
ALTER TABLE cruises DROP (start_date, end_date);
ALTER TABLE cruises SET UNUSED (status);
ALTER TABLE cruises DROP UNUSED COLUMNS;
```

---

## Imtihon xulosasi (Exam Watch)

1. **`DESC`/`DESCRIBE` SQL buyrug'i EMAS** — SQL*Plus/SQL Developer vositasi buyrug'i.
2. **DDL → implicit commit**, `ROLLBACK` yo'q.
3. **`NOT NULL` faqat In-line**; composite PK/UNIQUE/FK **faqat Out-of-line**.
4. Bir sxemada **TABLE va VIEW** bir nomda bo'lmaydi; **TABLE va INDEX/CONSTRAINT** bir nomda bo'lishi mumkin.
5. Bir bazada **USER va ROLE** bir nomda bo'lmaydi.
6. **ORA-01438**: butun qism xonalari `p−s` dan oshsa. Manfiy scale yaxlitlaydi, xato bermaydi.
7. **External Table** READ-ONLY; DML, INDEX, CONSTRAINT taqiqlanadi.
8. **LOB** PK/UNIQUE/DISTINCT/GROUP BY/ORDER BY/JOIN da ishlatilmaydi.
9. `SET UNUSED` qilingan ustunni **tiklab bo'lmaydi**; jadvalda kamida 1 aktiv ustun qolishi shart.
10. `VARCHAR2` da `n` majburiy; `CHAR` da ixtiyoriy (default 1).
