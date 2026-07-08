# Book Management System - Full Specification

## Overview

Библиотечная система управляет каталогом книг, поддерживает аренду и покупку как физических, так и цифровых экземпляров, обрабатывает платежи, контролирует штрафы за просрочку и обеспечивает защиту контента от пиратства.

Архитектура: **микросервисная** с Event-Driven взаимодействием через RabbitMQ.

---

## 1. Микросервисная архитектура

### 1.1 Разделение сервисов и баз данных

Каждый микросервис имеет **собственную независимую базу данных** (Database-per-Service).

#### Book Service (основной сервис)
Владеет таблицами:
- `books`, `book_pdfs`, `pdf_copies`
- `rentals`, `purchases`
- `fines`, `user_debt`
- `reviews`, `bookmarks`, `reading_progress`, `wishlist`, `reservations`
- `promo_codes`, `promo_code_usages`
- `authors`, `categories`, `book_authors`, `book_categories`
- `reading_sessions` (heartbeat)

#### Payment Service (отдельный микросервис)
Владеет таблицами:
- `payment_intents` (счета, привязанные к rental_id / purchase_id)
- `transactions` (id, amount, currency, status, gateway_response, payment_method)
- `payment_methods` (привязанные карты, токены)
- `refunds` (возвраты средств)

#### Notification Service (отдельный микросервис)
Владеет таблицами:
- `user_notification_preferences`
- `notification_templates`
- `notification_history`
- `message_queue`

### 1.2 Event-Driven взаимодействие

Сервисы общаются **только через асинхронные события** (RabbitMQ). Прямые REST/gRPC вызовы минимизированы.

**Пример флоу аренды (CARD):**
```
1. Book Service: Создаёт rental (PENDING_PAYMENT)
   → Публикует: RentCreated {rental_id, user_id, amount, deposit_amount}

2. Payment Service: Слушает RentCreated
   → Создаёт payment_intent, идёт в банк, делает pre-auth
   → Публикует: PaymentAuthorized {rental_id, payment_intent_id, status}

3. Book Service: Слушает PaymentAuthorized
   → Находит rental по rental_id
   → Меняет статус на ACTIVE
   → Уменьшает digital_licenses / physical_inventory
   → Публикует: RentActivated {rental_id, user_id, book_id}

4. Notification Service: Слушает RentActivated
   → Смотрит настройки юзера в своей БД
   → Отправляет Email/Push
```

**Преимущества:**
- Если Notification Service упадёт — книги всё равно выдаются
- Если Payment Service долго отвечает — Book Service не блокируется
- Каждый сервис масштабируется независимо

### 1.3 Связь между сервисами

| Book Service поле | Payment Service поле | Описание |
|-------------------|---------------------|----------|
| `payment_intent_id` (UUID) | `payment_intents.id` | Ссылка на счёт в Payment Service |
| — | `payment_intents.rental_id` | Обратная ссылка на аренду |
| — | `payment_intents.purchase_id` | Обратная ссылка на покупку |

**Важно:** Book Service **не хранит** `transaction_id`, `preauth_transaction_id`. Все транзакции — в Payment Service.

---

## 2. Разделение физических и цифровых книг

### 2.1 Ключевое различие

| Аспект | Физическая книга | Цифровая книга |
|--------|------------------|----------------|
| Наличие | Ограничено `physical_inventory` | Ограничено `digital_licenses` или `unlimited` |
| Аренда | Уменьшает `physical_inventory` на 1 | Уменьшает `digital_licenses` на 1 |
| Возврат | Требует подтверждения админом (сканирование) | Истекает автоматически по таймеру |
| Покупка | Уменьшает `physical_inventory` на 1 | Не уменьшает лицензии |
| Скачивание PDF | Нет | Да (при покупке) |

### 2.2 Типы аренды

| Тип | Описание | Возврат |
|-----|----------|---------|
| **Digital Rental** | Чтение на сайте | Автоматический по таймеру |
| **Physical Rental** | Бумажная книга на руки | Админ подтверждает сканированием |

---

## 3. Структура базы данных — Book Service

### 3.1 Таблица `books`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| title | VARCHAR(255) | INDEX | Название книги |
| description | TEXT | | Описание |
| isbn | VARCHAR(13) | UNIQUE | Уникальный ISBN |
| has_physical | BOOLEAN | | Доступна ли физическая версия |
| has_digital | BOOLEAN | | Доступна ли цифровая версия |
| physical_inventory | INTEGER | | Количество бумажных копий (0 = unavailable) |
| digital_licenses | INTEGER | | Цифровые лицензии (-1 = безлимит) |
| price_purchase | DECIMAL(10,2) | | Цена покупки (в тенге) |
| price_rental | DECIMAL(10,2) | | Цена аренды за период (в тенге) |
| deposit_amount | DECIMAL(10,2) | | Залог при аренде (в тенге) |
| is_available_for_rent | BOOLEAN | | Ручной тумблер доступности аренды |
| is_available_for_purchase | BOOLEAN | | Ручной тумблер доступности покупки |
| publication_year | INTEGER | INDEX | Год издания |
| cover_image_url | VARCHAR(500) | | URL обложки |
| total_rentals_count | INTEGER | | Счётчик аренд для аналитики |
| total_purchases_count | INTEGER | | Счётчик покупок для аналитики |
| is_active | BOOLEAN | INDEX | Soft delete флаг |
| version | INTEGER | | Optimistic locking (защита от race condition) |
| created_at | TIMESTAMP | | Дата создания |
| updated_at | TIMESTAMP | | Дата обновления |

### 3.2 Таблица `rentals`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| book_id | UUID | INDEX, FK → books | ID книги |
| rental_type | ENUM | INDEX | DIGITAL / PHYSICAL |
| payment_method | ENUM | | CASH / CARD |
| status | ENUM | INDEX | PENDING_PAYMENT / PENDING_CONFIRMATION / ACTIVE / OVERDUE / RETURNED / EXPIRED / LOST / CANCELLED |
| start_date | DATE | | Дата начала аренды |
| end_date | DATE | INDEX | Дата окончания аренды |
| actual_return_date | DATE | | Дата фактического возврата (NULL) |
| rental_price | DECIMAL(10,2) | | Цена аренды (в тенге) |
| deposit_amount | DECIMAL(10,2) | | Залог (в тенге) |
| payment_intent_id | UUID | FK → Payment Service | Ссылка на счёт в Payment Service |
| promo_code_id | UUID | FK → promo_codes | Использованный промокод (NULL) |
| original_amount | DECIMAL(10,2) | | Цена до скидок (в тенге) |
| discount_amount | DECIMAL(10,2) | | Сумма скидки (в тенге) |
| final_amount | DECIMAL(10,2) | | Итог к оплате (в тенге) |
| receipt_code | VARCHAR(64) | UNIQUE | Код чека/штрих-кода (для CASH) |
| is_receipt_confirmed | BOOLEAN | | Админ подтвердил чек (для CASH) |
| fine_amount | DECIMAL(10,2) | | Начисленный штраф (в тенге) |
| fine_status | ENUM | | NONE / PENDING / PAID / WAIVED |
| fine_payment_method | ENUM | | CASH / CARD / INTERNAL_DEBT |
| created_at | TIMESTAMP | | Дата создания |
| updated_at | TIMESTAMP | | Дата обновления |

### 3.3 Таблица `purchases`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| book_id | UUID | INDEX, FK → books | ID книги |
| purchase_type | ENUM | INDEX | PHYSICAL / DIGITAL |
| payment_method | ENUM | | CASH / CARD (для DIGITAL — только CARD) |
| status | ENUM | INDEX | PENDING_PAYMENT / PENDING_CONFIRMATION / COMPLETED / CANCELLED |
| purchase_date | TIMESTAMP | | Дата покупки |
| purchase_price | DECIMAL(10,2) | | Цена покупки (в тенге) |
| payment_intent_id | UUID | FK → Payment Service | Ссылка на счёт в Payment Service |
| promo_code_id | UUID | FK → promo_codes | Использованный промокод (NULL) |
| original_amount | DECIMAL(10,2) | | Цена до скидок (в тенге) |
| discount_amount | DECIMAL(10,2) | | Сумма скидки (в тенге) |
| final_amount | DECIMAL(10,2) | | Итог к оплате (в тенге) |
| receipt_code | VARCHAR(64) | UNIQUE | Код чека (для CASH, PHYSICAL only) |
| is_receipt_confirmed | BOOLEAN | | Админ подтвердил оплату (для CASH) |
| created_at | TIMESTAMP | | Дата создания |
| updated_at | TIMESTAMP | | Дата обновления |

### 3.4 Таблица `fines`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| rental_id | UUID | INDEX, FK → rentals | ID аренды |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| amount | DECIMAL(10,2) | | Сумма штрафа (в тенге) |
| status | ENUM | INDEX | PENDING / PAID / WAIVED |
| payment_method | ENUM | | CASH / CARD / INTERNAL_DEBT |
| reason | VARCHAR(500) | | Причина штрафа |
| days_overdue | INTEGER | | Количество дней просрочки |
| calculated_at | TIMESTAMP | | Дата расчёта штрафа |
| paid_at | TIMESTAMP | | Дата оплаты |
| created_at | TIMESTAMP | | Дата создания |
| updated_at | TIMESTAMP | | Дата обновления |

### 3.5 Таблица `user_debt`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| user_id | UUID | UNIQUE, FK → users | ID пользователя |
| total_debt | DECIMAL(10,2) | | Общая сумма задолженности (в тенге) |
| is_blocked | BOOLEAN | | Заблокирован от новых аренд |
| last_updated | TIMESTAMP | | Дата последнего обновления |

### 3.6 Таблица `pdf_copies`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| book_id | UUID | INDEX, FK → books | ID книги |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| purchase_id | UUID | FK → purchases | ID покупки (NULL если аренда) |
| rental_id | UUID | FK → rentals | ID аренды (NULL если покупка) |
| anti_piracy_code | VARCHAR(128) | UNIQUE | Уникальный код |
| watermarked_file_url | VARCHAR(500) | | URL в S3/MinIO (сгенерированный PDF) |
| pdf_file_hash | VARCHAR(64) | | SHA-256 хеш сгенерированного PDF |
| visible_watermark | TEXT | | Текст видимого водяного знака |
| invisible_metadata | TEXT | | Скрытые метаданные |
| download_count | INTEGER | | Количество скачиваний |
| max_downloads | INTEGER | | Максимум скачиваний (1 = покупка, 0 = аренда) |
| status | ENUM | INDEX | ACTIVE / REVOKED / FLAGGED |
| flagged_reason | VARCHAR(500) | | Причина флага |
| created_at | TIMESTAMP | | Дата создания |

### 3.7 Таблица `book_pdfs`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| book_id | UUID | UNIQUE, FK → books | ID книги |
| original_file_url | VARCHAR(500) | | URL оригинального PDF в S3/MinIO |
| file_size | BIGINT | | Размер файла в байтах |
| uploaded_by | UUID | FK → users | Админ, загрузивший файл |
| uploaded_at | TIMESTAMP | | Дата загрузки |

### 3.8 Таблица `reviews`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| book_id | UUID | INDEX, FK → books | ID книги |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| rating | INTEGER | | Оценка 1-5 |
| review_text | TEXT | | Текст рецензии |
| status | ENUM | INDEX | PENDING / APPROVED / REJECTED |
| created_at | TIMESTAMP | INDEX | Дата создания |
| updated_at | TIMESTAMP | | Дата обновления |

### 3.9 Таблица `bookmarks`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| book_id | UUID | INDEX, FK → books | ID книги |
| page_number | INTEGER | | Номер страницы |
| note | TEXT | | Заметка к закладке |
| created_at | TIMESTAMP | | Дата создания |

### 3.10 Таблица `reading_progress`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| book_id | UUID | INDEX, FK → books | ID книги |
| last_page | INTEGER | | Последняя прочитанная страница |
| total_pages | INTEGER | | Всего страниц в книге |
| last_read_at | TIMESTAMP | | Дата последнего чтения |

### 3.11 Таблица `wishlist`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| book_id | UUID | INDEX, FK → books | ID книги |
| notification_sent | BOOLEAN | | Уведомление уже отправлено |
| created_at | TIMESTAMP | | Дата добавления |

### 3.12 Таблица `reservations`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| book_id | UUID | INDEX, FK → books | ID книги |
| status | ENUM | INDEX | WAITING / READY / EXPIRED / CANCELLED |
| reserved_at | TIMESTAMP | | Дата бронирования |
| ready_at | TIMESTAMP | | Дата когда книга готова к выдаче |
| expires_at | TIMESTAMP | INDEX | Срок действия (24 часа после ready_at) |
| notification_sent | BOOLEAN | | Уведомление отправлено |

### 3.13 Таблица `promo_codes`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| code | VARCHAR(50) | UNIQUE | Уникальный промокод |
| discount_type | ENUM | | PERCENT / FIXED |
| discount_value | DECIMAL(10,2) | | Значение скидки |
| applicable_to | ENUM | | RENTAL / PURCHASE / BOTH |
| max_uses | INTEGER | | Максимум использований (NULL = безлимит) |
| used_count | INTEGER | | Количество использований |
| valid_from | TIMESTAMP | | Дата начала действия |
| valid_until | TIMESTAMP | | Дата окончания действия |
| is_active | BOOLEAN | INDEX | Активен ли промокод |

### 3.14 Таблица `reading_sessions` (Heartbeat)

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| book_id | UUID | INDEX, FK → books | ID книги |
| rental_id | UUID | FK → rentals | ID аренды |
| started_at | TIMESTAMP | | Начало сессии чтения |
| last_heartbeat | TIMESTAMP | INDEX | Последний сигнал активности |
| last_page | INTEGER | | Текущая страница |
| is_active | BOOLEAN | INDEX | Активна ли сессия |

### 3.15 Таблица `authors`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| full_name | VARCHAR(255) | INDEX | Полное имя автора |
| bio | TEXT | | Биография |
| photo_url | VARCHAR(500) | | URL фото автора |
| created_at | TIMESTAMP | | Дата создания |
| updated_at | TIMESTAMP | | Дата обновления |

### 3.16 Таблица `categories`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| name | VARCHAR(100) | UNIQUE | Название категории |
| parent_id | UUID | FK → categories, NULLABLE | Родительская категория (иерархия) |
| description | TEXT | | Описание категории |
| created_at | TIMESTAMP | | Дата создания |

### 3.17 Таблица `book_authors` (Many-to-Many)

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| book_id | UUID | PK, FK → books | ID книги |
| author_id | UUID | PK, FK → authors | ID автора |
| author_role | ENUM | | MAIN_AUTHOR / CO_AUTHOR |
| author_order | INTEGER | | Порядок отображения |

### 3.18 Таблица `book_categories` (Many-to-Many)

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| book_id | UUID | PK, FK → books | ID книги |
| category_id | UUID | PK, FK → categories | ID категории |

### 3.19 Таблица `promo_code_usages`

| Поле | Тип | Индекс | Описание |
|------|-----|--------|----------|
| id | UUID | PK | Первичный ключ |
| promo_code_id | UUID | INDEX, FK → promo_codes | ID промокода |
| user_id | UUID | INDEX, FK → users | ID пользователя |
| rental_id | UUID | FK → rentals, NULLABLE | ID аренды (если применён к аренде) |
| purchase_id | UUID | FK → purchases, NULLABLE | ID покупки (если применён к покупке) |
| discount_amount | DECIMAL(10,2) | | Сумма скидки (в тенге) |
| applied_at | TIMESTAMP | | Дата применения |

---

## 4. Бизнес-логика

### 4.1 Каталог книг

#### 4.1.1 Просмотр книг
- Все пользователи (авторизованные и нет) могут просматривать каталог
- Пагинация, сортировка по названию, цене, году издания, популярности
- Поиск по названию, автору, ISBN, категории, жанрам, тегам
- Фильтры:
  - «Только доступные сейчас» (physical_inventory > 0 или digital_licenses > 0/-1)
  - «Только электронные» (has_digital = true)
  - «Только бумажные» (has_physical = true)
  - По диапазону цен
  - По категориям/жанрам

#### 4.1.2 Детальная страница книги
Отображает:
- Название, описание, обложка
- Автор(ы) с ролями (основной, соавтор)
- Категории/жанры
- Год издания, ISBN
- Цены: покупка (тенге), аренда (тенге/период)
- Статус доступности:
  - Физическая: `Available (N copies)` / `Unavailable` / `Available for reservation`
  - Цифровая: `Available` / `Unavailable (no licenses)` / `Unlimited`
- Рейтинг (средний из отзывов)
- Количество аренд и покупок
- Кнопки: «Арендовать», «Купить», «В wishlist», «Зарезервировать»

#### 4.1.3 Создание книг (Admin)
- Заполнение всех метаданных
- Загрузка PDF файла
- Установка цен (покупка, аренда, залог)
- Установка количества физических копий (`physical_inventory`)
- Установка цифровых лицензий (`digital_licenses`: число или -1 для безлимита)
- Переключатели доступности (`is_available_for_rent`, `is_available_for_purchase`)
- Привязка авторов (из существующих или создание новых)
- Привязка категорий (из существующих или создание новых)

#### 4.1.4 Редактирование книг (Admin)
- Полное и частичное обновление
- Обновление цен отдельно
- Изменение количества копий (инвентаризация)
- Переключение доступности
- Замена PDF файла

#### 4.1.5 Удаление книг (Admin)
- Soft delete (isActive = false) — книга скрывается из каталога
- Hard delete — полное удаление (только если нет активных аренд/покупок)

### 4.2 Аренда книг

#### 4.2.1 Цифровая аренда (Digital Rental)
1. Пользователь выбирает «Арендовать цифровую версию»
2. Система проверяет:
   - `has_digital = true`
   - `is_available_for_rent = true`
   - `digital_licenses > 0` или `digital_licenses = -1`
   - У пользователя нет неоплаченных штрафов (`user_debt.total_debt = 0`)
   - Пользователь не заблокирован (`user_debt.is_blocked = false`)
   - Нет активной аренды этой же книги у пользователя
3. **Race Condition защита:** `SELECT ... FOR UPDATE` на запись книги + проверка `version` (optimistic locking)
4. Пользователь выбирает способ оплаты:
   - **CARD**: Book Service создаёт rental (PENDING_PAYMENT) → публикует событие `RentCreated` → Payment Service делает pre-auth → публикует `PaymentAuthorized` → Book Service активирует аренду
   - **CASH**: Book Service создаёт rental (PENDING_CONFIRMATION) → генерирует `receipt_code` → отправляет событие в очередь → Notification Service шлёт чек на email
5. При CARD (после `PaymentAuthorized`):
   - `digital_licenses = digital_licenses - 1` (если не -1)
   - Статус аренды → `ACTIVE`
   - Пользователь получает доступ к онлайн-чтению
   - Создаётся `reading_session` с heartbeat
   - Доступ истекает автоматически по `end_date`
   - `digital_licenses = digital_licenses + 1` (если не -1)
6. При CASH (после подтверждения админом):
   - Админ сканирует штрих-код → подтверждает аренду
   - `digital_licenses = digital_licenses - 1` (если не -1)
   - Статус → `ACTIVE`
   - Пользователь получает доступ к онлайн-чтению
   - Доступ истекает автоматически по `end_date`
   - `digital_licenses = digital_licenses + 1` (если не -1)

#### 4.2.2 Физическая аренда (Physical Rental)
1. Пользователь выбирает «Арендовать бумажную книгу»
2. Система проверяет:
   - `has_physical = true`
   - `is_available_for_rent = true`
   - `physical_inventory > 0`
   - У пользователя нет неоплаченных штрафов
   - Пользователь не заблокирован
   - Нет активной аренды этой же книги у пользователя
3. **Race Condition защита:** `SELECT ... FOR UPDATE` на запись книги + optimistic locking
4. Пользователь выбирает способ оплаты:
   - **CARD**: Book Service создаёт rental (PENDING_PAYMENT) → публикует `RentCreated` → Payment Service делает pre-auth (`price_rental + deposit_amount`) → публикует `PaymentAuthorized` → Book Service активирует аренду
   - **CASH**: Book Service создаёт rental (PENDING_CONFIRMATION) → генерирует чек → Notification Service шлёт на email
5. При CARD (после `PaymentAuthorized`):
   - `physical_inventory = physical_inventory - 1`
   - Статус аренды → `ACTIVE`
   - Пользователь забирает книгу из библиотеки
   - **Возврат требует подтверждения админом** (сканирование штрих-кода)
6. При CASH (после подтверждения админом):
   - Админ сканирует штрих-код → подтверждает выдачу
   - `physical_inventory = physical_inventory - 1`
   - Статус → `ACTIVE`
   - **Возврат требует подтверждения админом**

#### 4.2.3 Период аренды
- Стандартный период: 30 дней (настраивается админом)
- При создании аренды: `end_date = start_date + rental_period`
- Цифровая аренда: доступ прекращается автоматически в `end_date 00:00`
- Физическая аренда: требует возврата и подтверждения админом

#### 4.2.4 Heartbeat для цифровых сессий
- При открытии книги в онлайн-ридере создаётся `reading_session`
- Фронтенд отправляет heartbeat каждые 5 минут
- Если heartbeat не получен 15 минут → сессия закрывается
- При закрытии сессии: `is_active = false`
- Если все сессии для аренды закрыты и аренда ещё активна — лицензия **не** освобождается (аренда = время, а не сессия)
- Heartbeat нужен для аналитики и корректного сохранения прогресса

### 4.3 Покупка книг

#### 4.3.1 Покупка цифровой версии
1. Пользователь выбирает «Купить цифровую версию»
2. Система проверяет:
   - `has_digital = true`
   - `is_available_for_purchase = true`
   - У пользователя нет неоплаченных штрафов
   - У пользователя ещё нет покупки этой же цифровой книги
3. **Оплата только через карту (CARD)** — наличные не поддерживаются для цифровых покупок
4. Процесс оплаты:
   - Book Service создаёт purchase (PENDING_PAYMENT) → публикует `PurchaseCreated`
   - Payment Service слушает → создаёт invoice → списывает `price_purchase` → публикует `PaymentCompleted`
   - Book Service слушает → статус → `COMPLETED`
   - Генерируется `PdfCopy` с анти-пиратским кодом
   - PDF сохраняется в S3/MinIO, `watermarked_file_url` записывается в `pdf_copies`
   - Пользователь получает:
     - Постоянный доступ к онлайн-чтению
     - Возможность скачать PDF (максимум 1 раз)

#### 4.3.2 Покупка физической версии
1. Пользователь выбирает «Купить бумажную книгу»
2. Система проверяет:
   - `has_physical = true`
   - `is_available_for_purchase = true`
   - `physical_inventory > 0`
   - **Race Condition защита:** `SELECT ... FOR UPDATE` + optimistic locking
3. При CARD:
   - Book Service создаёт purchase (PENDING_PAYMENT) → публикует `PurchaseCreated`
   - Payment Service списывает `price_purchase` → публикует `PaymentCompleted`
   - Book Service: `physical_inventory = physical_inventory - 1`, статус → `COMPLETED`
   - Пользователь забирает книгу из библиотеки
4. При CASH:
   - Book Service создаёт purchase (PENDING_CONFIRMATION) → генерирует чек
   - Notification Service шлёт чек на email
   - Админ подтверждает оплату → статус → `COMPLETED`
   - `physical_inventory = physical_inventory - 1`
   - Пользователь забирает книгу

### 4.4 Возврат книг

#### 4.4.1 Цифровая аренда
- Возврат **автоматический** по истечении `end_date`
- Scheduled job проверяет просроченные аренды ежедневно
- При `current_date > end_date`:
  - Доступ к чтению отзывается
  - `reading_session.is_active = false` для всех активных сессий
  - `digital_licenses = digital_licenses + 1` (если не -1)
  - Статус → `EXPIRED`
  - Если есть просрочка → рассчитывается штраф

#### 4.4.2 Физическая аренда — возврат и депозит

**Флоу возврата:**
1. Пользователь возвращает книгу в библиотеку
2. Админ сканирует штрих-код (из чека или из системы)
3. Система находит аренду по коду
4. Админ подтверждает возврат:
   - `physical_inventory = physical_inventory + 1`
   - `actual_return_date = current_date`
   - Статус → `RETURNED`
   - Рассчитывается штраф (если есть просрочка)

**Флоу возврата депозита:**

| Сценарий | Действие с депозитом |
|----------|---------------------|
| **Возврат без просрочки** | Холдирование депозита **Release** (полное снятие блокировки, деньги возвращаются на карту) |
| **Возврат с просрочкой, штраф < депозит** | Холдирование депозита **Capture** на сумму штрафа → остаток **Release** |
| **Возврат с просрочкой, штраф >= депозит** | Холдирование депозита **Capture** полностью → разница начисляется в `user_debt` |
| **CASH оплата** | Депозит не холдировался → при штрафе пользователь оплачивает наличными, админ подтверждает → если не оплачивает → штраф в `user_debt` |

**Событийный флоу (CARD):**
```
1. Admin подтверждает возврат → Book Service публикует: ReturnConfirmed {rental_id, actual_return_date}
2. Book Service рассчитывает штраф
3. Если штраф > 0:
   → Book Service публикует: FineChargeRequested {rental_id, payment_intent_id, fine_amount, deposit_amount}
   → Payment Service слушает:
     - Если fine_amount <= deposit_amount: Capture fine_amount, Release остаток
     - Если fine_amount > deposit_amount: Capture deposit_amount, разницу → Book Service начисляет в user_debt
   → Payment Service публикует: FineChargeProcessed {rental_id, captured_amount, released_amount, debt_amount}
   → Book Service слушает → обновляет fine_status, user_debt
4. Если штраф = 0:
   → Book Service публикует: DepositReleaseRequested {payment_intent_id, deposit_amount}
   → Payment Service слушает → Release всего холдирования
   → Payment Service публикует: DepositReleased {payment_intent_id}
```

#### 4.4.3 Бронирование физической книги
- Если `physical_inventory = 0`, пользователь может зарезервировать книгу
- Создаётся запись в `reservations` со статусом `WAITING`
- Когда книга возвращается (админ подтверждает возврат):
  - Система проверяет очередь бронирований
  - Первому в очереди: статус → `READY`
  - `ready_at = current_time`
  - `expires_at = ready_at + 24 часа`
  - Отправляется уведомление: «Книга ждёт вас, заберите за 24 часа»
- Если пользователь не забрал за 24 часа:
  - Статус → `EXPIRED`
  - Уведомление следующему в очереди
- Когда пользователь приходит за книгой:
  - Админ подтверждает выдачу
  - `physical_inventory = physical_inventory - 1`
  - Создаётся аренда

### 4.5 Система штрафов

#### 4.5.1 Алгоритм расчёта

```
Grace Period: 5 дней (штраф = 0, только напоминания)

День 6+:
  daily_fine = price_purchase * 0.02  (2% от стоимости покупки в день)

  Если days_overdue <= 10:
    multiplier = 1.0
  Если days_overdue <= 20:
    multiplier = 1.5
  Если days_overdue > 20:
    multiplier = 2.0

  total_fine = daily_fine * (days_overdue - 5) * multiplier

Max Cap: total_fine НЕ МОЖЕТ превышать price_purchase (100% стоимости книги)

Если total_fine >= price_purchase:
  - Книга помечается как LOST
  - Списывается полная стоимость книги (price_purchase)
  - physical_inventory = physical_inventory + 1
  - digital_licenses = digital_licenses + 1 (если не -1)
  - Аренда закрывается со статусом LOST
```

#### 4.5.2 Примеры расчёта (в тенге)

| Цена покупки | Дней просрочки | Расчёт | Итого штраф |
|--------------|----------------|--------|-------------|
| 5 000 ₸ | 3 | Grace period | 0 ₸ |
| 5 000 ₸ | 5 | Grace period | 0 ₸ |
| 5 000 ₸ | 6 | 100 × 1 × 1.0 | 100 ₸ |
| 5 000 ₸ | 10 | 100 × 5 × 1.0 | 500 ₸ |
| 5 000 ₸ | 15 | 100 × 10 × 1.5 | 1 500 ₸ |
| 5 000 ₸ | 20 | 100 × 15 × 1.5 | 2 250 ₸ |
| 5 000 ₸ | 25 | 100 × 20 × 2.0 | 4 000 ₸ |
| 5 000 ₸ | 30 | Cap reached | 5 000 ₸ (максимум) |

#### 4.5.3 Оплата штрафов

**При возврате (физическая книга):**
1. Админ подтверждает возврат
2. Система рассчитывает штраф
3. Если штраф = 0:
   - Холдирование снимается полностью (Release)
   - Депозит возвращается
4. Если штраф > 0:
   - **CARD**: Штраф списывается из заблокированных средств (pre-authorization capture)
     - Если холдирования достаточно: списывается штраф, остаток возвращается
     - Если холдирования недостаточно: разница начисляется в `user_debt`
   - **CASH**:
     - Пользователь оплачивает штраф наличными
     - Админ подтверждает оплату в панели
     - Если пользователь не оплачивает: штраф начисляется в `user_debt`

**Внутренний долг (`user_debt`):**
- Если `total_debt > 0`:
  - Пользователь не может арендовать новые книги
  - Пользователь не может скачивать PDF
  - Пользователь видит баланс задолженности в личном кабинете
- Пользователь может погасить долг:
  - Оплатить онлайн (CARD) из личного кабинета → Book Service публикует `DebtPaymentRequested` → Payment Service обрабатывает → публикует `DebtPaymentCompleted` → Book Service обновляет `user_debt`
  - Оплатить наличными в библиотеке (админ подтверждает)
- При `total_debt = 0`: блокировка снимается автоматически

#### 4.5.4 Уведомления о просрочке

| День | Действие |
|------|----------|
| 1-5 | Ежедневное email-напоминание: «Пожалуйста, верните книгу. Штраф пока не начислен.» |
| 6 | Email: «Начислен штраф: X ₸. Верните книгу как можно скорее.» |
| 10 | Email: «Штраф увеличился до X ₸.» |
| 15 | Email: «Штраф: X ₸. Ваш аккаунт будет заблокирован при достижении лимита.» |
| 20 | Email: «Штраф: X ₸. Критическая просрочка.» |
| 25+ | Email: «Книга будет помечена как утеряна. Штраф достигнет стоимости книги.» |
| Cap reached | Email: «Книга помечена как утеряна. Списана стоимость: X ₸.» |

### 4.6 Анти-пиратская система PDF

#### 4.6.1 Генерация защищённой копии
При скачивании PDF (только при покупке):
1. Проверяется `download_count < max_downloads`
2. Если `watermarked_file_url` уже существует в `pdf_copies` → отдаётся готовый файл из S3
3. Если файла нет → генерируется персонализированная копия:
   - Генерируется уникальный `anti_piracy_code`: `LIB-{userId}-{bookId}-{timestamp}-{randomHash}`
   - **Видимый водяной знак** на каждой странице: email, имя, anti-piracy code
   - **Невидимые метаданные** в свойствах PDF: anti-piracy code, timestamp, user ID
   - **Скрытый текстовый слой**: anti-piracy code повторяется через каждые N страниц
4. Сгенерированный PDF загружается в S3/MinIO
5. `watermarked_file_url` записывается в `pdf_copies`
6. `pdf_file_hash` (SHA-256) вычисляется и сохраняется
7. `download_count = download_count + 1`

#### 4.6.2 Ограничения скачивания
- **Аренда**: скачивание запрещено (`max_downloads = 0`), только онлайн-чтение
- **Покупка**: максимум 1 скачивание (`max_downloads = 1`)
- Готовый PDF кешируется в S3 — повторное скачивание не требует регенерации

#### 4.6.3 Идентификация пиратской копии
1. Админ загружает найденный пиратский PDF
2. Система извлекает anti-piracy code из метаданных/скрытого слоя
3. Или вычисляет hash и сравнивает с `pdf_file_hash`
4. Находит запись в `pdf_copies` по коду
5. Определяется пользователь (`user_id`)
6. Запись получает статус `FLAGGED` с причиной
7. Админ видит информацию о пользователе

### 4.7 Отзывы и рейтинги

#### 4.7.1 Создание отзыва
- Только пользователи с историей аренды/покупки данной книги могут оставить отзыв
- Оценка: 1-5 звёзд
- Текст рецензии (опционально)
- Статус: `PENDING` (ожидает модерации)

#### 4.7.2 Модерация (Admin)
- Админ видит все отзывы со статусом `PENDING`
- Может: `APPROVED` (публикуется) или `REJECTED` (удаляется)

#### 4.7.3 Отображение
- Средний рейтинг на странице книги
- Список утверждённых отзывов

### 4.8 Закладки и прогресс чтения

#### 4.8.1 Прогресс чтения
- При чтении онлайн система сохраняет `last_page`
- При повторном открытии: переход на последнюю страницу
- Прогресс: «Страница X из Y»

#### 4.8.2 Закладки
- Пользователь создаёт закладку на любой странице
- Опциональная заметка
- Список закладок в личном кабинете
- Быстрый переход из ридера

### 4.9 Wishlist (Список желаний)

- Пользователь добавляет книгу в wishlist
- Если книга становится доступной (`physical_inventory > 0` или `digital_licenses > 0`):
  - Отправляется уведомление всем пользователям в wishlist
  - `notification_sent = true`
- Пользователь может удалить книгу из wishlist

### 4.10 Промокоды

- Админ создаёт промокоды:
  - Тип скидки: процент (%) или фиксированная сумма (тенге)
  - Применение: аренда, покупка, или оба
  - Лимит использований
  - Срок действия
- Пользователь вводит промокод при оформлении аренды/покупки
- Система валидирует:
  - Промокод существует и активен
  - Срок действия не истёк
  - Лимит использований не превышен
  - Промокод применим к данному типу операции
- Скидка применяется к итоговой сумме
- Запись в `promo_code_usages` для аналитики
- `used_count` в `promo_codes` инкрементируется

### 4.11 Удаление пользователей (GDPR / Anonymization)

При запросе на удаление аккаунта:
- **Hard Delete невозможен** — на пользователе висят rentals, fines, purchases
- **Anonymization:**
  - `users.email` → `deleted_{uuid}@deleted.local`
  - `users.username` → `deleted_user_{uuid}`
  - `users.first_name`, `users.last_name`, `users.phone` → очищаются
  - `users.password_hash` → заменяется на случайный хеш
  - Все персональные данные удаляются
- Записи в `rentals`, `fines`, `purchases` **сохраняются** для финансовой отчётности
- `user_debt` сохраняется до полного погашения
- Анонимизация выполняется асинхронно через событие `UserDeletionRequested`

---

## 5. Личный кабинет пользователя

Описание личного кабинета вынесено в отдельный файл: [User.md](./User.md)

### Краткое содержание
- Профиль, Мои книги, История чтений
- Закладки, Штрафы и задолженность
- Wishlist, Бронирования
- Настройки уведомлений, Отзывы
- Онлайн-ридер с heartbeat

---

## 6. Админ-панель

Описание админ-панели вынесено в отдельный файл: [Admin.md](../../Library_Documentation/Library_Documentation/Admin.md)

### Краткое содержание
- Управление книгами, пользователями
- Управление арендами, возвратами, покупками
- Управление штрафами, контентом
- Аналитика и отчёты
- Инвентаризация

---

## 7. Технические схемы

### 7.1 Схема аренды цифровой книги (CARD) — Event-Driven

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  User    │────>│ Book Service │────>│  RabbitMQ    │────>│Payment Service│
│          │     │              │     │  Exchange    │     │              │
│          │<────│              │<────│              │<────│              │
└──────────┘     │              │     │              │     │              │
                 │ 1. Проверка доступности           │     │              │
                 │ 2. Проверка debt пользователя     │     │              │
                 │ 3. SELECT ... FOR UPDATE (book)   │     │              │
                 │ 4. Создание Rental (PENDING)      │     │              │
                 │ 5. Публикация: RentCreated        │────>│              │
                 │              │                    │     │ 6. Создание   │
                 │              │                    │     │    invoice    │
                 │              │                    │     │ 7. Pre-auth   │
                 │              │                    │     │    в банке    │
                 │              │                    │     │ 8. Публикация:│
                 │              │                    │<────│ PaymentAuth   │
                 │ 9. digital_licenses - 1           │     │              │
                 │ 10. Status → ACTIVE               │     │              │
                 │ 11. Создание reading_session      │     │              │
                 │ 12. Публикация: RentActivated     │────>│              │
                 ▼              ▼                    ▼     ▼              ▼
          ┌─────────────────────────────────────────────────────────────┐
          │              Book Service Database                          │
          │  - books (digital_licenses - 1, version + 1)               │
          │  - rentals (status = ACTIVE, payment_intent_id)            │
          │  - reading_sessions                                        │
          └─────────────────────────────────────────────────────────────┘

          ┌─────────────────────────────────────────────────────────────┐
          │              Payment Service Database                       │
          │  - payment_intents (rental_id, amount, status)             │
          │  - transactions                                            │
          └─────────────────────────────────────────────────────────────┘
```

### 7.2 Схема аренды физической книги (CASH) — Event-Driven

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│  User    │────>│ Book Service │────>│  RabbitMQ    │────>│Notification Svc  │
│          │     │              │     │  Exchange    │     │                  │
│          │<────│              │     │              │     │                  │
│  (Receipt)│    │              │     │              │     │                  │
└──────────┘     │              │     │              │     │                  │
                 │ 1. Проверка доступности           │     │                  │
                 │ 2. Проверка debt пользователя     │     │                  │
                 │ 3. SELECT ... FOR UPDATE (book)   │     │                  │
                 │ 4. Создание Rental (PENDING_CONF) │     │                  │
                 │ 5. Генерация receipt_code         │     │                  │
                 │ 6. Публикация: RentalReceiptReady │────>│                  │
                 │              │                    │     │ 7. Отправка      │
                 │              │                    │     │    email с чеком │
                 ▼              ▼                    ▼     ▼                  ▼
          ┌─────────────────────────────────────────────────────────────┐
          │              Book Service Database                          │
          │  - books (physical_inventory, version + 1)                 │
          │  - rentals (status = PENDING_CONFIRMATION)                 │
          └─────────────────────────────────────────────────────────────┘

--- Admin Confirmation ---

┌──────────┐     ┌──────────────┐     ┌──────────────────┐
│  Admin   │────>│ Book Service │────>│  Database        │
│ (Scan QR)│     │              │     │                  │
│          │     │ 1. Найти rental по receipt_code       │
│          │     │ 2. physical_inventory - 1             │
│          │     │ 3. Status → ACTIVE                    │
│          │     │ 4. is_receipt_confirmed = true        │
│          │     │ 5. Публикация: RentActivated          │
└──────────┘     └──────────────┘     └──────────────────┘
```

### 7.3 Схема возврата физической книги — Event-Driven

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Admin   │────>│ Book Service │────>│  RabbitMQ    │────>│Payment Svc   │
│ (Scan QR)│     │              │     │  Exchange    │     │              │
│          │     │              │     │              │     │              │
│          │<────│              │<────│              │<────│              │
│  Result  │     │              │     │              │     │              │
└──────────┘     │              │     │              │     │              │
                 │ 1. Найти rental по коду           │     │              │
                 │ 2. Рассчитать штраф (если есть)   │     │              │
                 │ 3. physical_inventory + 1         │     │              │
                 │ 4. actual_return_date = now       │     │              │
                 │ 5. Status → RETURNED              │     │              │
                 │ 6. Если штраф > 0:                │     │              │
                 │    Публикация: FineChargeRequested│────>│              │
                 │    → Payment: Capture/Release     │     │              │
                 │    → Ответ: FineChargeProcessed   │<────│              │
                 │ 7. Если штраф = 0:                │     │              │
                 │    Публикация: DepositReleaseReq  │────>│              │
                 │    → Payment: Release             │     │              │
                 │    → Ответ: DepositReleased       │<────│              │
                 │ 8. Обновить fine_status, user_debt│     │              │
                 │              │                    │     │              │
                 ▼              ▼                    ▼     ▼              ▼
          ┌─────────────────────────────────────────────────────────────┐
          │              Book Service Database                          │
          │  - books (physical_inventory + 1, version + 1)             │
          │  - rentals (status = RETURNED, actual_return_date)         │
          │  - fines (если есть штраф)                                 │
          │  - user_debt (если штраф > депозит)                        │
          └─────────────────────────────────────────────────────────────┘
```

### 7.4 Scheduled Job: Проверка просроченных аренд

```
┌─────────────────────────────────────────────────────────────┐
│                    Scheduled Job (Daily)                     │
│                    @Scheduled(cron = "0 0 3 * * *")          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
          ┌────────────────────────────────┐
          │  Найти все аренды где:         │
          │  status = ACTIVE               │
          │  AND end_date < current_date   │
          │  (SELECT ... FOR UPDATE)       │
          └────────────────┬───────────────┘
                           │
                           ▼
          ┌────────────────────────────────┐
          │  Для каждой просроченной:      │
          │                                │
          │  1. Рассчитать days_overdue    │
          │  2. Рассчитать fine_amount     │
          │  3. Если fine >= price:        │
          │     → Status = LOST            │
          │     → Списать стоимость        │
          │     → inventory + 1            │
          │     → reading_sessions close   │
          │  4. Иначе:                     │
          │     → Status = OVERDUE         │
          │     → Создать Fine запись      │
          │     → Начислить в user_debt    │
          │  5. Отправить уведомление      │
          │     → Публикация в RabbitMQ    │
          └────────────────┬───────────────┘
                           │
                           ▼
          ┌────────────────────────────────┐
          │  RabbitMQ Queue                │
          │  → Email Notification Service  │
          │  → Fine Calculation Service    │
          └────────────────────────────────┘
```

### 7.5 Схема покупки цифровой книги (только CARD) — Event-Driven

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  User    │────>│ Book Service │────>│  RabbitMQ    │────>│Payment Svc   │
│          │     │              │     │  Exchange    │     │              │
│          │<────│              │<────│              │<────│              │
└──────────┘     │              │     │              │     │              │
                 │ 1. Проверка доступности           │     │              │
                 │ 2. Проверка debt пользователя     │     │              │
                 │ 3. Проверка: нет покупки этой книги│    │              │
                 │ 4. Создание Purchase (PENDING)    │     │              │
                 │ 5. Публикация: PurchaseCreated    │────>│              │
                 │              │                    │     │ 6. Создание   │
                 │              │                    │     │    invoice    │
                 │              │                    │     │ 7. Списание   │
                 │              │                    │     │    price      │
                 │              │                    │     │ 8. Публикация:│
                 │              │                    │<────│ PaymentComp   │
                 │ 9. Status → COMPLETED             │     │              │
                 │ 10. Генерация PdfCopy             │     │              │
                 │ 11. PDF → S3/MinIO                │     │              │
                 │ 12. watermarked_file_url → БД     │     │              │
                 │ 13. total_purchases_count + 1     │     │              │
                 │ 14. Публикация: PurchaseCompleted │────>│              │
                 ▼              ▼                    ▼     ▼              ▼
          ┌─────────────────────────────────────────────────────────────┐
          │              Book Service Database                          │
          │  - purchases (status = COMPLETED)                          │
          │  - pdf_copies (anti_piracy_code, watermarked_file_url)     │
          │  - books (total_purchases_count + 1)                       │
          └─────────────────────────────────────────────────────────────┘

Примечание: Покупка цифровой версии доступна ТОЛЬКО через карту (CARD).
Оплата наличными не поддерживается — цифровой товар доставляется мгновенно.
```

### 7.6 Схема бронирования физической книги

```
┌──────────┐     ┌──────────────┐     ┌──────────────────┐
│  User    │────>│ Book Service │────>│  Database        │
│          │     │              │     │                  │
│  (Reserve)│    │ 1. Проверить inventory = 0           │
│          │     │ 2. Создать Reservation (WAITING)     │
│          │     │ 3. Добавить в очередь                │
└──────────┘     └──────────────┘     └──────────────────┘

--- Book Returned ---

┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│  Admin   │────>│ Book Service │────>│  RabbitMQ    │────>│Notification Svc  │
│ (Confirm  │     │              │     │  Exchange    │     │                  │
│  Return)  │     │ 1. physical_inventory + 1        │     │                  │
│          │     │ 2. Проверить reservations         │     │                  │
│          │     │ 3. Если есть WAITING:             │     │                  │
│          │     │    → Первый → READY               │     │                  │
│          │     │    → ready_at = now               │     │                  │
│          │     │    → expires_at = now + 24h       │     │                  │
│          │     │ 4. Публикация: ReservationReady   │────>│                  │
│          │     │                                    │     │ 5. Отправить     │
│          │     │                                    │     │    уведомление   │
└──────────┘     └──────────────┘     └──────────────┘     └──────────────────┘

--- Reservation Expired ---

┌─────────────────────────────────────┐
│  Scheduled Job (Hourly)             │
│                                     │
│  Найти reservations где:            │
│  status = READY                     │
│  AND expires_at < current_time      │
│                                     │
│  Для каждой:                        │
│  → Status = EXPIRED                 │
│  → Следующий WAITING → READY        │
│  → Отправить уведомление            │
└─────────────────────────────────────┘
```

### 7.7 Схема скачивания PDF с анти-пиратской защитой

```
┌──────────┐     ┌──────────────┐     ┌──────────────────┐     ┌──────────┐
│  User    │────>│ Book Service │────>│  S3 / MinIO      │────>│  User    │
│ (Download)│    │              │     │  Object Storage  │     │ (PDF)    │
│          │     │ 1. Проверить purchase               │     │          │
│          │     │ 2. Проверить download_count         │     │          │
│          │     │    < max_downloads                  │     │          │
│          │     │ 3. Если watermarked_file_url есть:  │     │          │
│          │     │    → Отдать URL из S3               │     │          │
│          │     │ 4. Если файла нет:                  │     │          │
│          │     │    → Сгенерировать anti_piracy_code │     │          │
│          │     │    → Применить водяные знаки        │     │          │
│          │     │    → Вычислить SHA-256 hash         │     │          │
│          │     │    → Загрузить в S3                 │     │          │
│          │     │    → Сохранить URL и hash в БД      │     │          │
│          │     │    → download_count + 1             │     │          │
│          │     │    → Отдать URL из S3               │     │          │
│          │<────│              │     │                  │     │          │
└──────────┘     └──────────────┘     └──────────────────┘     └──────────┘
                           │
                           ▼
          ┌────────────────────────────────┐
          │     Book Service Database      │
          │ - pdf_copies                   │
          │   - anti_piracy_code           │
          │   - watermarked_file_url       │
          │   - pdf_file_hash              │
          │   - visible_watermark          │
          │   - invisible_metadata         │
          │   - download_count             │
          └────────────────────────────────┘
```

### 7.8 Схема удаления пользователя (GDPR Anonymization)

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  User    │────>│ Book Service │────>│  RabbitMQ    │────>│Notification Svc│
│ (Delete  │     │              │     │  Exchange    │     │              │
│  Request)│     │              │     │              │     │              │
│          │     │ 1. Валидация запроса             │     │              │
│          │     │ 2. Публикация: UserDeletionReq   │────>│              │
│          │     │ 3. Анонимизация users:           │     │              │
│          │     │    - email → deleted_{uuid}@...  │     │              │
│          │     │    - username → deleted_user_... │     │              │
│          │     │    - name, phone → NULL          │     │              │
│          │     │    - password → random hash      │     │              │
│          │     │ 4. rentals, fines, purchases     │     │              │
│          │     │    → СОХРАНЯЮТСЯ (отчётность)    │     │              │
│          │     │ 5. user_debt → сохраняется       │     │              │
│          │<────│ 6. Ответ: UserAnonymized         │     │              │
└──────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

---

## 8. Очереди сообщений (RabbitMQ)

### 8.1 Exchange: `library.book.events`

| Queue | Routing Key | Описание |
|-------|-------------|----------|
| `library.payment.rent.created` | `rent.created` | Book → Payment: запрос pre-auth для аренды |
| `library.payment.purchase.created` | `purchase.created` | Book → Payment: запрос оплаты покупки |
| `library.payment.fine.charge` | `fine.charge` | Book → Payment: списание штрафа из депозита |
| `library.payment.deposit.release` | `deposit.release` | Book → Payment: снятие холдирования депозита |
| `library.payment.debt.payment` | `debt.payment` | Book → Payment: оплата задолженности |

### 8.2 Exchange: `library.payment.events`

| Queue | Routing Key | Описание |
|-------|-------------|----------|
| `library.book.payment.authorized` | `payment.authorized` | Payment → Book: pre-auth успешен |
| `library.book.payment.completed` | `payment.completed` | Payment → Book: оплата завершена |
| `library.book.payment.failed` | `payment.failed` | Payment → Book: оплата отклонена |
| `library.book.fine.charge.processed` | `fine.charge.processed` | Payment → Book: штраф обработан |
| `library.book.deposit.released` | `deposit.released` | Payment → Book: депозит возвращён |
| `library.book.debt.payment.completed` | `debt.payment.completed` | Payment → Book: долг погашен |

### 8.3 Exchange: `library.notification.events`

| Queue | Routing Key | Описание |
|-------|-------------|----------|
| `library.notification.rent.activated` | `rent.activated` | Book → Notification: аренда активирована |
| `library.notification.receipt.ready` | `rental.receipt` | Book → Notification: чек аренды готов |
| `library.notification.purchase.receipt` | `purchase.receipt` | Book → Notification: чек покупки готов |
| `library.notification.overdue.reminder` | `overdue.reminder` | Book → Notification: напоминание о просрочке |
| `library.notification.fine.alert` | `fine.notification` | Book → Notification: уведомление о штрафе |
| `library.notification.availability.alert` | `availability.alert` | Book → Notification: книга стала доступна (wishlist) |
| `library.notification.reservation.ready` | `reservation.ready` | Book → Notification: бронирование готово |
| `library.notification.return.confirmation` | `return.confirmation` | Book → Notification: подтверждение возврата |
| `library.notification.user.anonymized` | `user.anonymized` | Book → Notification: пользователь анонимизирован |

---

## 9. Защита от Race Condition

### 9.1 Пессимистичная блокировка

При аренде/покупке последней копии:
```sql
SELECT * FROM books WHERE id = ? FOR UPDATE;
-- Проверка physical_inventory / digital_licenses
-- Обновление с проверкой что значение > 0
UPDATE books SET physical_inventory = physical_inventory - 1, version = version + 1 WHERE id = ?;
```

### 9.2 Optimistic Locking

Поле `version` в таблице `books`:
```sql
UPDATE books SET physical_inventory = ?, version = version + 1
WHERE id = ? AND version = ?;
-- Если 0 строк обновлено → конфликт, повторить операцию
```

### 9.3 Redis (опционально)

Для высоконагруженных операций:
```
DECR book:{id}:inventory
Если результат < 0 → INCR (откат) → ошибка "нет в наличии"
```

---

## 10. Индексы для частых запросов

### 10.1 Критичные индексы

| Таблица | Поле(я) | Тип | Для чего |
|---------|---------|-----|----------|
| `books` | `isbn` | UNIQUE | Проверка уникальности ISBN |
| `books` | `title` | B-Tree | Поиск по названию |
| `books` | `publication_year` | B-Tree | Фильтрация по году |
| `books` | `is_active` | B-Tree | Фильтрация активных книг |
| `rentals` | `user_id` | B-Tree | Аренды пользователя |
| `rentals` | `book_id` | B-Tree | Аренды книги |
| `rentals` | `status` | B-Tree | Фильтрация по статусу |
| `rentals` | `end_date` | B-Tree | Поиск просроченных аренд |
| `rentals` | `receipt_code` | UNIQUE | Поиск по штрих-коду |
| `purchases` | `user_id` | B-Tree | Покупки пользователя |
| `purchases` | `book_id` | B-Tree | Покупки книги |
| `purchases` | `status` | B-Tree | Фильтрация по статусу |
| `fines` | `rental_id` | B-Tree | Штрафы аренды |
| `fines` | `user_id` | B-Tree | Штрафы пользователя |
| `fines` | `status` | B-Tree | Фильтрация по статусу |
| `pdf_copies` | `anti_piracy_code` | UNIQUE | Поиск по коду |
| `pdf_copies` | `status` | B-Tree | Фильтрация по статусу |
| `reservations` | `status` | B-Tree | Очередь бронирований |
| `reservations` | `expires_at` | B-Tree | Поиск истёкших бронирований |
| `reading_sessions` | `last_heartbeat` | B-Tree | Поиск неактивных сессий |
| `reading_sessions` | `is_active` | B-Tree | Активные сессии |
| `reviews` | `book_id` | B-Tree | Отзывы книги |
| `reviews` | `status` | B-Tree | Модерация отзывов |
| `wishlist` | `user_id, book_id` | COMPOSITE | Проверка наличия в wishlist |
| `bookmarks` | `user_id, book_id` | COMPOSITE | Закладки пользователя |
| `reading_progress` | `user_id, book_id` | COMPOSITE | Прогресс пользователя |
| `promo_codes` | `code` | UNIQUE | Поиск промокода |
| `promo_codes` | `is_active` | B-Tree | Фильтрация активных |
| `promo_code_usages` | `promo_code_id` | B-Tree | Аналитика промокодов |
| `book_authors` | `book_id, author_id` | COMPOSITE PK | Связь книга-автор |
| `book_categories` | `book_id, category_id` | COMPOSITE PK | Связь книга-категория |
| `authors` | `full_name` | B-Tree | Поиск авторов |
| `categories` | `name` | UNIQUE | Уникальность категорий |

---

## 11. Валюта

Все денежные суммы в системе указываются в **тенге (₸)**.

| Операция | Пример суммы |
|----------|-------------|
| Аренда цифровой книги | 500 ₸ / 30 дней |
| Аренда физической книги | 1 000 ₸ / 30 дней |
| Покупка цифровой книги | 5 000 ₸ |
| Покупка физической книги | 8 000 ₸ |
| Залог | 2 000 ₸ |
| Штраф за просрочку | 2% от стоимости покупки в день |
| Максимальный штраф | 100% от стоимости покупки |

---

## 12. Статусы сущностей

### 12.1 Rental Status

```
PENDING_PAYMENT → PENDING_CONFIRMATION → ACTIVE → OVERDUE → RETURNED
                                              ↓
                                            EXPIRED (digital auto)
                                            LOST (cap reached)
                                            CANCELLED
```

### 12.2 Purchase Status

**DIGITAL purchase (только CARD):**
```
PENDING_PAYMENT → COMPLETED
      ↓
    CANCELLED
```

**PHYSICAL purchase (CARD или CASH):**
```
PENDING_PAYMENT → PENDING_CONFIRMATION → COMPLETED
                      ↓
                    CANCELLED
```

### 12.3 Fine Status

```
PENDING → PAID
       → WAIVED
```

### 12.4 Reservation Status

```
WAITING → READY → EXPIRED
       → CANCELLED
```

### 12.5 Review Status

```
PENDING → APPROVED
       → REJECTED
```

### 12.6 Payment Intent Status (Payment Service)

```
CREATED → AUTHORIZED → CAPTURED
               ↓
           RELEASED
               ↓
           FAILED
```

---

## 13. Правила и ограничения

### 13.1 Блокировки пользователя

Пользователь блокируется от:
- Новых аренд
- Скачивания PDF
- Покупок (опционально)

Причины блокировки:
- `user_debt.total_debt > 0`
- Есть активные просроченные аренды (`status = OVERDUE`)
- Админ вручную забанил пользователя

### 13.2 Ограничения на аренду

- Нельзя арендовать книгу если:
  - Пользователь заблокирован
  - `physical_inventory = 0` (физическая)
  - `digital_licenses = 0` (цифровая, если не -1)
  - `is_available_for_rent = false`
  - Уже есть активная аренда этой же книги

### 13.3 Ограничения на покупку

- Нельзя купить книгу если:
  - Пользователь заблокирован (опционально)
  - `physical_inventory = 0` (физическая)
  - `is_available_for_purchase = false`
  - Уже есть покупка этой же книги (цифровая)
- **Цифровая версия** — оплата **только через карту (CARD)**, наличные не поддерживаются
- **Физическая версия** — оплата через карту (CARD) или наличными (CASH)

### 13.4 Ограничения на скачивание PDF

- Только при покупке цифровой версии
- Максимум 1 скачивание
- При аренде — только онлайн-чтение
- Готовый PDF кешируется в S3/MinIO

### 13.5 Ограничения на отзывы

- Только пользователи с историей аренды/покупки данной книги
- Один отзыв на книгу от пользователя
- Требуется модерация перед публикацией

### 13.6 Транзакционность критичных операций

| Операция | ACID гарантия |
|----------|--------------|
| Создание аренды + decrement inventory | Локальная транзакция Book Service |
| Подтверждение возврата + increment inventory | Локальная транзакция Book Service |
| Оплата через Payment Service | Saga-паттерн через события |
| Начисление штрафа + update user_debt | Локальная транзакция Book Service |
| Генерация PDF + запись в S3 | Компенсирующая транзакция при ошибке |

### 13.7 Heartbeat правила

- Интервал heartbeat: каждые 5 минут
- Таймаут без heartbeat: 15 минут
- При таймауте: `is_active = false`, сессия закрывается
- Heartbeat не освобождает лицензию (аренда = время, а не сессия)
- Heartbeat нужен для: аналитики, сохранения прогресса, обнаружения зависших сессий
