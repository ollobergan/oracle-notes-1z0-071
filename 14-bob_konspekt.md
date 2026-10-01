# 14-BOB: CONTROLLING USER ACCESS (FOYDALANUVCHILAR KIRISH HUQUQINI BOSHQARISH)

Bu bob 1Z0-071 imtihonining uchta obyektini qamrab oladi:
1. **System va Object huquqlarini farqlash** (system privileges vs object privileges).
2. **Jadvallar va foydalanuvchilarga huquq berish/qaytarib olish** (GRANT / REVOKE).
3. **Huquqlar (privileges) va Rollar (roles)ni farqlash.**

Asosiy g'oya: har qanday amal uchun foydalanuvchida tegishli **huquq (privilege)** bo'lishi shart. Huquq ikki yo'l bilan keladi — **to'g'ridan-to'g'ri** (direct grant) yoki **rol orqali** (role).

---

# 1-BO'LIM: SYSTEM VS OBJECT HUQUQLARI

## 1.1. System huquqlari (System Privileges)
Baza miqyosidagi amalni yoki DDL operatsiyasini bajarish huquqi (ulanish, obyekt yaratish, foydalanuvchi yaratish).

- **Minimal ulanish:** bazaga kirish uchun kamida **`CREATE SESSION`** kerak. Aks holda `ORA-01045: user lacks CREATE SESSION privilege; logon denied`.
- **`CREATE TABLE`** — o'z sxemasida jadval yaratish, uni `ALTER`/`DROP` qilish va unda indeks yaratish huquqini beradi (alohida `CREATE INDEX` system huquqi **o'z** jadvallaringiz uchun shart emas).
- **`ANY` kalit so'zi:** nomida `ANY` bo'lsa (`CREATE ANY TABLE`, `SELECT ANY TABLE`, `DROP ANY TABLE`), amal **istalgan foydalanuvchi sxemasida** bajarilishi mumkin.
- **`UNLIMITED TABLESPACE`** — jadvalga ma'lumot yozish uchun kvota beradi. **Faqat foydalanuvchiga** berish mumkin, **rolga berib bo'lmaydi.**

**Sintaksis** (`ON` yo'q):
```sql
GRANT CREATE SESSION, CREATE TABLE TO lisa;
```

## 1.2. Object huquqlari (Object Privileges)
Aniq bir mavjud obyekt (jadval, view, sequence, procedure) ustidagi amal huquqi.

- **Egalik (Ownership):** obyektni yaratgan foydalanuvchi uning barcha object huquqlariga avtomatik ega.
- **Jadval huquqlari:** `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `ALTER`, `INDEX`, `REFERENCES`, `READ`.
- **Procedure/function/package huquqi:** `EXECUTE`.
- **Sequence huquqlari:** faqat **2 ta** object huquqi bor — **`SELECT`** (`NEXTVAL`/`CURRVAL` olish uchun) va **`ALTER`** (`INCREMENT`/`MAXVALUE` kabilarni o'zgartirish). `INSERT`/`UPDATE`/`DELETE` sequence'ga tegishli emas.
  ```sql
  GRANT SELECT ON order_seq TO henry;   -- henry endi order_seq.NEXTVAL ishlata oladi
  ```
- **`MERGE`** uchun alohida huquq yo'q — nishon bo'yicha `INSERT`/`UPDATE`/`DELETE` va manba bo'yicha `SELECT` huquqlari kerak.

**Sintaksis** (`ON object` majburiy):
```sql
GRANT SELECT, UPDATE ON webinars TO henry;
```

### Ustun darajasidagi huquqlar (Column-level) — imtihon nuqtasi
`INSERT`, `UPDATE`, `REFERENCES` huquqlarini **aniq ustunlarga** berish mumkin:
```sql
GRANT UPDATE (salary) ON employees TO henry;
GRANT INSERT (employee_id, last_name) ON employees TO henry;
```
- **`SELECT` va `DELETE` ustun darajasida berib bo'lmaydi** (ular butun qatorga ishlaydi). Ustunlarni `SELECT`dan yashirish uchun **view** ishlatiladi.

> **⚠️ REVOKE ustun darajasida ISHLAMAYDI (EXAM):** huquqni ustun bo'yicha `GRANT` qilsa bo'ladi, lekin ustun bo'yicha **`REVOKE` qilib bo'lmaydi**. `REVOKE` har doim **butun** object huquqini (barcha ustunlar bilan) olib tashlaydi:
> ```sql
> GRANT UPDATE (salary, commission_pct) ON employees TO henry;
>
> -- Faqat bitta ustunni olib tashlashga urinish → XATO:
> REVOKE UPDATE (salary) ON employees FROM henry;   -- sintaksis xatosi
>
> -- To'g'ri yo'l: butun UPDATE ni revoke qilib, keraklisini qayta grant qilish:
> REVOKE UPDATE ON employees FROM henry;            -- HAMMA ustun olib tashlanadi
> GRANT  UPDATE (commission_pct) ON employees TO henry;  -- kerakligini qayta berish
> ```

### READ vs SELECT (Oracle 12c+/19c) — imtihon nuqtasi
- **`SELECT`** — o'qish + qatorlarni qulflash: `SELECT ... FOR UPDATE` va `LOCK TABLE ... IN EXCLUSIVE MODE` ishlaydi.
- **`READ`** — faqat o'qish. `SELECT ... FOR UPDATE` yoki `LOCK TABLE` urinilsa **`ORA-01031: insufficient privileges`**.
- Shu bois faqat-o'qish foydalanuvchilari uchun `READ` xavfsizroq (tasodifiy qulflashning oldini oladi). System varianti: `READ ANY TABLE`.

## 1.3. Guruhlab berish va PUBLIC
- **`GRANT ALL PRIVILEGES TO user;`** — barcha **system** huquqlari (`PRIVILEGES` so'zi shart).
- **`GRANT ALL ON table TO user;`** — obyektning barcha **object** huquqlari (`PRIVILEGES` ixtiyoriy).
- **`PUBLIC`** — barcha mavjud va kelajakdagi foydalanuvchilarni bildiruvchi virtual guruh. `GRANT CREATE SESSION TO PUBLIC;` → hamma ulana oladi. Huquqlarni tekshirishda `GRANTEE = 'PUBLIC'` qatorlarini ham hisobga olish kerak.

---

# 2-BO'LIM: GRANT / REVOKE

## 2.1. Uzatish opsiyalari — ikkisi ikki xil
| | System huquqi / Rol | Object huquqi |
| :--- | :--- | :--- |
| Uzatish opsiyasi | **`WITH ADMIN OPTION`** | **`WITH GRANT OPTION`** |

- **`WITH ADMIN OPTION`** (system huquqi yoki rol): qabul qiluvchi shu huquqni/rolni boshqalarga ham bera oladi.
- **`WITH GRANT OPTION`** (object huquqi): qabul qiluvchi shu object huquqini boshqalarga bera oladi.
- **Diqqat (tuzoq):** rolni berishda **`WITH ADMIN OPTION`** ishlatiladi, `WITH GRANT OPTION` emas.
- **Diqqat (tuzoq):** object huquqini **rolga** `WITH GRANT OPTION` bilan **berib bo'lmaydi** — bu cheklov imtihonda tez-tez tekshiriladi.
- `WITH GRANT OPTION` bilan berilgan huquqni egasi `REVOKE` qilsa, opsiya ham yo'qoladi.

## 2.2. REVOKE kaskadliligi — eng muhim farq

**System huquqi — KASKADSIZ (non-cascading):**
A → B ga `WITH ADMIN OPTION` bilan `CREATE TABLE` bersa, B → C ga bersa, keyin A → B dan `REVOKE` qilsa — **C huquqni saqlab qoladi.**

**Object huquqi — KASKADLI (cascading):**
A → B ga `WITH GRANT OPTION` bilan `SELECT` bersa, B → C ga bersa, keyin A → B dan `REVOKE` qilsa — **C ham huquqni yo'qotadi** (zanjir bo'ylab tarqaladi).

## 2.3. REVOKE ... CASCADE CONSTRAINTS — imtihon nuqtasi
Agar foydalanuvchi `REFERENCES` huquqi bilan FK (foreign key) constraint yaratgan bo'lsa, oddiy `REVOKE REFERENCES` xato beradi. Bog'liq constraint'larni ham o'chirish uchun:
```sql
REVOKE REFERENCES ON employees FROM henry CASCADE CONSTRAINTS;
```

## 2.4. Boshqa sxemaga murojaat va synonymlar
- Boshqa sxema jadvaliga murojaat qilganda **sxema prefiksi** shart: `SELECT * FROM lisa.webinars;`
- Prefikssiz ishlatish uchun **synonym** yaratiladi. `CREATE PUBLIC SYNONYM webinars FOR lisa.webinars;` nomni barchaga ko'rsatadi, **lekin asosiy jadvalda object huquqi baribir kerak** — synonym huquq bermaydi.

## 2.5. View orqali kirish
User A `EMPLOYEES` egasi bo'lib, uning ustiga `EMP_VIEW` yaratib, B ga faqat view bo'yicha `SELECT` bersa — **B asosiy jadvalga huquqsiz ham view orqali o'qiy oladi.** View — ustunlar/qatorlarni cheklab ko'rsatishning asosiy vositasi.

## 2.6. DROP qilingan obyekt va FLASHBACK
- Jadval `DROP` qilinsa, unga berilgan **barcha object huquqlari lug'atdan o'chadi**. Jadval qayta yaratilsa, huquqlarni **qaytadan berish kerak**.
- **Istisno:** `FLASHBACK TABLE ... TO BEFORE DROP` bilan savatchadan (Recycle Bin) tiklansa — **avvalgi object huquqlari va indekslar birga qaytadi.**

---

# 3-BO'LIM: ROLLAR (ROLES)

## 3.1. Rol nima
Bir yoki bir nechta system/object huquqlari va boshqa rollarni jamlovchi obyekt. Boshqaruvni soddalashtiradi: har foydalanuvchiga 20 ta huquqni alohida berish o'rniga, huquqlar rolga yuklanadi, rol foydalanuvchiga beriladi.

- Rol **nonschema** obyekt — hech bir sxemaga tegishli emas, baza darajasidagi nom fazosida yashaydi.
- Yaratish uchun **`CREATE ROLE`** huquqi kerak.
- Rol va foydalanuvchi **bir xil nom fazosini** baham ko'radi (bir xil nomli user va rol bo'lmaydi), lekin rol va jadval bir xil nomli bo'lishi mumkin.

```sql
CREATE ROLE cruise_analyst;
GRANT SELECT ON ships TO cruise_analyst;      -- rolga huquq yuklash
GRANT cruise_analyst TO henry WITH ADMIN OPTION;  -- rolni berish
```
- Rol tarkibidagi huquq o'zgarsa, o'zgarish shu rolga ega **barcha foydalanuvchilarga darhol** ta'sir qiladi.

## 3.2. Rol vs To'g'ridan-to'g'ri huquq — mustaqillik
Direct grant va role-based grant **bir-biridan mustaqil**:
- A da `SELECT ON t1` to'g'ridan-to'g'ri bor + `R1` roli ham `SELECT ON t1` beradi. `REVOKE R1 FROM a` qilinsa — A direct huquq evaziga **o'qishda davom etadi**.
- Teskarisi: direct huquq olib tashlansa, lekin `R1` qolsa — A rol orqali baribir o'qiydi.

## 3.3. Rol cheklovi — View/PL/SQL yaratishda (imtihon tuzog'i)
**View yoki stored procedure/function yaratishda kerakli object huquq to'g'ridan-to'g'ri (direct) berilgan bo'lishi shart.** Rol orqali kelgan huquqlar bunday obyektlarni yaratishda **hisobga olinmaydi**.

## 3.4. Oldindan belgilangan rollar (19c)
| Rol | Tarkibi |
| :--- | :--- |
| **`CONNECT`** | Zamonaviy Oracle'da (10gR2+/19c) faqat **`CREATE SESSION`**. |
| **`RESOURCE`** | `CREATE TABLE`, `CREATE SEQUENCE`, `CREATE PROCEDURE`, `CREATE TRIGGER`, `CREATE TYPE`, `CREATE CLUSTER`, `CREATE INDEXTYPE`, `CREATE OPERATOR`. |
| **`DBA`** | 100 dan ortiq system huquq — bazani to'liq boshqarish. |

- **Nozik nuqta:** `RESOURCE` berilganda foydalanuvchi amalda **`UNLIMITED TABLESPACE`** ham oladi, lekin bu `DBA_SYS_PRIVS`da rol tarkibida **ko'rinmaydi** (foydalanuvchiga alohida direct huquq sifatida biriktiriladi).
- Oracle bu klassik rollardan **foydalanishni tavsiya etmaydi** (kelajakda o'zgarishi mumkin), lekin imtihonda uchraydi.

---

# 4-BO'LIM: FOYDALANUVCHINI BOSHQARISH (ASOSIY)
```sql
CREATE USER lisa IDENTIFIED BY poe;     -- yangi foydalanuvchi (hali ulana olmaydi)
GRANT CREATE SESSION TO lisa;           -- ulanish huquqi
ALTER USER lisa IDENTIFIED BY newpass;  -- parolni o'zgartirish
DROP USER lisa;                         -- o'chirish (sxemasida obyekt bo'lsa — CASCADE kerak)
DROP USER lisa CASCADE;                 -- foydalanuvchi + uning barcha obyektlari bilan
```
- Yangi yaratilgan foydalanuvchi `CREATE SESSION` berilmaguncha ulana olmaydi.

---

# 5-BO'LIM: IMPLICIT COMMIT (MUHIM)
`GRANT` va `REVOKE` — bu **DDL** buyruqlari. Bajarilishi bilan undan oldingi DML tranzaksiyalarini **avtomatik COMMIT** qiladi. Shu sababli `GRANT`dan oldingi `INSERT`/`UPDATE`ni keyin `ROLLBACK` qilib bo'lmaydi.

---

# 6-BO'LIM: DATA DICTIONARY VIEWS (HUQUQLARNI TEKSHIRISH)
| View | Nimani ko'rsatadi |
| :--- | :--- |
| `USER_SYS_PRIVS` | Joriy foydalanuvchiga berilgan **system** huquqlari |
| `DBA_SYS_PRIVS` | Barcha foydalanuvchi/rollarning system huquqlari |
| `SESSION_PRIVS` | Joriy seansda **faol** system huquqlari |
| `USER_TAB_PRIVS` | Joriy foydalanuvchi ega/beruvchi/qabul qiluvchi bo'lgan object huquqlari |
| `USER_COL_PRIVS` | **Ustun darajasidagi** object huquqlar |
| `ALL_TAB_PRIVS_RECD` | Foydalanuvchiga, `PUBLIC`'ga yoki faol rolga berilgan object huquqlari |
| `DBA_ROLES` | Bazadagi barcha rollar |
| `DBA_ROLE_PRIVS` | Foydalanuvchi/rollarga berilgan rollar |
| `ROLE_SYS_PRIVS` | Rolga biriktirilgan system huquqlari |
| `ROLE_TAB_PRIVS` | Rolga biriktirilgan object huquqlari |
| `SESSION_ROLES` | Joriy seansda faol rollar |

---

# 7-BO'LIM: TAQQOSLASH JADVALLARI

### System vs Object huquqlari
| Xususiyat | System Privilege | Object Privilege |
| :--- | :--- | :--- |
| Qamrov | Butun baza (CREATE TABLE, CREATE SESSION) | Aniq obyekt (SELECT ON t1) |
| Sintaksis | `GRANT priv TO user;` | `GRANT priv ON obj TO user;` |
| Uzatish opsiyasi | `WITH ADMIN OPTION` | `WITH GRANT OPTION` |
| REVOKE kaskadliligi | **Kaskadsiz** | **Kaskadli** |
| Ustun darajasida | — | GRANT: `INSERT/UPDATE/REFERENCES` (ha), `SELECT/DELETE` (yo'q); REVOKE esa doim **butun huquq** (ustun bo'yicha emas) |
| `ALL` sintaksisi | `GRANT ALL PRIVILEGES TO user;` | `GRANT ALL ON obj TO user;` |

### Direct vs Role-based huquq
| Xususiyat | Direct (to'g'ridan-to'g'ri) | Role orqali |
| :--- | :--- | :--- |
| Boshqaruv | Har user uchun alohida GRANT/REVOKE | Rol o'zgarsa — barcha userlarda birdan |
| View/PL/SQL yaratish | **O'tadi** (ishlaydi) | **O'tmaydi** (hisobga olinmaydi) |
| Rolni uzatish | — | Faqat `WITH ADMIN OPTION` |
| Object `WITH GRANT OPTION` | Mumkin | **Mumkin emas** (rolga berilmaydi) |

---

# 8-BO'LIM: ORA XATOLARI — TEZKOR MA'LUMOTNOMA
| Xato | Sabab |
| :--- | :--- |
| `ORA-01045` | Foydalanuvchida `CREATE SESSION` yo'q — ulanish rad etildi |
| `ORA-01031` | `insufficient privileges` — huquq yetishmaydi (mas., `READ`-only user `FOR UPDATE` urindi) |
| `ORA-01927` | O'zingiz bermagan huquqni REVOKE qilishga urinish |

---

*Konspekt Steve O'Hearn darsligining 14-bobi ("Controlling User Access") asosida, Oracle 19c xususiyatlari (READ huquqi, zamonaviy CONNECT/RESOURCE rollari) bilan to'ldirilgan holda tayyorlandi.*
