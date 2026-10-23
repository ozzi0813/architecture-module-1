# OrderFlow: Architecture under Change


OrderFlow не потребує переходу на мікросервіси. Проблема не в тому, що це modular monolith, а в тому, що **boundaries всередині нього протікають**, а **delivery process не використовує модульність**. Пропоную evolution plan у 4 кроки, який лишає один deployable, але виправляє три конкретні речі: Payments читає чужий repository, Notifications сидить у critical path checkout, Reporting сканує operational data в тому ж process. Перший кандидат на окремий deployment, якщо умови справдяться, це Reporting, а не Payments чи Orders. Для AI Time to Market реалістична ціль 2-3 місяці (точкова оцінка \~2,5), а не 1 місяць.

## 1. Поточна архітектура

OrderFlow це один process, одна in-memory database, один deployment artifact. Checkout координує Orders, Inventory, Payments і Notifications синхронно, в одному call stack. Reporting читає ті самі колекції напряму.

Найважливіші dependencies, які видно в коді:

- `PaymentService` отримує `OrderRepository` в конструкторі й сам читає order (коментар у коді: deliberate boundary leak). Payments знає внутрішню структуру Orders.
- `OrderService` бере ціни через `this.database.products`, тобто Orders залежить від даних Inventory в обхід `InventoryService`.
- `CheckoutService` після `complete()` робить `await notificationService.sendOrderConfirmation()`. Email у critical path.
- `ReportingService` читає `database.orders`, `payments`, `notifications` напряму і для кожного order робить лінійний пошук payment та notification (O(n·m)) синхронно, на тому ж event loop, що й checkout.
- Шаблон листа жорстко зашитий у `NotificationService` і деплоїться разом із усім застосунком.

**Діаграма поточної архітектури** додана нижче.

&#91;embedded content: поточна архітектура · один process, 3 проблемні залежності\]

Червоним позначено залежності, які збільшують cost of change: Payments читає чужий repository, checkout чекає на email, Reporting сканує operational data.

## 2. Запропонована архітектура

Цільовий стан: той самий modular monolith, але з явними contracts. Payments не знає про Orders і працює через `PaymentGateway` port (adapter для FakePay, пізніше другий provider). Notifications отримує подію `OrderCompleted` через in-process outbox і відправляє лист асинхронно з retry. Reporting будує read model з подій і більше не сканує operational collections. Inventory та Orders лишаються в одній синхронній транзакційній межі.

**Діаграма запропонованої архітектури** додана нижче.

&#91;embedded content: запропонована архітектура · один deployable, sync і async взаємодії\]

Sync-ланцюжок Checkout → Inventory → Payments лишається синхронним. Все, що не потребує негайної відповіді (`OrderCompleted` → Notifications і Reporting), іде асинхронно через outbox.

## 3. Що показують дані

| Metric | 6 міс. тому | Зараз |
| --- | --: | --: |
| Modules Changed per Feature | 1,7 | 4,3 |
| Change Lead Time | 2 дні | 6 днів |
| Average PR Size | 180 LOC | 520 LOC |
| Release Frequency | 5/тиж | 2/тиж |
| Rollback Rate | 2% | 9% |
| Production Incidents | 1/міс | 5/міс |
| MTTR | 35 хв | 95 хв |

Додатково: 48% feature PR зачіпають Orders і Payments одночасно, 37% змін notification templates вимагають повного regression checkout, два останні інциденти почались із latency spikes payment provider.

**Факти, припущення, причинність.** Фактом є самі числа й структура коду. Припущення: зростання Modules per Feature та PR Size спричинене протіканням boundaries (код це підтримує, але кореляція не доводить причину). Частина зростання Deployment Duration пояснюється test suite і ручними approvals, тобто це process, не architecture. Це я вважаю окремою гіпотезою, яку треба перевірити.

## 4. Decision summary

### Decision 1. Payments через port замість прямого доступу до Orders

- **Problem:** зміна payment rule зачіпає Orders і Payments (48% PR).
- **Evidence:** `PaymentService` приймає `OrderRepository`; Modules per Feature 4,3; Scenario B потребує другого provider з іншим error model.
- **Decision:** `PaymentService.charge({orderId, amountCents, currency, token})` отримує суму від Checkout; provider ховається за `PaymentGateway`. Прямий імпорт `OrderRepository` у Payments заборонено. Додати timeout і idempotency key на виклик provider.
- **Trade-offs:** Checkout передає трохи більше даних; з'являється ще один інтерфейс, який треба підтримувати.
- **Success:** зміна payment rule торкається 1-2 modules; fitness function нижче проходить.

### Decision 2. Notifications виходить із critical path

- **Problem:** checkout чекає на email, і його збій ламає успішне замовлення.
- **Evidence:** у `CheckoutService` `await sendOrderConfirmation` стоїть **після** `complete()` і всередині того ж `try`. Якщо email provider кине помилку (default failureRate 2%), `catch` викличе `inventoryService.release()` і `orderService.fail()` для вже оплаченого й завершеного замовлення. Клієнта списано, замовлення FAILED, склад повернено. Це конкретний defect, а не лише ризик. Scenario E прямо каже, що email не визначає успіх платежу.
- **Decision:** (а) негайний hotfix: виконувати notification після завершення checkout, поза compensation-логікою; (б) далі in-process outbox з retry і шаблонами як даними (config/feature flag), що можна змінити без deploy.
- **Trade-offs:** eventual delivery замість миттєвого листа; потрібен моніторинг outbox і dedup, щоб не надіслати лист двічі.
- **Success:** checkout success rate не залежить від mail provider; template change не торкається Orders/Checkout.

### Decision 3. Reporting отримує read model

- **Problem:** Reporting росте (traffic x3 за рік, очікується x8-10) і ділить process із checkout.
- **Evidence:** `buildSalesSummary` робить `find` по payments для кожного order, синхронно; Node виконує це на тому ж event loop, тож довгий звіт затримує checkout. Scenario C допускає дані з точністю до хвилин. Також `productPerformance` рахує виручку за поточною ціною товару, а не за ціною на момент замовлення: тихе розходження з реальною виручкою.
- **Decision:** Reporting підписується на події (`OrderCompleted`, `PaymentCaptured`, `PaymentFailed`) і веде власну проєкцію з цінами на момент продажу. Спершу в тому ж process, окрема структура даних.
- **Trade-offs:** eventual consistency (хвилини), дублювання даних, потреба в reconciliation.
- **Success:** p95 `/reports/sales` не залежить від кількості orders лінійно; checkout p95 не росте під час важких звітів.

### Decision 4. Boundaries тестуються автоматично

Додаємо dependency test (див. розділ 8). Без цього кроки 1-3 деградують за кілька спринтів.

### Що навмисно НЕ змінюємо

- **Orders ↔ Inventory лишаються синхронними й в одному sync-виклику.** Це «хороший» coupling: duplicate reserve не має двічі змінювати stock (Scenario D), і тест `duplicate inventory reservation` це вже перевіряє. Розрив цієї межі вимагав би saga й eventual consistency саме там, де бізнес не готовий на overselling. Існуюча idempotency в `reserve/release` (`operationKey`) залишається.
- **Checkout orchestrator лишається in-process.** Один власник end-to-end reliability, простий rollback логіки компенсації. Відкрите питання з proposal «хто відповідає за checkout reliability» закриваємо так: Checkout належить команді Orders.
- **Один deployable artifact на цьому етапі.** Команда 8 engineers + 2 QA, немає platform team, один instance з резервним.
- **Назви й публічні HTTP endpoints.** Зворотна сумісність API не потрібна для внутрішніх змін.

## 5. Відхилений proposal: п'ять independently deployable services

Пропозицію з `architecture-proposal.md` я **відхиляю в повному обсязі** і приймаю лише її кроки 1-2 (contracts і data ownership) та частину кроку 3 (async для flows без негайної відповіді).

Чому складніша architecture ще не виправдана:

- Checkout зараз це послідовність Orders → Inventory → Payments у одному call stack. Розділення створить distributed transaction і потребує saga, outbox і compensation. Це нові failure modes саме в revenue-critical flow.
- MTTR уже виріс з 35 до 95 хв. П'ять сервісів додають observability, tracing і incident response, а platform team немає. Ризик, що MTTR погіршиться ще більше, реальний.
- Великою частиною болю є process (спільне deployment window двічі на тиждень, один test environment, ручні approvals), який services самі по собі не виправляють.
- 5 services на 8 engineers: кожен service \~1,6 людини, bus factor гірший.
- Більшість вигод (ownership, менший blast radius, швидші зміни Notifications) дають boundaries і feature flags, без мережі.

## 6. Characteristics за пріоритетом

1. **Availability checkout та resilience до provider** (Scenario E). Напряму revenue. Готові поступитися: миттєвістю confirmation email і свіжістю звітів.
2. **Data consistency та reliability операцій** (Orders/Inventory/Payments, Scenario D). Готові поступитися швидкістю окремих операцій (короткі затримки, idempotency checks), але не коректністю stock і платежів.
3. **Modifiability та deployability** (Scenarios A, B, F). Це головна причина evolution. Готові платити додатковими interfaces і boundary tests.
4. **Maintainability / local reasoning.** Наслідок пункту 3, вимірюється через PR size і modules per feature.
5. **Scalability Reporting** (Scenario C). Важливо, але це проблема майбутніх 12 місяців, тому вирішуємо мінімально (read model), без окремої інфраструктури.

Оперативна простота (low operational cost) тут обмеження, а не ціль: бюджет зростає лише за вимірюваний результат.

## 7. Principles

| Principle | Зараз | Після proposal |
| --- | --- | --- |
| **Information Hiding** | Порушено: Payments знає схему order, Orders читає `database.products`, Reporting читає всі колекції | Payments бачить лише `charge(...)`; ціни Orders бере через Inventory API; Reporting працює з власною проєкцією |
| **Explicit Contracts** | Відсутні; contract = структура об'єктів у shared DB | `PaymentGateway`, події `OrderCompleted`/`PaymentCaptured`, contract tests |
| **Dependency Direction** | Checkout → всі (ок), але Payments → Orders, Reporting → усі storage | Залежності йдуть до Checkout і від нього, Payments і Notifications не залежать від Orders |
| **Local Reasoning** | Слабко: зміна template потребує regression checkout (37%) | Зміна template/payment rule розуміється в межах одного module |
| **Fitness Functions** | Немає | Dependency test і latency/behaviour тести в CI |

Не всі principles застосовуються однаково: High Cohesion і Ownership лишаються на другому плані, бо команда мала й «ownership per module» ще не потрібен.

## 8. Scenarios

| Scenario | Stimulus → Response | Affected | Response measure | Потрібен окремий service? |
| --- | --- | --- | --- | --- |
| A: Notifications | зміна template 2-3 рази/тиж → без deploy | Notifications | 0 змін в Orders/Checkout, rollback експерименту < 5 хв | Ні, config + flag |
| B: другий provider | outage/ новий регіон → fallback | Payments | додавання adapter ≤ 2 modules | Ні, `PaymentGateway` |
| C: Reporting x8-10 | більше запитів → стабільний checkout | Reporting | checkout p95 не росте при x8 звітів; staleness ≤ 5 хв | Можливо пізніше |
| D: Inventory | повтор reserve/release → stock не змінюється | Inventory | 0 duplicate списань | Ні, лишаємо |
| E: Checkout availability | provider повільний 2-3 с / недоступний → checkout не падає через email, timeout до provider | Checkout, Payments, Notifications | success rate ≥ 99% без урахування відмов provider, p95 ≤ погодженого threshold | Ні |
| F: discount rule | нове правило щомісяця → коректна сума до списання | Orders/Pricing, Reporting | зміна ≤ 2 modules; сума в email, платежі й звітах збігається | Ні |

## 9. Change cost analysis

Допущення: Hourly Cost $50 для обох варіантів. Оцінки експертні, їх треба звіряти з реальними даними після першого кварталу.

**Change 1: додати payment rule (fraud rule), Scenario B/F.**

|  | Поточна | Запропонована |
| --- | --- | --- |
| Modules touched | 3-4 (Payments, Orders, Checkout, Reporting) | 1-2 (Payments, інколи Checkout) |
| Implementation | 16 h | 14 h |
| Coordination | 8 h (спільне планування Orders/Payments) | 2 h |
| Testing | 12 h (повний regression checkout) | 6 h (Payments contract tests + checkout smoke) |
| Release | 8 h (весь release, спільне window) | 4 h (менший scope, частіші релізи) |
| **Разом** | **44 h × $50 = $2,200** | **26 h × $50 = $1,300** |

Architecture Tax = $2,200 − $1,300 = **$900 на зміну**. Baseline: схожа зміна в кращій boundary; джерело цифр: 1,7 modules per feature 6 місяців тому проти 4,3 зараз.

**Change 2: notification experiment (Scenario A).** Поточна: 2 modules (Notifications + Checkout), 37% випадків повний regression, \~12 h = $600. Запропонована: template як config/feature flag, \~3 h = $150, без release. Різниця $450 × \~125 змін на рік (2,5/тиж) ≈ **$56k/рік**.

**Інвестиція.** Кроки 1-3 приблизно 300-350 engineering hours (\~$15-17k, \~6-8 тижнів часткової зайнятості 2-3 engineers). Окупність за Change 2 приблизно 3-4 місяці після впровадження. Невизначеність велика: частота змін і реальні години треба валідувати.

## 10. Risk analysis

Формула: Expected Risk Cost = Probability × Business Impact. Період: 1 місяць, якщо не вказано інше. Значення це явні припущення.

| Risk | Probability | Business Impact | Expected Risk Cost | Mitigation |
| --- | --- | --- | --- | --- |
| **R1. Збій/затримка payment provider знижує checkout availability.** Два останні інциденти почались з latency spikes, timeout немає | 50%/міс | $15k (втрачений revenue + recovery) | $7,5k | timeout, circuit breaker, retry з idempotency key, потім fallback provider (Scenario B) |
| **R2. Збій email після оплати переводить order у FAILED і повертає склад** (дефект у `CheckoutService`) | 2% на checkout; при \~1000 checkout/міс це \~20 випадків | $75 на випадок (refund/support/ручний fix) | \~$1,5k | hotfix notification поза compensation, regression test, далі outbox |
| **R3. Reporting під зростанням x8-10 затримує checkout (спільний event loop)** | 40% за 12 міс | $10k | $4k за період | read model, ліміти на date range, потім окремий worker/process |
| **R4. (створює саме рішення) Outbox/read model: втрачені або задубльовані листи, drift звітів** | 20% | $3k | $600 | dedup key, моніторинг outbox lag, reconciliation job, alert на розбіжність звітів і payments |
| **R5. (створює рішення) Нові interfaces застарівають або обходяться, а boundaries знову деградують** | 30% | $5k | $1,5k | dependency test у CI, review rule, метрика modules per feature |

R2 важливий ще й тому, що це найдешевше виправлення (години) з найпомітнішим ефектом для довіри клієнтів.

## 11. Metrics та Fitness Functions

| Metric | Зараз → ціль | Чому важлива | Напрям |
| --- | --- | --- | --- |
| Modules Changed per Feature | 4,3 → < 2,5 (через 6 міс) | Прямий індикатор протікання boundaries | ↓ |
| Change Lead Time | 6 днів → ≤ 3 дні | Підсумок delivery friction | ↓ |
| Rollback Rate | 9% → ≤ 4% | Менший scope релізу, кращі contracts | ↓ |
| Checkout success rate та p95 latency | baseline виміряти в першому тижні | Захищає пріоритет №1 | success ↑, p95 ↓ |
| MTTR | 95 хв → < 45 хв | Не дати складності вирости | ↓ |
| Average PR Size | 520 → < 250 LOC | Швидший review, особливо при AI | ↓ |

Метрики треба інтерпретувати з обережністю: на них впливають і process-зміни, тому окремо відстежуємо, які зміни зроблені в architecture, а які в process.

### Fitness Functions (автоматизовані)

1. **Dependency rule:** у `src/payments/` немає згадок `OrderRepository` чи `/orders/`, а `PaymentService` не приймає repository в конструкторі. Витік Payments → Orders зараз іде через DI в `app.js`, а не через import, тому перевірка самих import-ів його б не зловила. `OrderRepository` створюється лише в `orders/` і в `app.js` (composition root). `reporting/` не звертається до `database.orders/payments/notifications` напряму.
2. **Checkout resilience:** тест із notification provider, що завжди кидає помилку, очікує `order.status === "COMPLETED"`, `payment.status === "CAPTURED"` і що stock не повернуто.
3. **Checkout latency:** при mail latency 5 с checkout p95 не перевищує threshold payment path.
4. **Notification change:** PR, що змінює лише template, не змінює файли в `orders/` та `checkout/` (review rule / CI check).

Скелет dependency test (Node test runner, без нових залежностей):

```js
import test from "node:test";
import assert from "node:assert/strict";
import { readdirSync, readFileSync } from "node:fs";

function sources(dir) {
  return readdirSync(dir, { recursive: true })
    .filter((f) => f.endsWith(".js"))
    .map((f) => `${dir}/${f}`);
}

// Витік Payments -> Orders іде через DI (app.js), тому шукаємо згадки, а не лише import.
// Поки крок 2 плану не виконано, тест падає: це і є baseline.
test("payments does not know about orders", () => {
  for (const file of sources("src/payments")) {
    assert.doesNotMatch(
      readFileSync(file, "utf8"),
      /orderRepository|OrderRepository|\/orders\//,
      file
    );
  }
});
```

**Що може спростувати рішення:** якщо через 3 місяці modules per feature не нижчий за \~3,5, а lead time не змінився, значить причина була не в boundaries (а в process або в самому характері змін), і план треба переглянути до будь-яких нових інвестицій.

## 12. Умови майбутньої evolution

**Відповідь на питання «який module першим стане independently deployable».** Не зараз. Якщо умови справдяться, першим буде **Reporting** (read-only, eventual consistency прийнятна, є прямий scaling driver, не бере участі в checkout write path). Другий варіант дешевий: Notifications як окремий worker process з тією самою кодовою базою.

Тригери для виділення Reporting: p95 звітів або їх CPU помітно впливає на checkout p95 попри read model; traffic понад \~5x від сьогодні; команда зростає до 12-15 engineers із окремим owner. Для Payments: другий provider та PCI/security вимоги потребують окремого release lifecycle. Для Orders/Inventory: лише якщо з'явиться окремий scaling або ownership, і тоді з saga та idempotency. До цього modular monolith залишається доречним.

## 13. AI та Time to Market

### Перевірка assumption

Stakeholders припускають, що Implementation 40 → 10 днів дасть 4 місяці → 1 місяць. Арифметика показує інше: 20 + 10 + 25 + 25 + 10 = **90 днів** (≈ 3 місяці). Навіть ідеальний 4x speed-up implementation дає лише −25% end-to-end, бо implementation це 1/3 pipeline. Щоб дійти до 30 днів, усі інші stages разом мають скоротитися з 80 до 20 днів (−75%) за відсутності будь-якої інвестиції. Це закон Amdahl на рівні delivery.

### Оцінка по stages

| Stage | Зараз | Прогноз | Основні assumptions |
| --- | --: | --: | --- |
| Analysis | 20 | 15-18 | AI допомагає в research і прототипах, але залежить від доступності stakeholders; −2-5 днів |
| Implementation | 40 | 12-20 | AI реально прискорює scaffolding, tests, refactoring, docs; обмежує review (2 senior) і обов'язковий human review security/finance; для різних features різниться |
| Integration | 25 | 17-21 | потрібні explicit contracts і менші changesets (наш proposal), −4-10 днів |
| Validation | 25 | 17-20 | другий test environment, parallel testing, contract tests; зараз черга 3-5 днів; −5-10 днів |
| Release | 10 | 5-8 | частіші release windows, менший scope; −2-6 днів |
| **Разом (сума)** | **120** | **66-87** |  |

Baseline модель це сума stages, хоча частина роботи йде паралельно, а покращення частково перекриваються, тож додавати їх як незалежні не можна. Тому я не беру оптимістичну суму (59) за прогноз.

**Мій target: 2-3 місяці (точкова оцінка \~2,5 місяця, ≈ 75 днів), досяжний через 6-9 місяців підготовчої роботи.** Для першого major feature, коли prerequisites ще не готові, реалістично \~3-3,5 місяці (\~95-105 днів): виграє майже лише Implementation, і навіть він обмежений review. Stretch до 2 місяців можливий лише за значного overlap stages (analysis і implementation паралельно, feature flags, незалежні релізи).

### Чи реальний 1 місяць?

Ні, не для major feature за наявних умов. Мінімальна сума при всіх оптимістичних значеннях \~59 днів. 1 місяць реалістичний лише для **тонкого зрізу** major feature за feature flag (наприклад, без повного reporting/discount) або для малих змін. Для повного major feature потрібно змінити не лише код.

### Bottlenecks

**Architecture constraints:** зміни зачіпають 4,3 modules, отже integration і regression великі; 37% змін template вимагають регресії checkout; спільний release усіх modules, rollback всього release; shared DB без contracts.

**Process constraints (не вирішуються генерацією коду):** один test environment з чергою 3-5 днів; спільне deployment window двічі на тиждень; ручні approvals; обов'язковий human review security/finance; знання про architecture в двох senior engineers (review queue); невизначеність вимог в Analysis.

Після прискорення implementation bottleneck **переміщується** в review, integration queue і validation, а не зникає.

### Що має змінитися для місячного циклу

Boundaries із цього плану (modules per feature < 2); тестове середовище на PR або мінімум два; release у будь-який день з feature flags і менший deployment scope; розподіл review (більше reviewer-ів, архітектурні правила в CI); паралельні analysis/implementation; scope-нарізка major feature на тонкі зрізи; baseline для AI defect rate.

### Нові risks від зростання throughput

Перегрузка reviewers і ще більші PR (вже 520 LOC); більше дефектів без baseline (немає даних про defect rate AI-коду); security/finance-помилки в згенерованому коді; зростання черги integration (більше змін в тій самій черзі); ерозія знань (ніхто не розуміє згенерований код); «швидкість implementation» маскує повільний end-to-end.

### Як виміряти вплив AI

Не через LOC або кількість PR. Вимірювати: cycle time по кожному stage, час очікування review, Change Failure Rate і rework rate для AI-assisted PR проти baseline (позначати PR), escaped defects, lead time від ідеї до production, пропускну здатність (completed features per quarter), частку часу в черзі. Ввести baseline зараз, до розгортання AI на повну.

### Що змусить переглянути прогноз після першого major feature

Фактичний Implementation > 20 днів або review wait > 3 днів; Change Failure Rate AI-assisted PR вищий за baseline; integration/validation не скоротились хоча б на нижню межу (4 і 5 днів); test environment queue досі 3-5 днів; modules per feature не нижчий за 3,5. Якщо так, target зсувається до 3-4 місяців, а інвестиції переорієнтовуються на process.

## 14. План впровадження

1. **Тиждень 1-2:** hotfix R2, timeout до payment provider, baseline метрики (checkout p95/success, час stage-ів, AI-tag PR). Dependency test, поки що з allow-list поточних винятків.
2. **Тиждень 3-5:** `PaymentGateway` і прибирання `OrderRepository` з Payments; Orders бере ціни через Inventory API; contract tests.
3. **Тиждень 5-8:** in-process outbox для Notifications, шаблони як config + feature flag; другий test environment.
4. **Тиждень 8-12:** Reporting read model, оцінка метрик, рішення за тригерами з розділу 12 (виділяти Reporting чи ні).

Optional implementation: найменший крок із evidence-цінністю це hotfix R2 разом із regression тестом і dependency test для Payments. Вони можуть увійти в репозиторій без rewrite.

## 15. Припущення та неточності в матеріалах

Під час перевірки матеріалів я знайшов місця, які легко прийняти за факт. Я врахував їх так:

- **In-memory database.** Вона не має ні durability, ні справжніх транзакцій. Атомарність `reserve` зараз дає лише синхронний код в одному event loop, а standby instance не бачив би ні стану, ні idempotency-записів. Я вважаю це спрощенням навчального коду, а не знахідкою. У реальній системі durability стала б ризиком №1, а «одна межа» означала б транзакцію БД.
- **Proposal обіцяє окреме scaling Reporting і checkout.** Поки дані в одному in-memory store одного process, це неможливо. Спершу потрібна спільна БД або read store, тобто це prerequisite, а не наслідок розділення на services.
- **Scenario B (другий provider).** Provider підтримує asynchronous confirmation, а контракт не затверджено. Тому `PaymentGateway` має допускати статус `PENDING` і подію підтвердження. Checkout не може вважати, що `charge` одразу повертає `CAPTURED`.
- **Scenario F (discount за payment method).** У поточному коді спосіб оплати невідомий при `createDraft`, у Payments іде лише `paymentToken`. Тож розрахунок суми доведеться розширити, і де саме він житиме (Orders чи окремий pricing), залишається відкритим.
- **Metrics.** Дані агреговані, а кореляція не доводить причину. Частина росту Deployment Duration пояснюється test suite та ручними approvals, тобто process. У даних метрика зветься Release Frequency, у README Deployment Frequency, я читаю їх як одне.
- **AI Time to Market.** Baseline 120 днів це сума stages, хоча частина роботи паралельна. Покращення між stages частково перекриваються, тому я їх не додаю. Немає baseline defect rate для коду з AI.
- **Склад модулів.** README називає 5 модулів, `system-overview` додає Checkout як координатора. Я розглядаю Checkout як окремий orchestrator, що належить команді Orders.
- **Мої власні числа.** 1000 checkout/міс, $50/год, $75 на інцидент, ймовірності ризиків і години в change cost це мої припущення. Числа з `engineering-metrics`, `business-context` і `ai-time-to-market` взято з матеріалів.
