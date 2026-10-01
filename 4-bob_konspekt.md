# Oracle 1Z0-071 — 4-Bob Konspekti
## Ma'lumotlarni Cheklash va Saralash (Restricting and Sorting Data)

Konspekt faqat imtihonda tekshiriladigan bilimlarga qaratilgan: qoidalar, istisno holatlar (edge cases) va sintaktik tuzoqlar.

---

## 1-Qism: WHERE Klauzulasi (Ma'lumotlarni Cheklash)

`WHERE` qatorlarni mantiqiy shart (Boolean) asosida filtrlaydi. Natijaga faqat sharti **TRUE** bo'lgan qatorlar tushadi; **FALSE** va **UNKNOWN (NULL)** qatorlar tushmaydi.

**Sintaktik tartib:** `WHERE` har doim `FROM` dan keyin, `GROUP BY` / `ORDER BY` dan oldin keladi.

### 1.1. Taqqoslash operatorlari
*   `=`, `>`, `<`, `>=`, `<=`
*   Teng emas — uchta yozuv bir xil ishlaydi: `!=`, `<>`, `^=`
*   `BETWEEN a AND b` — a va b oralig'i, **chegaralar kiritilgan (inclusive)**; `>= a AND <= b` ga teng.
*   `IN (v1, v2, ...)` — ro'yxatdagi qiymatlardan biriga mos kelish (`= v1 OR = v2 OR ...` ga teng).
*   `LIKE` — shablonli qidiruv (3-qismga qarang).
*   `IS NULL` / `IS NOT NULL` — bo'sh qiymatni tekshirish.

> **BETWEEN tuzog'i (muhim!):** kichik qiymat birinchi, katta qiymat keyin yozilishi shart.
> `WHERE salary BETWEEN 2000 AND 1000` — **0 ta qator** qaytaradi (xatolik emas), chunki u `>= 2000 AND <= 1000` ga aylanadi. To'g'risi: `BETWEEN 1000 AND 2000`.
> Bu qoida sonlar, matnlar va sanalar uchun ham amal qiladi.

### 1.2. Ifodalar har ikki tomonda bo'lishi mumkin
Operatorning chap tomonida ham, o'ng tomonida ham ustun bo'lishi shart emas — har qanday SQL ifodasi bo'la oladi.
```sql
WHERE ((2 * lifeboats) + 57) - capacity IN (lifeboats * 20, lifeboat_count + length);
```

### 1.3. Mantiqiy operatorlar va ustunlik tartibi (Operator Precedence)
Bir so'rovda bir nechta operator bo'lsa, Oracle quyidagi **to'liq tartibda** baholaydi (yuqoridan pastga):

1.  Arifmetik amallar: `*`, `/`, keyin `+`, `-`
2.  Konkatenatsiya: `||`
3.  Taqqoslash: `=`, `!=`, `<`, `>`, `<=`, `>=`
4.  `IS [NOT] NULL`, `LIKE`, `[NOT] IN`
5.  `[NOT] BETWEEN`
6.  `NOT` (mantiqiy inkor)
7.  `AND`
8.  `OR`

**Qavslar `()` har doim tartibni ustun qilib o'zgartiradi.**

> **AND/OR tuzog'i:** `AND` har doim `OR` dan oldin bajariladi.
> ```sql
> WHERE style = 'Suite' OR style = 'Stateroom' AND window = 'Ocean';
> ```
> Bu `style = 'Suite'` **YOKI** `(style = 'Stateroom' AND window = 'Ocean')` demakdir — ya'ni barcha Suite xonalari derazasidan qat'i nazar qaytadi. Kutilgan natija uchun qavs shart:
> ```sql
> WHERE (style = 'Suite' OR style = 'Stateroom') AND window = 'Ocean';
> ```

### 1.4. Turli ma'lumot turlarini taqqoslash
*   **Sonlar:** matematik mantiq bo'yicha (manfiy < 0 < musbat).
*   **Matnlar:** belgilar ASCII/Unicode qiymati bo'yicha chapdan o'ngga.
    *   Katta harf kichik harfdan kichik: `'A'`=65, `'a'`=97 → `'Michael' < 'michael'`.
    *   **Matndagi sonlar tuzog'i:** `VARCHAR2` da saqlangan sonlar belgi sifatida solishtiriladi → `'3' > '22'` (birinchi belgi `'3' > '2'`).
*   **Sanalar:** oldingi sana kichikroq: `'01-JAN-2026' < '31-DEC-2026'`.

### 1.5. NULL bilan ishlash mantiqi (juda muhim!)
NULL — bu 0 ham, bo'shliq (space) ham emas; u **noma'lum (unknown)** qiymat.

| Holat | Natija |
| :--- | :--- |
| Arifmetika: `10 + NULL` | **NULL** |
| Taqqoslash: `salary = NULL`, `salary != NULL` | **UNKNOWN** (hech qachon qator qaytmaydi) |
| To'g'ri tekshirish | faqat `IS NULL` / `IS NOT NULL` |
| `COUNT(col)`, `AVG`, `SUM` | NULL qatorlarni **tashlab ketadi** |
| `COUNT(*)` | NULL qatorlarni ham **sanaydi** |

> **NOT IN + NULL tuzog'i (imtihonda tez-tez tushadi):**
> ```sql
> WHERE manager_id NOT IN (100, 101, NULL);   -- 0 ta qator!
> ```
> Chunki u `!= 100 AND != 101 AND != NULL` ga aylanadi; `!= NULL` har doim UNKNOWN, `AND` zanjiri esa butunlay buziladi → **no rows selected**.
> Yechim: ro'yxat yoki subquery ichida `IS NOT NULL` bilan NULL larni chiqarib tashlash.

> **IN + NULL esa xavfsiz:** `WHERE department_id IN (10, 20, NULL)` normal ishlaydi, chunki u `= 10 OR = 20 OR = NULL` ga aylanadi — `OR` zanjirida bitta TRUE yetarli. NULL shart shunchaki hech qachon mos kelmaydi, xolos. `NOT IN` dan farqi shunda.

---

## 2-Qism: LIKE va Wildcard Qidiruv

*   `_` (pastki chiziq) — **aynan bitta** ixtiyoriy belgi.
*   `%` (foiz) — **nol, bitta yoki ko'p** belgi.
*   LIKE **registrga sezgir** (case-sensitive): `'a%'` va `'A%'` turlicha natija beradi.

**Wildcardlarni birlashtirish:** `LIKE 'Codd_%'` → `'Codd'` dan keyin kamida bitta belgi (`_`), undan keyin ixtiyoriy belgilar (`%`). Shuning uchun `'Codd'` so'zining o'zi bu shablondan **o'tmaydi** (kamida 5 belgi kerak).

**ESCAPE** — matndagi haqiqiy `_` yoki `%` ni qidirish uchun:
```sql
WHERE product_code LIKE 'AB\_%' ESCAPE '\';   -- 'AB_' bilan boshlanadiganlar
```

> **LIKE NULL tuzog'i:** `WHERE manager_id LIKE NULL` — xatoliksiz bajariladi, lekin **0 ta qator** (UNKNOWN). To'g'risi: `IS NULL`.

---

## 3-Qism: ORDER BY Klauzulasi (Saralash)

`ORDER BY` qaytgan qatorlarni tartiblaydi va **har doim so'rovning eng oxirgi klauzulasi** bo'ladi (Set operatorlar bo'lsa ham umumiy so'rov oxirida).

*   **ASC** — o'sish (standart, yozish majburiy emas).
*   **DESC** — kamayish.
*   `ASC`/`DESC` har bir ustunga alohida qo'llanadi; bitta qavs bilan hammaga birdan qo'llab bo'lmaydi:
    ```sql
    ORDER BY ship_id ASC, project_cost DESC;
    ```

### 3.1. Nimaga ko'ra saralash mumkin
*   **Ustun nomi** — SELECT ro'yxatida bo'lmagan ustun bo'yicha ham mumkin.
*   **Ustun taxallusi (alias)** — SELECT da berilgan alias `ORDER BY` da ishlaydi (chunki ORDER BY eng oxirida bajariladi).
*   **Ifoda** — `ORDER BY (salary * 12) + NVL(bonus, 0)`.
*   **Pozitsiya (1-indexed)** — SELECT ro'yxatidagi ustun tartib raqami bo'yicha:
    ```sql
    SELECT ship_name, capacity, length FROM ships ORDER BY 2 DESC, 1 ASC;
    ```
    Pozitsiya, nom va ifodalarni **aralashtirib** ishlatish mumkin. Pozitsiya soni SELECT dagi ustunlar sonidan katta bo'lsa — xatolik.

> **Muhim farq:** ustun taxallusi (alias) `ORDER BY` da ishlaydi, lekin `WHERE` (va `GROUP BY`) da **ishlamaydi** — chunki WHERE SELECT dan oldin bajariladi va aliasni hali bilmaydi.

### 3.2. ORDER BY va NULL
Oracle saralashda NULL ni **eng katta qiymat** deb oladi:
*   **ASC** → NULL lar **oxirida** (standart `NULLS LAST`).
*   **DESC** → NULL lar **boshida** (standart `NULLS FIRST`).

Majburiy boshqarish:
```sql
ORDER BY salary ASC NULLS FIRST;   -- o'sish bo'lsa ham NULLlar birinchi
ORDER BY salary DESC NULLS LAST;   -- kamayish bo'lsa ham NULLlar oxirida
```

### 3.3. Cheklovlar
*   `LIKE` ni `ORDER BY` ichida ishlatib bo'lmaydi (sintaktik xato).
*   `CLOB`, `BLOB`, `NCLOB` (LOB) turlari bo'yicha saralab bo'lmaydi.

---

## 4-Qism: Row Limiting Clause (12c+)

Top-N va pagination uchun. Sintaksis:
```sql
[OFFSET offset {ROW | ROWS}]
[FETCH {FIRST | NEXT} [rowcount | percent PERCENT] {ROW | ROWS} {ONLY | WITH TIES}]
```

*   **OFFSET** — nechanchi qatordan keyin boshlash (tashlab yuboriladigan qatorlar soni; standart 0).
*   **FIRST / NEXT** — sinonim, farqi yo'q.
*   **ROW / ROWS** — sinonim, farqi yo'q.
*   **PERCENT** — aniq son o'rniga qatorlarning foizini oladi.
*   **ONLY** — faqat so'ralgan miqdordagi qatorlar.
*   **WITH TIES** — oxirgi qatorning saralash qiymatiga teng barcha qatorlarni ham qo'shadi.
    *   **WITH TIES uchun `ORDER BY` shart.** Bo'lmasa, teng qiymatni aniqlab bo'lmaydi va u amalda `ONLY` kabi ishlaydi (xatolik bermaydi).

### 4.1. Qiymat anomaliyalari (imtihon uchun kritik)
`OFFSET`, `rowcount` va `percent` uchun **bir xil qoida**:

| Qiymat | Natija |
| :--- | :--- |
| **Manfiy** | `0` deb olinadi (xato emas) |
| **Kasr (masalan 5.5)** | kasr qismi tashlanadi (truncate → 5) |
| **NULL** | **0 ta qator qaytadi** (xato emas, lekin natija bo'sh) |

> **Diqqat (tuzatilgan):** NULL `OFFSET` **0 deb olinmaydi** — u *hech qanday qator qaytarmaydi*. Faqat **manfiy** qiymat 0 ga aylanadi. `OFFSET -5` → 0 (boshidan boshlaydi), lekin `OFFSET NULL` → bo'sh natija.

### 4.2. Tipik misollar
```sql
-- Eng katta maosh bo'yicha 6-10 o'rin (pagination):
SELECT employee_id, salary FROM employees
ORDER BY salary DESC
OFFSET 5 ROWS FETCH NEXT 5 ROWS ONLY;
```

---

## 5-Qism: Ampersand Substitution (SQL*Plus)

Bu SQL standarti emas — **SQL*Plus interfeysining interaktiv imkoniyati**, lekin imtihonda tushadi.

### 5.1. `&` va `&&`
*   `&var` — har safar o'zgaruvchi uchraganda foydalanuvchidan qiymat so'raydi (sessiyada **saqlanmaydi**).
*   `&&var` — faqat **birinchi marta** so'raydi, qiymatni sessiya xotirasida saqlaydi, keyin avtomatik qo'yadi.

**Qo'llash doirasi:** o'zgaruvchi nafaqat qiymat, balki ustun nomi, jadval nomi, butun SELECT ro'yxati yoki klauzula o'rnida ham ishlatiladi:
```sql
SELECT &column_list FROM &table_name WHERE &condition;
```
Matn/sana o'rnida ishlatilsa, tirnoq kerak: `WHERE name = '&username'`.

### 5.2. O'zgaruvchi buyruqlari
*   **DEFINE** — faol o'zgaruvchilarni ko'rsatadi; yangi o'zgaruvchi yaratadi (`DEFINE v_room = 104;`). DEFINE da `&` yozilmaydi, so'rovda chaqirganda `&` kerak.
*   **UNDEFINE** — o'zgaruvchini xotiradan o'chiradi.
*   **ACCEPT ... PROMPT** — maxsus xabar bilan qiymat so'raydi:
    ```sql
    ACCEPT emp_id NUMBER PROMPT 'Xodim ID raqamini kiriting: ';
    ```

### 5.3. SET buyruqlari
*   **SET DEFINE ON/OFF** — substitution tizimini yoqadi/o'chiradi. OFF bo'lsa `&` oddiy matn belgisi deb qaraladi.
*   **SET DEFINE '<belgi>'** — prefiks belgisini o'zgartiradi (faqat **bitta** belgi), masalan `SET DEFINE '*';`.
*   **SET VERIFY ON/OFF** — qiymat qo'yilishidan oldingi (`old`) va keyingi (`new`) so'rov satrini ko'rsatishni boshqaradi.

---

## 6-Qism: Imtihon Tuzoqlari — Tezkor Takrorlash

*   `BETWEEN katta AND kichik` → **0 qator** (tartib: kichik AND katta).
*   `= NULL` / `!= NULL` / `LIKE NULL` → **0 qator**; faqat `IS [NOT] NULL`.
*   `NOT IN (..., NULL)` → **0 qator** (AND zanjiri buziladi); `IN (..., NULL)` esa normal ishlaydi.
*   `AND` `OR` dan oldin bajariladi — qavsga e'tibor.
*   Alias `ORDER BY` da ishlaydi, `WHERE` da **ishlamaydi**.
*   ORDER BY: ASC→NULL oxirida, DESC→NULL boshida.
*   LIKE registrga sezgir.
*   Row limiting: **manfiy → 0**, **kasr → truncate**, **NULL → bo'sh natija** (NULL ≠ 0!).
*   `WITH TIES` uchun `ORDER BY` shart.
*   `COUNT(*)` NULL ni sanaydi, `COUNT(col)`/`AVG`/`SUM` sanamaydi.
