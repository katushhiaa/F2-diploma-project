# Специфікація вимог (SRS)

**Версія:** 2.0.0

**Дата:** **2026**-11-16

**Статус:** Audit-ready RC

**Нормативна база та стандарти:** **ISO**/**IEC**/**IEEE** **29148**:**2018**, **GDPR** (Регламент ЄС **2016**/**679**, зокрема ст. 5(1)(c), 6, 7, 12, 15, 17, 20, 25, 28, 30, 33, 34), Закон України «Про захист персональних даних», **OWASP** **ASVS** v4.0.3 (Level 2), **NIST** SP **800**-**63B**-4, **RFC** **6238**, **RFC** **6455**, **RFC** **7519**, **RFC** **8725**, **RFC** **9110**, **RFC** **9457**, **ISO** **2108**, **ISO**/**IEC** **15420**, **ISO**/**IEC** **25010**, **WCAG** 2.2 (Level AA), **IETF** Idempotency-Key draft.

---

## 0. Document Control & History

| Версія | Дата | Автор / Архітектор | Опис змін |
| --- | --- | --- | --- |
| 1.0.0 | 2026-10-20 | Команда розробки | Початковий проєкт на основі постановки задачі. |
| 1.5.0 | 2026-11-16 | Системний архітектор | Проміжна редакція з базовим покриттям NFR. |
| 1.6.0-RC | 2026-11-16 | Головний системний архітектор | Фінальне усунення S1/S2 дефектів, виправлення логіки fail-closed фільтрації спойлерів, декомпозиція Gap Analysis на вимірювані вимоги. |
| 2.0.0 | 2026-11-16 | Lead Requirements Engineer / Principal Systems Analyst | Виправлено blocker/critical/major дефекти аудиту: суперечності spoiler policy, дублювання SRS-FUN-02.4, невалідний MoSCoW, розділення складених вимог, уточнення JWT/CSRF/MFA, idempotency, retry/DLQ, external API status matrix, DSR, audit log, WS security, queue backpressure, OpenAPI та верифікаційні інструменти. |

---

## 1.  System Context

### 1.1 Purpose & Intended Audience

Цей документ визначає вичерпний набір формалізованих, атомарних і верифікованих вимог до модульної вебплатформи соціального читання. Документ призначений для backend та frontend розробників, DevOps-інженерів, QA-фахівців, архітекторів системи та аудиторів вимог.

### 1.2 Scope (In-Scope / Out-of-Scope)

**In-Scope:** Реєстрація, верифікація email, управління обліковими записами (**JWT**, **MFA**/**TOTP**, refresh, ротація, **CSRF**), ведення цифрової бібліотеки та відгуків, клієнтська детекція штрихкодів **ISBN** із fallback на ручне введення, кадрування **ROI** на клієнті, асинхронний **OCR**-контур з ідемпотентністю, retry, **DLQ** і захистом від переповнення, нечітке зіставлення та знеособлена агрегація цитат (`k ≥ 5`), синхронізація читацьких кімнат у реальному часі з replay за `lastEventId`, зв’язки дружби, єдина серверна fail-closed фільтрація спойлерів, **GDPR**/**DSR**, **OWASP** **ASVS** L2, OpenAPI 3.1 contract.

**Out-of-Scope:** Інтеграція із зовнішніми провайдерами автентифікації (OAuth2/**OIDC**, Google/GitHub login) — перенесено в наступні мажорні релізи; нативні мобільні клієнти (iOS/Android); сторонні розсилки — поштовий канал використовується виключно для транзакційних потреб (підтвердження email, відновлення пароля) через зовнішній **SMTP**-провайдер.

### 1.3 Product Perspective & Context

Система є модульним вебдодатком: **SPA**, серверний **REST**/WebSocket **API**, ізольований воркер-пул, MongoDB, Redis, BullMQ.

**Контекст взаємодії та зовнішні інтерфейси:**

- ****REST** **API**** (`/api/v1/...`): бізнес-логіка, **JSON**, помилки у форматі **RFC** **9457**, OpenAPI 3.1.
- **WebSocket** (`/ws/...`): Socket.io, sticky sessions, Redis adapter для pub/sub, автентифікація під час handshake, sequence number на кімнату, replay 24 год.
- **Зовнішні каталоги:** Open Library **API** та Google Books **API** через Circuit Breaker, timeout, single-flight, positive-only cache.
- ****SMTP**:** транзакційні листи, токени email verification/reset зберігаються лише у вигляді hash.

### 1.4 User Characteristics

- **Гість (**ACT**-01):** неавтентифікований користувач; доступ до публічного каталогу, публічних профілів, загальної інформації.
- **Читач (**ACT**-02):** автентифікований користувач; управляє власною бібліотекою, цитатами, відгуками, сесіями.
- **Учасник кімнати (**ACT**-03):** Читач із дійсним членством у читацькій кімнаті.
- **Адміністратор кімнати (**ACT**-04):** власник/адміністратор кімнати; генерація токенів запрошення, виключення учасників.
- **Адміністратор сайту (**ACT**-05):** привілейована роль модератора з обов’язковою **MFA**/**TOTP**; модерація, аудит, **DLQ**.
- **Система (**ACT**-06):** фоновий агент обслуговування даних.

### 1.5 Assumptions & Dependencies

- Стабільний інтернет-зв’язок і підтримка сучасних WebAPI (BarcodeDetector або ручний fallback, Canvas **API**).
- Доступність Open Library та Google Books; у разі недоступності — ручне введення метаданих.
- Налаштований зовнішній **SMTP** для транзакційних повідомлень.
- Доступність **HIBP** Pwned Passwords range **API** або локального списку скомпрометованих паролів; за недоступності обох — fail-closed для реєстрації/зміни пароля.

### 1.6 References & Standards

- [**ISO**-**29148**] **ISO**/**IEC**/**IEEE** **29148**:**2018** — Requirements engineering.
- [**ISO**-**2108**] **ISBN**.
- [**ISO**-**15420**] **EAN**/**UPC** bar code symbology.
- [**GDPR**] Regulation (EU) **2016**/**679**.
- [**ZKU**-PD] Закон України «Про захист персональних даних».
- [**OWASP**-**ASVS**] **OWASP** **ASVS** v4.0.3 Level 2.
- [**NIST**-**800**-**63B**] **NIST** SP **800**-**63B**-4.
- [**ISO**-**25010**] **ISO**/**IEC** **25010**:**2011**.
- [**WCAG**-2.2] **WCAG** 2.2 Level AA.
- [**RFC**-**6238**, **6455**, **7519**, **8725**, **9110**, **9457**] **IETF** standards.
- [Idempotency-Draft] **IETF** **HTTP** Idempotency-Key Draft.
- [OpenAPI-3.1] OpenAPI Specification 3.1.0.

### 1.7  Glossary

- ****OCR**:** оптичне розпізнавання тексту.
- ****DLQ**:** Dead-Letter Queue.
- **Token Bucket:** rate limiting algorithm.
- **Circuit Breaker:** fault-tolerance pattern.
- ****SLO** / **SLI**:** Service Level Objective / Indicator.
- ****RTO** / **RPO**:** Recovery Time / Point Objective.
- ****MFA** / **TOTP**:** багатофакторна автентифікація / **RFC** **6238**.
- **k-anonymity:** неможливість ідентифікації суб’єкта серед `k ≥ 5` осіб.
- ****PII**:** Personally Identifiable Information.
- **Spoiler Prevention (Fail-Closed):** серверне маскування за замовчуванням; розкриття лише за явними винятками.
- ****WORM** / Append-only:** захист аудит-логів від модифікації/видалення.
- **Idempotency-Key:** клієнтський ідентифікатор одноразовості операції.
- **Token Family:** група пов’язаних refresh-токенів.
- **jti:** **JWT** ID.
- ****CER**:** Character Error Rate.
- **Sticky Sessions:** закріплення сесії за екземпляром сервера.
- **Кластер уривків:** група семантично схожих цитат.
- ****DSR**:** Data Subject Request.
- ****CSRF**:** Cross-Site Request Forgery.
- ****JWKS**:** **JSON** Web Key Set.

### 1.8 Design Constraints (DC)

- **DC-01 (Оркестрація та хостинг):** single-host Docker Compose; HA кластерного рівня не гарантується; **CPU**/**RAM** ліміти контейнерів.
- **DC-02 (Спостережуваність):** **JSON**-логи, обов’язковий `correlationId`, маскування **PII** (`pino.redact`); Prometheus, Grafana; гістограми 0.1/0.2/0.3/0.5/1/2 с.
- **DC-03 (Ізоляція Redis):** два логічно незалежні екземпляри: cache (`allkeys-lru`, без персистентності) та broker/security (`noeviction`, **AOF** `appendfsync everysec`, `maxmemory`).
- **DC-04 (Тестове середовище):** Vitest, Supertest, k6, Artillery, Stryker, fast-check, testssl.sh, Semgrep, Gitleaks, Trivy, Toxiproxy, Testcontainers, promtool, redis_exporter, mongodb_exporter, **MSW**/nock, Playwright + axe-core, Lighthouse CI, **OWASP** **ZAP** baseline, Schemathesis, Spectral.
- **DC-05 (Технологічний стек):** Node.js **LTS**, Express.js, Socket.io + Redis adapter, BullMQ, MongoDB/Mongoose, Redis, Vue.js 3, Pinia, Vue Router, Tesseract.js, Zod, jsonwebtoken, bcrypt, pino, Docker Compose.
- **DC-06 (**API** contract):** OpenAPI 3.1 є джерелом істини для **REST**; помилки **RFC** **9457**.

---

## 2. Roles, Actors & Access Matrix

### 2.1 Актори системи

**ACT**-01 Гість; **ACT**-02 Читач; **ACT**-03 Учасник кімнати; **ACT**-04 Адміністратор кімнати; **ACT**-05 Адміністратор сайту; **ACT**-06 Система.

### 2.2 Матриця доступу (Роль × Ресурс × Дія)

| Ресурс / Сутність | ACT-01 | ACT-02 | ACT-03 | ACT-04 | ACT-05 | ACT-06 |
| --- | --- | --- | --- | --- | --- | --- |
| Публічний профіль / каталог | Read | Read | Read | Read | Read, Delete | Read |
| Бібліотека / Записи | — | CRUD (свої) | CRUD (свої) | CRUD (свої) | Read, Delete (GDPR) | — |
| Приватні цитати | — | CRUD (свої) | CRUD (свої) | CRUD (свої) | — | — |
| Публічні цитати / відгуки | Read | CRUD (свої) | CRUD (свої) | CRUD (свої) | Read, Hide, Delete | Read |
| Коментарі кімнати | — | — | CRUD (свої) | CRUD (свої, kick) | Read, Hide, Delete | — |
| Читацька кімната | — | Join (запрош.) | Read, WS-sync | CRUD (управління) | Manage all | Read |
| Зв’язок дружби | — | CRUD (свої) | CRUD (свої) | CRUD (свої) | — | — |
| OCR-завдання | — | Create, Read | Create, Read | Create, Read | Read, Restart | Execute |
| Журнал згод / Журнал видалень | — | Read (свої) | Read (свої) | Read (свої) | Read (повний аудит) | Create |
| Сесії / Токени | — | CRUD (свої) | CRUD (свої) | CRUD (свої) | Manage all | Cleanup |
| Запрошення в кімнати | — | Read | CRUD | CRUD | Manage all | Expire |
| Історія прогресу | — | CRUD (свої) | CRUD (свої) | CRUD (свої) | Read | — |
| Кластер / Статистика | Read | Read | Read | Read | Read | Compute |
| DLQ | — | — | — | — | Read, Restart | Manage |
| Журнал аудиту | — | — | — | — | Read | Create |

### 2.3 Політика кодів помилок доступу

- Приватні ресурси / чужі сутності → `**404** Not Found`.
- Конфлікт прав на існуючі публічні/спільні ресурси → `**403** Forbidden`.
- Час відповіді для **404** та **403** вирівнюється програмно для запобігання side-channel атакам.
- Недоступність upstream-провайдера → `**503** Service Unavailable` з `Retry-After`; `**502**` — лише для некоректної відповіді upstream.

---

## 3. Functional Requirements

### 3.1 Модуль автентифікації, облікових записів, профілю та соціальних зв’язків

****SRS**-**FUN**-01.1** (Джерело: **NIST** SP **800**-**63B**-4 | MoSCoW: Must | Ризик: Low)

Система повинна забезпечувати реєстрацію за email і паролем. Пароль: мінімум 12 code points, максимум **128** code points. Система повинна перевіряти пароль за **HIBP** Pwned Passwords range **API** з k-anonymity (передаються лише перші 5 символів **SHA**-1, повний хеш не передається) або за локальним списком скомпрометованих паролів. За недоступності обох джерел реєстрація та зміна пароля повертають `**503** Service Unavailable` (fail-closed).

****SRS**-**FUN**-01.2** (Джерело: **RFC** **7519**, **RFC** **8725**, DC-05 | MoSCoW: Must | Ризик: Medium)

Система повинна надавати `**POST** /api/v1/auth/login` для видачі Access Token у форматі **JWT**: **TTL** 15 хв, алгоритм `**RS256**`, обов’язковий `kid`, claims `sub`, `iss`, `aud`, `exp`, `iat`, `jti`, `role`. Перевірка claims виконується з clock skew ≤30 с. **JWKS** endpoint повинен публікувати публічні ключі; ротація ключів підтримується.

****SRS**-**FUN**-01.3** (Джерело: **NIST** **800**-**63B** | MoSCoW: Must | Ризик: Low)

Система повинна видавати Refresh Token як opaque-рядок ентропії ≥**256** bit, **TTL** 7 діб. Refresh Token зберігається у браузері через cookie з атрибутами `httpOnly`, `Secure`, `SameSite=Strict`, `Path=/api/v1/auth`. На сервері зберігається лише hash refresh-токена.

****SRS**-**FUN**-01.4** (Джерело: аудит ітерації 2 | MoSCoW: Must | Ризик: Medium)

Система повинна підтримувати Token Family Revocation з grace-window 30 с для паралельних вкладок. Повторне використання відкликаного refresh-токена поза grace-window інвалідує всю token family та генерує audit event.

****SRS**-**FUN**-01.5** (Джерело: DC-05 | MoSCoW: Must | Ризик: Low)

Система повинна хешувати паролі точним конвеєром: **UTF**-8 → **SHA**-**256** → base64 → bcrypt із cost ≥12. Усічення пароля не допускається. Pepper не застосовується; секрети bcrypt зберігаються поза репозиторієм.

****SRS**-**FUN**-01.6a** (MoSCoW: Must | Ризик: Low)

`**POST** /api/v1/auth/logout` повинен заносити `jti` Access Token у Redis denylist до закінчення **TTL** токена та відкликати відповідний refresh-токен.

****SRS**-**FUN**-01.6b** (MoSCoW: Should | Ризик: Low)

Система повинна надавати відкликання всіх активних сесій користувача з інвалідацією всіх token families.

****SRS**-**FUN**-01.6c1** (MoSCoW: Must | Ризик: Low)

Автентифікований користувач повинен мати можливість змінити пароль після перевірки поточного пароля. Після зміни всі інші сесії відкликаються.

****SRS**-**FUN**-01.6c2** (MoSCoW: Must | Ризик: Low)

Система повинна надавати reset password через одноразовий токен: ентропія ≥**128** bit, зберігання лише hash, **TTL** 1 год, single-use, інвалідація після використання. Відповідь і таймінг для відомого та невідомого email повинні бути однаковими.

****SRS**-**FUN**-01.6d** (MoSCoW: Must | Ризик: Low)

Система повинна верифікувати email при реєстрації через криптографічний токен: ентропія ≥**128** bit, зберігання лише hash, **TTL** 24 год, single-use, інвалідація після використання.

****SRS**-**FUN**-01.7** (MoSCoW: Must | Ризик: Low)

Система повинна вимагати **MFA**/**TOTP** (**RFC** **6238**) для **ACT**-05. **TOTP** secret зберігається зашифрованим; recovery codes зберігаються як hash; повторне використання **TOTP** у межах кроку заборонене.

****SRS**-**FUN**-01.8** (MoSCoW: Must | Ризик: High)

Система повинна захищати cookie-based refresh/logout від **CSRF**: double-submit **CSRF** token, перевірка `Origin`/`Referer` за allowlist, `SameSite=Strict`. Невалідний **CSRF** → `**403** Forbidden`.

****SRS**-**FUN**-01.9** (MoSCoW: Must | Ризик: Medium)

Система повинна підтримувати керування ключами **JWT**: `kid`, **JWKS**, ротація, ізоляція приватного ключа, deny-list `jti`, **TTL** denylist дорівнює залишку **TTL** токена.

****SRS**-**FUN**-01.10** (MoSCoW: Must | Ризик: Medium)

Система повинна підтримувати життєвий цикл **MFA**: enrollment, підтвердження, recovery codes, адміністративний reset, audit подій enrollment/reset.

****SRS**-**FUN**-02.1** (MoSCoW: Must | Ризик: Low)

Публічний профіль повинен містити nickname, `avatar_url`, статуси бібліотеки, публічні цитати, Telegram-контакт лише за активної згоди. `avatar_url`: **HTTPS**, max **2048** символів, дозволені `image/jpeg`, `image/png`, `image/webp`, max 2 МБ; сервер повторно кодує зображення.

****SRS**-**FUN**-02.2** (**GDPR** ст. 7 | MoSCoW: Must | Ризик: Low)

Система повинна надавати інтерфейс надання/відкликання згоди. Запис згоди містить `consentType`, `version`, `grantedAt`, `revokedAt`, `evidence`, `actorId`. Відкликання діє негайно; інвалідація кешу профілю — згідно **SRS**-**NFR**-**PERF**-06.

****SRS**-**FUN**-02.3a** (**GDPR** ст. 15, 20 | MoSCoW: Must | Ризик: Medium)

Система повинна виконувати **DSR** export: підтвердження особи, асинхронна задача, статуси `requested/processing/ready/failed`, строк ≤30 днів, формат **JSON**. Експорт включає профіль, бібліотеку, цитати, відгуки, коментарі, згоди, історію прогресу; audit logs виключені або псевдонімізовані.

****SRS**-**FUN**-02.3b** (**GDPR** ст. 17 | MoSCoW: Must | Ризик: Medium)

Система повинна виконувати каскадне видалення облікового запису та пов’язаних **PII** протягом ≤30 днів. Виняток — обов’язкові audit logs у псевдонімізованому вигляді. Видалення включає профіль, бібліотеку, цитати, відгуки, коментарі, дружні зв’язки, сесії, **OCR**-результати; кластери перераховуються.

****SRS**-**FUN**-02.3c** (**GDPR** ст. 17 | MoSCoW: Must | Ризик: Medium)

Система повинна вести deletion log/tombstone і повторно застосовувати його після відновлення з бекапу. Після restore deletion log застосовується до завершення процедури відновлення.

****SRS**-**FUN**-02.4** (MoSCoW: Should | Ризик: Low)

Система повинна управляти сутністю «Зв’язок дружби»: створення запиту, прийняття, відхилення, розрив. Вимога розміщується лише в цьому модулі; дублювання в модулі кімнат заборонене.

### 3.2 Модуль каталогу та бібліотеки

****SRS**-**FUN**-03.1a** (**ISO** **2108** | MoSCoW: Must | Ризик: Low)

Система повинна перевіряти синтаксичну правильність **ISBN**-10 або **ISBN**-13.

****SRS**-**FUN**-03.1b** (**ISO**/**IEC** **15420** | MoSCoW: Must | Ризик: Low)

Система повинна перевіряти контрольну цифру **ISBN**-13 за **EAN**-13 (mod 10), **ISBN**-10 — за mod 11 з урахуванням `X`.

****SRS**-**FUN**-03.2** (MoSCoW: Must | Ризик: Low)

Система повинна конвертувати валідний **ISBN**-10 у **ISBN**-13 перед зверненням до зовнішніх джерел.

****SRS**-**FUN**-03.3a** (MoSCoW: Must | Ризик: Medium)

Система повинна запитувати метадані спочатку в Open Library **API**. `**200**` → positive cache. `**404**` → запит до Google Books **API**. Якщо обидва джерела повернули `**404**` → остаточна відповідь `**404** Not Found` з пропозицією ручного введення.

****SRS**-**FUN**-03.3b** (MoSCoW: Must | Ризик: Medium)

Система повинна застосовувати таку матрицю статусів:

| Стан провайдера | Дія | Відповідь клієнту |
| --- | --- | --- |
| Open Library 200 | Повернути метадані, positive cache 24 год | 200 |
| Open Library 404 | Запитати Google Books | Залежно від Google Books |
| Open Library 429/5xx/timeout | Не кешувати негативно; за можливості запитати Google Books | 200 або 503 |
| Google Books 200 | Повернути метадані, positive cache 24 год | 200 |
| Google Books 404 | Обидва 404 | 404 + manual entry |
| Обидва 429/5xx/timeout або CB open | Не кешувати негативно | 503 + `Retry-After` + manual entry |
| Некоректна відповідь upstream | Не кешувати | 502 |

`**404**` не вважається збоєм Circuit Breaker. Помилки `**429**`, `5xx`, timeout не кешуються.

****SRS**-**FUN**-03.3c** (MoSCoW: Must | Ризик: Low)

Система повинна забезпечувати single-flight дедуплікацію паралельних запитів для однакового **ISBN**.

****SRS**-**FUN**-03.4** (MoSCoW: Must | Ризик: Low)

Система повинна кешувати виключно успішні позитивні відповіді зовнішніх **API** в Redis із **TTL** 24 год. Помилки `**429**`, `5xx`, таймаути не кешуються.

****SRS**-**FUN**-03.5** (MoSCoW: Should | Ризик: Low)

Система повинна підтримувати точний/префіксний пошук за нормалізованою назвою з case-insensitive collation та індексом. Anchored regex дозволений лише для префіксного пошуку. Максимальна видача — 50 записів, stable sort за `title`, `_id`. Full-text search Atlas не використовується.

****SRS**-**FUN**-03.6** (MoSCoW: Must | Ризик: Low)

Система повинна постійно зберігати метадані успішно знайдених або введених вручну книг у MongoDB та підтримувати ручне введення при недоступності зовнішніх каталогів.

****SRS**-**FUN**-04.1** (MoSCoW: Must | Ризик: Medium)

Система повинна підтримувати записи бібліотеки через детерміновану state machine. Інваріант: `0 ≤ current_page ≤ page_count`, де `current_page` і `page_count` — цілі числа. Для книг із ручним введенням `page_count` обов’язковий. Для зовнішніх книг із `page_count = null` перехід у `finished` заборонений до ручного встановлення `page_count`.

| Поточний стан | Дозволений перехід | Умова |
| --- | --- | --- |
| planned | reading | — |
| planned | dropped | — |
| reading | finished | `page_count != null` та `current_page = page_count` |
| reading | dropped | — |
| dropped | reading | — |
| finished | reading | повторне читання, `current_page` скидається за рішенням користувача |

****SRS**-**FUN**-04.2** (MoSCoW: Must | Ризик: Low)

Система повинна зберігати відгуки з унікальністю `(userId, bookId)`, рейтингом 1–5, текстом ≤**1000** символів, `page` у діапазоні `0 ≤ page ≤ page_count`, `has_spoiler: boolean`.

****SRS**-**FUN**-04.3** (MoSCoW: Must | Ризик: Low)

Система повинна зберігати `has_spoiler` та `page` відгуку. Маскування відгуків виконується виключно єдиною серверною політикою **SRS**-**FUN**-08.4a. Окрема логіка маскування в модулі бібліотеки не допускається.

### 3.3 Модуль OCR та асинхронного контуру

****SRS**-**FUN**-05.1** (MoSCoW: Must | Ризик: Medium)

`**POST** /api/v1/ocr-jobs` повинен приймати **JPEG**/**PNG**/WebP ≤5 МБ, max **4000**×**4000** пікселів, у строгому порядку: валідація формату/лімітів → magic bytes → зачистка **EXIF** → запис у temp storage → постановка в BullMQ. Якщо будь-який крок не пройдено, створення job заборонене, temp-файл видаляється, повертається відповідна помилка.

****SRS**-**FUN**-05.2** (MoSCoW: Must | Ризик: Low)

Система повинна вимагати `Idempotency-Key` (**UUID**) для створення **OCR**-завдань. Відсутній/невалідний ключ → `**400** Bad Request`.

****SRS**-**FUN**-05.3** (MoSCoW: Must | Ризик: Low)

Idempotency scope: `(userId, endpoint, Idempotency-Key)`, **TTL** 24 год. Система зберігає hash тіла запиту та відповідь. Повторний запит із тим самим ключем і тим самим тілом → `**200** OK` або `**202** Accepted` з тим самим `jobId`. Той самий ключ із іншим тілом → `**422** Unprocessable Entity`. Конкурентні запити з тим самим ключем серіалізуються атомарним Redis lock; переможець створює job, інший отримує збережений результат або `**409** Conflict` із `Retry-After`, якщо результат ще не готовий.

****SRS**-**FUN**-05.4** (MoSCoW: Must | Ризик: Low)

Система повинна повертати `**202** Accepted` та `jobId` після успішного додавання завдання до BullMQ.

****SRS**-**FUN**-05.5** (MoSCoW: Must | Ризик: Medium)

Фоновий воркер повинен виконувати **OCR** (ukr+eng) із жорстким wall-clock таймаутом 30 с на одне завдання.

****SRS**-**FUN**-05.6** (MoSCoW: Must | Ризик: Low)

Retry policy: 1 initial attempt + 3 retries = 4 total attempts. Затримка `min(5·2^n, 60) ±20%` секунд для `n = 1..3`. Биті/деформовані файли не retry-яться.

****SRS**-**FUN**-05.7a** (MoSCoW: Must | Ризик: Low)

Після вичерпання всіх спроб job переміщується до **DLQ** (`failed set`).

****SRS**-**FUN**-05.7b** (MoSCoW: Should | Ризик: Low)

**ACT**-05 повинен мати право ручного restart job із **DLQ** лише за наявності оригінального файлу в temp storage. Якщо файл видалено — restart заборонений, повертається `**410** Gone`.

****SRS**-**FUN**-05.8** (MoSCoW: Must | Ризик: Low)

Результат успішного **OCR** зберігається в MongoDB 30 діб, публікується в WebSocket-канал користувача, доступний через `**GET** /api/v1/ocr-jobs/{id}`.

****SRS**-**FUN**-05.9** (MoSCoW: Must | Ризик: Low)

Вихідний файл видаляється одразу після завершення обробки або через 24 год (що раніше). Для `failed` job файл зберігається до рішення адміністратора або до 24 год. Після видалення файлу restart із **DLQ** повертає `**410** Gone`.

### 3.4 Модуль цитат та агрегації

****SRS**-**FUN**-06.1** (MoSCoW: Must | Ризик: Low)

Цитата: довжина ≤**500** code points, `0 ≤ page ≤ page_count`, видимість `private/public`, `has_spoiler`, `consent: boolean` (default `false`).

****SRS**-**FUN**-06.2** (MoSCoW: Must | Ризик: Low)

Особистий кабінет повинен надавати власні цитати, згруповані за книгами, з курсорною пагінацією, редагуванням, видаленням, зміною видимості.

****SRS**-**FUN**-07.1** (MoSCoW: Must | Ризик: Low)

Нормалізація цитати: нижній регістр, Unicode **NFC**, видалення пунктуації категорії `P*`, крім апострофа з уніфікацією `’ ʼ '`.

****SRS**-**FUN**-07.2a** (MoSCoW: Must | Ризик: Medium)

Candidate generation: MinHash/**LSH**, shingle = 3 words. Параметри bands/rows конфігуруються та калібруються на еталонному корпусі ≥**500** пар.

****SRS**-**FUN**-07.2b** (MoSCoW: Must | Ризик: Medium)

Verification: Jaro-Winkler ≥0.92. Калібрування на еталонному корпусі ≥**500** пар: precision ≥0.9, recall ≥0.9. Результат перевіряється CI gate.

****SRS**-**FUN**-07.3** (MoSCoW: Must | Ризик: Medium)

Знеособлена статистика кластера публікується лише за `≥5` унікальних авторів із `consent=true`, з мінімальним порогом стабільності та округленням лічильників.

****SRS**-**FUN**-07.4** (MoSCoW: Must | Ризик: Low)

Кластер автоматично знімається з публікації, якщо кількість унікальних авторів стає <5.

### 3.5 Модуль читацьких кімнат, зв’язків та спойлерів

****SRS**-**FUN**-08.1a** (MoSCoW: Must | Ризик: Medium)

WebSocket handshake повинен автентифікувати користувача, перевіряти `Origin` за allowlist, перевіряти членство в кімнаті до підписки на події. Неавторизована підписка → `**403**`/закриття з’єднання.

****SRS**-**FUN**-08.1b** (MoSCoW: Must | Ризик: Medium)

Кожна кімната має монотонний `sequence number`. Клієнт передає `lastEventId`; сервер виконує replay пропущених подій із персистентного сховища з вікном 24 год. Події мають версіоновану схему.

****SRS**-**FUN**-08.1c** (MoSCoW: Must | Ризик: Medium)

WebSocket повинен мати max payload ≤1 КБ, rate limit повідомлень, backpressure, Redis adapter для pub/sub, reconnect із повторною автентифікацією.

****SRS**-**FUN**-08.2** (MoSCoW: Must | Ризик: Low)

Управління кімнатою: створення, дедлайн, `visibility` з enum, вступ за токеном запрошення, вихід, виключення. Токен запрошення: ентропія ≥**128** bit, **TTL** ≤72 год, single-use, revocation. Коментарі: ≤**1000** символів, прив’язка до сторінки.

****SRS**-**FUN**-08.3** (MoSCoW: Must | Ризик: Low)

Добова норма сторінок: `max(0, ceil((page_count - current_page) / max(days_left, 1)))` у **IANA** timezone кімнати з урахуванням **DST**.

****SRS**-**FUN**-08.4a** (MoSCoW: Must | Ризик: High)

Єдина серверна spoiler policy застосовується до **REST**, WebSocket і коментарів. За замовчуванням контент `**MASKED**`. Початкова відповідь не містить поле `text` для замаскованого контенту.

| № | Умова | Результат |
| --- | --- | --- |
| 0 | Інші випадки | `MASKED` |
| 1 | Запитувач є автором контенту | `FULL` |
| 2 | `has_spoiler = true` | `MASKED`, крім автора |
| 3 | `has_spoiler = false`, viewer authenticated, книга є в бібліотеці, `page != null`, `current_page != null`, `page <= current_page` | `FULL` |
| 4 | `has_spoiler = false`, viewer authenticated, книга є в бібліотеці, але `page = null` або `current_page = null` або `page > current_page` | `MASKED` |
| 5 | `has_spoiler = false`, viewer — гість або не має книги в бібліотеці, контент — публічна цитата/відгук на публічному профілі | `FULL` як явний business exception |
| 6 | Коментар кімнати, viewer — ACT-03/ACT-04, `has_spoiler = false`, `page != null`, `current_page != null`, `page <= current_page` | `FULL` |
| 7 | Коментар кімнати, viewer — ACT-03/ACT-04, але progress відсутній або `page > current_page` | `MASKED` |
| 8 | Гість намагається отримати коментар кімнати | `403` |

****SRS**-**FUN**-08.4b** (MoSCoW: Must | Ризик: High)

Розкриття замаскованого контенту виконується окремим авторизованим endpoint після явного підтвердження користувача. Endpoint повторно застосовує **SRS**-**FUN**-08.4a, не повертає текст у початковій відповіді, пише audit event. Гість не може викликати reveal. Для `has_spoiler=true` reveal дозволений лише автентифікованому користувачу після підтвердження; автор отримує `**FULL**` без reveal.

****SRS**-**FUN**-08.5** (MoSCoW: Must | Ризик: Low)

Курсорна пагінація: default 20, max 50 записів; пагінація застосовується після серверної фільтрації.

---

## 4. Non-Functional Requirements

### 4.1 Продуктивність та масштабованість

****SRS**-**NFR**-**PERF**-01a** (MoSCoW: Must | Ризик: High)

**REST** **API** без **OCR**: ≥**500** **RPS** на екземпляр 2 vCPU / 4 ГБ **RAM**, read/write 80/20, payload ≤10 КБ, MongoDB seed ≥**100** **000** книг і ≥1 **000** **000** записів бібліотеки, Redis warm, connection pool ≥50. k6 thresholds: `http_req_failed < 0.1%`, read `p95 ≤**200** мс`, write `p95 ≤**300** мс`, exit code 0.

****SRS**-**NFR**-**PERF**-01b** (MoSCoW: Must | Ризик: High)

**REST** + **OCR**: **OCR** worker із **CPU** limit 2 vCPU, concurrency 1; **REST** p95 не деградує >20% від **PERF**-01a; **OCR** throughput ≥5 jobs/min; **OCR** p95 ≤15 с на 2 Мпикс. k6 + Prometheus + BullMQ metrics.

****SRS**-**NFR**-**PERF**-02** (MoSCoW: Must | Ризик: Medium)

**SLI**: read p95 ≤**200** мс, write p95 ≤**300** мс; успішність ≥95% за 30 діб; 4xx не входять у numerator. Prometheus buckets: 0.1/0.2/0.3/0.5/1/2 с. Multi-window burn-rate: 2% budget/1h, 5%/6h, 10%/3d. Алерт при порушенні.

****SRS**-**NFR**-**PERF**-03** (MoSCoW: Must | Ризик: High)

**OCR** прийом: p95 ≤**150** мс до `**202**`. Обробка: p95 ≤15 с на 2 Мпикс. **CER** ≤5% лише на typed-text dataset ukr+eng із preprocessing; для фото — окремий dataset із зафіксованим **CER** baseline та 95% CI. Benchmark ≥**200** зображень.

****SRS**-**NFR**-**PERF**-04** (MoSCoW: Must | Ризик: Medium)

WebSocket: p95 ≤**500** мс при **1000** одночасних з’єднаннях на екземпляр, 0.2 msg/s на з’єднання, payload ≤1 КБ, Socket.io Redis adapter, sticky sessions, backpressure, max payload ≤1 КБ, rate limit.

****SRS**-**NFR**-**PERF**-05** (MoSCoW: Must | Ризик: Low)

Redis metadata cache hit ratio ≥80% після 30 хв прогріву, Zipf s=1.0, fixed key universe, **TTL** 24 год, negative cache відсутній. Вимірюється `redis_exporter`.

****SRS**-**NFR**-**PERF**-06** (MoSCoW: Must | Ризик: Medium)

Інвалідація серверного кешу публічного профілю після зміни/відкликання згоди, edit/delete цитати, зміни видимості або прогресу — ≤1 с (p95).

### 4.2 Безпека, приватність та комплаєнс

****SRS**-**NFR**-**SEC**-01** Inbound **TLS** 1.3; outbound мінімум **TLS** 1.2; дискові томи **AES**-**256** (**LUKS**); бекапи шифруються. **TLS** для Redis/Mongo — обов’язковий або явно прийнятий risk у deployment record.

****SRS**-**NFR**-**SEC**-02** Rate limiting Token Bucket у Redis:

| Клас | Capacity | Refill | Scope |
| --- | --- | --- | --- |
| Гість | 60 | 1/s | IP + trusted proxy allowlist |
| Читач | 300 | 5/s | userId |
| Адміністратор | 1000 | 16.67/s | userId |
| Auth | 10 | 0.1667/s | IP + login |
| OCR upload | 10/min | 0.1667/s | userId |
| WS messages | 20/s | 20/s | userId + room |

Перевищення → `**429**` + `Retry-After`. Redis failure: Auth fail-closed; публічні каталоги fail-open; **OCR**/WS fail-closed.

****SRS**-**NFR**-**SEC**-03** **OWASP** **ASVS** v4.0.3 Level 2, V2–**V14**. Обов’язкові: Zod validation, NoSQL injection protection, output encoding, magic bytes, **SSRF** allowlist, **CORS**, **CSP**, **HSTS**, `X-Content-Type-Options`, **CSRF**, body limits 5 МБ media / **100** КБ **JSON**. Evidence: Semgrep rules, **OWASP** **ZAP** baseline, testssl.sh, Trivy.

****SRS**-**NFR**-**SEC**-04** **GDPR**/ЗКУ: **DSR** ≤30 днів; backup retention 14 діб; deletion log reapplied; audit logs append-only/hash-chain; **PII** pseudonymized; breach notification процедура ст. 33/34 — ≤72 год; retention schedule; **DPIA** для **OCR** та соціального графа.

****SRS**-**NFR**-**SEC**-05** Приватні ресурси/чужі сутності → `**404**`; конфлікт прав на існуючі публічні/спільні ресурси → `**403**`; timing equalization.

****SRS**-**NFR**-**SEC**-06** Audit log: append-only/hash-chain або immutable storage; схема `eventId`, `timestamp`, `actorId`, `actorRole`, `action`, `resourceType`, `resourceId`, `correlationId`, `ipHash`, `result`; **PII** pseudonymized; доступ **ACT**-05; retention ≥1 рік; захист від модифікації/видалення.

****SRS**-**NFR**-**SEC**-07** **JWT**/session: `kid`, **JWKS**, ротація, claims, clock skew ≤30 с, `jti` denylist **TTL** = remaining token **TTL**; refresh token hash at rest.

****SRS**-**NFR**-**SEC**-08** WebSocket: auth handshake, per-room authorization, Zod message schema, rate limit, max payload, backpressure, reconnect token refresh, Redis adapter.

****SRS**-**NFR**-**SEC**-09** File storage: temp storage access control, virus scan, decompression bomb protection, magic bytes, **EXIF** strip, cleanup job, retention 24 год, no public **URL**.

****SRS**-**NFR**-**SEC**-10** External APIs: `User-Agent`, quota, versioning, **429** backoff, CB per provider, no negative cache, **SSRF** allowlist.

****SRS**-**NFR**-**SEC**-11** Secrets/config: поза репозиторієм, ротація, no secrets in logs, `pino.redact`.

### 4.3 Надійність, операційність та CI/CD

****SRS**-**NFR**-**REL**-01** Доступність ≥99.5% на календарний місяць у Docker Compose без урахування узгоджених maintenance windows. **RTO** ≤1 год, **RPO** ≤24 год. Blackbox `/ready` кожні 30 с. Визнається, що 99.5% залежить від single-host.

****SRS**-**NFR**-**REL**-02** Circuit Breaker: open після 5 послідовних 5xx/timeout (**404** не failure), timeout 3 с, open 30 с, half-open probe. Per provider.

****SRS**-**NFR**-**REL**-03** Observability: pino **JSON** + `correlationId`; Prometheus; alerts: event loop lag p99 >**100** мс, queue depth > threshold, **DLQ** >0, Redis memory >80%, Mongo connections >80%, WS connections > threshold.

****SRS**-**NFR**-**REL**-04a** Branch coverage бізнес-логіки ≥80%.

****SRS**-**NFR**-**REL**-04b** Spoiler module: branch coverage **100**%, fast-check `numRuns ≥**10000**`, Stryker mutation score ≥80%.

****SRS**-**NFR**-**REL**-04c** **WCAG** 2.2 AA: axe-core 0 critical/serious; Playwright + manual checklist.

****SRS**-**NFR**-**REL**-05** Queue backpressure: max queue length, **OCR** concurrency, **CPU**/**RAM** limits, **DLQ** alert, retry policy, BullMQ graceful shutdown, `removeOnComplete`/`removeOnFail` limits.

****SRS**-**NFR**-**REL**-06** **API**/MongoDB: OpenAPI 3.1 contract; **RFC** **9457** errors; schema validation; migrations; unique indexes (`email`, `(userId, bookId)`, `(userId, bookId, createdAt)`); optimistic locking `If-Match`/`version` для кімнат.

****SRS**-**NFR**-**REL**-07** Graceful shutdown: **SIGTERM**, drain 15 с, stop new requests, finish workers, close WS, close Mongo/Redis. CI gates: lint, unit, integration, **SAST**, **SCA**, image scan, **SBOM** CycloneDX; block Critical/High.

### 4.4 Frontend та клієнтські вимоги

****SRS**-**NFR**-FE-01** BarcodeDetector за наявності; інакше ручне введення. Авто-детекція не гарантується для браузерів без **API**.

****SRS**-**NFR**-FE-02** **ROI** cropping tool для **OCR** перед відправкою.

****SRS**-**NFR**-FE-03** Initial JS bundle ≤**300** КБ gzip; **TTI** ≤2.5 с на Lighthouse mobile Slow 4G, Moto G4; route-level code splitting; bundle analyzer budget.

****SRS**-**NFR**-FE-04** Останні 2 мажорні версії Chrome, Firefox, Safari, Edge.

---

## 5. Verification Criteria

| Вимога ID | Джерело | Метод | Інструмент / Середовище | Критерій Pass / Fail |
| --- | --- | --- | --- | --- |
| SRS-FUN-01.1–01.10 | NIST 800-63B, RFC 7519/8725, GDPR | Integration, Security | Supertest, Vitest, Testcontainers Redis, otplib, fake timers, Schemathesis | Реєстрація/вхід/refresh/logout/CSRF/MFA/email tokens; HIBP fail-closed; bcrypt cost ≥12; TOTP replay blocked |
| SRS-FUN-02.1–02.4 | GDPR ст. 7, 15, 17, 20 | Integration, DSR drill | Supertest, Testcontainers Mongo, DSR script | Consent schema; export JSON; deletion cascade; deletion log reapply; friendship CRUD |
| SRS-FUN-03.1–03.6 | ISO 2108, ISO 15420 | Unit, Property, Contract | Vitest, fast-check, MSW/nock, Toxiproxy, Testcontainers Redis | ISBN validation; ISBN-10→13; provider status matrix; single-flight; positive-only cache |
| SRS-FUN-04.1–04.3 | — | Unit, Integration | Vitest, Supertest | State machine transitions; page invariants; spoiler policy delegated to 08.4a |
| SRS-FUN-05.1–05.9 | Idempotency draft | Integration, Fault Injection | Testcontainers Redis/Mongo, Supertest, BullMQ harness, Toxiproxy | Idempotency scope; retry 1+3; DLQ; temp cleanup; 410 after file deletion |
| SRS-FUN-06.1–06.2 | — | Unit, Integration | Vitest, Supertest | Quote length ≤500; grouping; cursor pagination |
| SRS-FUN-07.1–07.4 | — | Unit, Mutation, Benchmark | Vitest, Stryker, corpus ≥500 pairs | Jaro-Winkler ≥0.92; precision/recall ≥0.9; k≥5; auto-unpublish |
| SRS-FUN-08.1–08.5 | RFC 6455 | Property-based, WS Integration, Load | fast-check ≥10000, Artillery socket.io-engine, Testcontainers Redis | WS auth/origin; replay; spoiler invariants 100%; reveal audit; pagination |
| SRS-NFR-PERF-01a–01b | — | Load / Stress | k6 thresholds, Prometheus, Grafana, mongodb_exporter | ≥500 RPS; p95 read ≤200/write ≤300; 5xx <0.1%; OCR throughput ≥5/min |
| SRS-NFR-PERF-02 | — | SLO Analysis | Prometheus, promtool, Grafana | SLI ≥95%; burn-rate alerts |
| SRS-NFR-PERF-03 | — | Load, Benchmark | k6, Prometheus, CER calculator | OCR p95 ≤15 s; CER ≤5% typed; photo baseline recorded |
| SRS-NFR-PERF-04 | — | Load WS | Artillery + Redis adapter | p95 ≤500 ms at 1000 connections |
| SRS-NFR-PERF-05 | — | Analysis + Load | redis_exporter, k6 Zipf | Hit ratio ≥80% after 30 min |
| SRS-NFR-PERF-06 | — | Integration | Supertest, Redis monitor | Cache invalidation ≤1 s p95 |
| SRS-NFR-SEC-01–03, 06–11 | OWASP ASVS L2 | Scan / Audit | testssl.sh, Semgrep, Gitleaks, Trivy, ZAP baseline, npm audit, CycloneDX | TLS 1.3; no Critical/High; ASVS evidence; secrets absent |
| SRS-NFR-SEC-04 | GDPR | Audit / Drill | DSR script, backup restore drill | DSR ≤30 days; breach procedure; deletion log reapply |
| SRS-NFR-SEC-05 | — | Integration | Supertest timing tests | 404/403 policy and timing equalization |
| SRS-NFR-REL-01 | — | Demonstration | Blackbox exporter, RTO drill | Availability ≥99.5%; RTO ≤60 min; RPO ≤24 h |
| SRS-NFR-REL-02 | — | Fault Injection | Toxiproxy + Vitest | CB opens after 5 failures; 404 not failure |
| SRS-NFR-REL-03 | — | Inspection | promtool, pino test, Grafana | correlationId present; alert rules valid |
| SRS-NFR-REL-04a–04c | WCAG 2.2 | Coverage / Mutation / Accessibility | c8/Vitest, Stryker, axe-core, Playwright | Branch ≥80%; spoiler 100%; mutation ≥80%; 0 critical axe |
| SRS-NFR-REL-05 | — | Load / Fault | BullMQ metrics, Toxiproxy | Backpressure; DLQ alert; graceful shutdown |
| SRS-NFR-REL-06 | OpenAPI 3.1 | Contract | Spectral, Schemathesis, Testcontainers Mongo | Contract valid; migrations; indexes; optimistic locking |
| SRS-NFR-REL-07 | — | CI Audit | GitHub Actions, Trivy, SBOM | Gates block Critical/High; SIGTERM drain 15 s |
| SRS-NFR-FE-01–04 | ISO 25010, WCAG 2.2 | Frontend Audit | Playwright, axe-core, Lighthouse CI, bundle analyzer | Bundle ≤300 KB gzip; TTI ≤2.5 s; browsers; ROI; fallback |