# MP Inventory Sync — синхронизация остатков и цен WB / Ozon

Микросервис сверки локального склада с личными кабинетами **Wildberries** и **Ozon**.
Предотвращает кассовые разрывы, пересортицу и штрафы маркетплейсов за счёт
автоматической сверки остатков и цен и (опционально) их актуализации на площадках.

Написан на **нативном PHP 8.2** без фреймворков. Запускается «из коробки» в
**demo-режиме** на встроенной заглушке — реальные ключи WB/Ozon для демонстрации не нужны.

---

## Что нового в этой версии (production-доработки)

- 🔐 **Авторизация в панели** — вход по логину/паролю, защита всех страниц и API,
  CSRF-токены на запросах, изменяющих данные.
- 🔑 **Шифрование секретов** — API-ключи маркетплейсов хранятся в БД в зашифрованном
  виде (XSalsa20-Poly1305, `sodium`). В коде и интерфейсе — расшифрованные/маскированные.
- 🖥 **Страница настроек** — добавление/обновление ключей WB и Ozon через интерфейс
  (с шифрованием при сохранении), без ручных правок БД.
- ⏱ **Запуск по расписанию (cron)** — консольная команда `bin/console.php sync`
  запускает ту же синхронизацию без веб-слоя; каждый прогон пишется в журнал `sync_runs`.
- 🧩 **Корректный маппинг WB** — остатки по баркоду, цены по `nmID` (отдельное поле
  `nm_id`); постраничная выгрузка (пагинация) для больших каталогов (WB offset, Ozon cursor).

---

## Стек

| Слой       | Технологии                                                       |
|------------|------------------------------------------------------------------|
| Backend    | Native PHP 8.2 (`strict_types`, ООП, без фреймворков)            |
| БД         | MySQL 8.0 + PDO (только подготовленные выражения)                |
| Frontend   | HTML5, Bootstrap 5 (dark), Vanilla JS (Fetch API)               |
| Безопасность | сессии, bcrypt, CSRF, sodium (шифрование секретов)            |
| Интеграции | cURL → WB Marketplace/Prices API, Ozon Seller API               |

---

## Структура

```
mp-inventory-sync/
├── bootstrap.php                 # автозагрузчик (PSR-4) + загрузка .env
├── .env / .env.example           # конфигурация
├── bin/
│   └── console.php               # CLI: key:generate, user:password, sync (cron)
├── sql/
│   └── schema.sql                # схема MySQL + demo-данные (+ sync_runs, nm_id)
├── src/
│   ├── Contracts/                # MarketplaceClientInterface
│   ├── Clients/                  # Abstract / Wildberries / Ozon / Mock
│   ├── Services/                 # InventorySyncService (ядро сверки)
│   ├── Factory/                  # MarketplaceClientFactory
│   ├── Database/                 # Database (PDO), ConnectionRepository (шифрование)
│   ├── Support/                  # Env, Security, Crypto, Auth
│   └── Exceptions/               # Api / Auth / RateLimit
├── public/
│   ├── index.php                 # дашборд (требует входа)
│   ├── login.php / logout.php    # авторизация
│   ├── settings.php              # управление ключами (шифрование при сохранении)
│   ├── api/sync.php              # REST-эндпоинт (auth + CSRF)
│   └── assets/{css,js}
└── tests/
    └── mock_smoke.php            # быстрый прогон логики без БД и сети
```

---

## Установка

1. **Схема и demo-данные:**
   ```bash
   mysql -u root -p < sql/schema.sql
   ```
2. **Конфигурация:**
   ```bash
   cp .env.example .env
   php bin/console.php key:generate     # вписать APP_KEY= в .env
   php bin/console.php user:password    # вписать AUTH_PASSWORD_HASH= в .env
   # задать AUTH_USER, проверить DB_*; APP_DEMO_MODE=true уже включён
   ```
   > В комплекте уже есть рабочий `.env` для демо (логин **admin** / пароль **admin**,
   > сгенерированный APP_KEY). **Для боевого использования смените и то, и другое.**
3. **Запуск** (корень — папка `public`):
   ```bash
   php -S localhost:8000 -t public
   ```
4. Откройте <http://localhost:8000>, войдите и нажмите **«Запустить синхронизацию»**.

> Требуются расширения PHP: `pdo_mysql`, `curl`, `sodium`, `mbstring`.

---

## Автоматическая синхронизация (cron)

Чтобы сверка шла сама, добавьте команду в планировщик. Пример — каждые 15 минут:

```cron
*/15 * * * * /usr/bin/php /путь/к/mp-inventory-sync/bin/console.php sync >> /var/log/mp-sync.log 2>&1
```

Команда не требует авторизации (запускается локально на сервере) и пишет результат
каждого прогона в таблицу `sync_runs`.

---

## Demo vs. боевой режим

```dotenv
APP_DEMO_MODE=true    # заглушка, без реальных запросов и ключей
APP_AUTO_PUSH=false   # только фиксировать расхождения (не пушить на МП)
```

Для боевого режима: `APP_DEMO_MODE=false`, затем на странице **«Настройки»** введите
ключи WB (JWT-токен + `warehouse_id`) и Ozon (`Client-Id`, `Api-Key`, `warehouse_id`).
Они сохранятся в БД в зашифрованном виде.

---

## Контракт REST-эндпоинта

`POST /api/sync.php` → `application/json`. **Требует авторизации и CSRF-токена**
(заголовок `X-CSRF-Token` или поле `csrf`). Без них — `401` / `419`.

```json
{
  "ok": true,
  "data": [ { "product_id": 1, "name": "…", "sku": "TS-001",
              "local_stock": 120, "local_price": 990.0,
              "wb_stock": 120, "wb_price": 990.0, "wb_status": "synced",
              "ozon_stock": 118, "ozon_price": 990.0, "ozon_status": "mismatch",
              "status": "mismatch" } ],
  "summary": { "total": 6, "synced": 0, "mismatch": 4, "error": 2,
               "demo_mode": true, "auto_push": false, "generated_at": "…" }
}
```

Статусы: `synced` · `mismatch` · `error` · `pending`.

---

## Безопасность

- **Авторизация** — все страницы и API закрыты сессионной аутентификацией; пароль
  хранится в виде bcrypt-хеша; CSRF-токены на изменяющих запросах.
- **Шифрование секретов** — API-ключи лежат в БД зашифрованными (`sodium`,
  формат `enc:v1:`), ключ шифрования — `APP_KEY` из `.env`.
- **SQL-инъекции** — исключены: только prepared statements PDO, `EMULATE_PREPARES=false`.
- **XSS** — весь вывод проходит через `Security::e()`, на фронтенде — `textContent`.
- **Маскирование** — ключи в интерфейсе показываются замаскированными.
- **Валидация входа** — строгая фильтрация параметров, методы по белому списку.

> ⚠️ Для боевого использования: смените демо-логин/пароль и `APP_KEY`, работайте
> по HTTPS (раскомментируйте `secure` в cookie сессии), ограничьте доступ к панели.

---

## Проверка

```bash
php tests/mock_smoke.php   # логика клиента + Security + Crypto + Auth, без БД/сети
```

---

## Примечания по API маркетплейсов (актуально на 2025–2026)

- **WB остатки:** `POST/PUT https://marketplace-api.wildberries.ru/api/v3/stocks/{warehouseId}` — по баркоду.
- **WB цены:** список `POST https://discounts-prices-api.wildberries.ru/api/v2/list/goods/filter`,
  запись `POST .../api/v2/upload/task` — по `nmID`.
- **Ozon остатки:** чтение `POST /v4/product/info/stocks`, запись `/v2/products/stocks`.
- **Ozon цены:** чтение `POST /v5/product/info/prices`, запись `/v1/product/import/prices`.

Боевой режим перед использованием на реальном кабинете обязательно протестируйте на
тестовых данных площадки: форматы ответов API периодически меняются.

---

## Лицензия

Распространяется по лицензии MIT (см. `LICENSE`) — без гарантий, ответственность за
использование несёт пользователь. См. также `DISCLAIMER.md`.
