# Техническое задание на frontend приложения Library

> Документ составлен по фактическому коду backend на 24.07.2026.
> Backend: Spring Boot, REST API, JWT. Frontend: React SPA.

## 1. Цель приложения

Нужно реализовать веб-приложение библиотеки, в котором:

- пользователь регистрируется и входит в систему;
- авторизованный пользователь просматривает каталог, ищет и фильтрует книги;
- пользователь просматривает карточку книги и управляет своим профилем;
- администратор управляет книгами и пользователями;
- администратор запускает асинхронный импорт книг из Open Library.

В текущей версии backend **нет** API аренды, покупки, возврата, оплаты,
избранного, отзывов и онлайн-чтения. Кнопки этих действий нельзя делать
рабочими или имитировать на frontend. Их можно показывать только как
отключённые элементы с подписью «Скоро», если это требуется дизайном.

## 2. Рекомендуемый стек

### Обязательные технологии

| Задача | Библиотека |
|---|---|
| UI | `react`, `react-dom`, TypeScript |
| Сборка и dev server | `vite`, шаблон `react-ts` |
| Маршрутизация | `react-router` |
| Запросы, кеш и server state | `@tanstack/react-query` |
| HTTP-клиент и interceptors | `axios` |
| Формы | `react-hook-form` |
| Схемы и клиентская валидация | `zod`, `@hookform/resolvers` |
| Чтение payload JWT | `jwt-decode` |
| Unit/component-тесты | `vitest`, `@testing-library/react`, `@testing-library/user-event`, `jsdom` |
| Проверка кода | `eslint`, `prettier` |

React-приложение следует создавать через Vite, а не через устаревший
Create React App. Версии пакетов зафиксировать lock-файлом и использовать
последние совместимые стабильные версии.

Официальные справочные материалы:

- [React](https://react.dev/learn/installation)
- [Vite](https://vite.dev/guide/)
- [React Router](https://reactrouter.com/)
- [TanStack Query](https://tanstack.com/query/latest/docs/framework/react/overview)
- [React Hook Form](https://react-hook-form.com/get-started)
- [Zod](https://zod.dev/)

### UI-kit

Рекомендуется выбрать **один** UI-kit, например `@mui/material` с
`@emotion/react` и `@emotion/styled`. Допустимы Ant Design, Mantine или
собственная дизайн-система, но нельзя смешивать несколько UI-kit.

Redux для текущего объёма приложения не требуется:

- server state хранится в TanStack Query;
- состояние сессии — в `AuthProvider`;
- локальное состояние форм — в React Hook Form;
- фильтры каталога желательно хранить в query-параметрах URL.

## 3. Конфигурация и запуск

Backend по умолчанию доступен на:

```text
http://localhost:8080
```

Frontend должен читать адрес API из переменной:

```env
VITE_API_URL=/api-backend
```

В dev-режиме настройте proxy Vite:

```ts
// vite.config.ts
server: {
  proxy: {
    '/api-backend': {
      target: 'http://localhost:8080',
      changeOrigin: true,
      rewrite: (path) => path.replace(/^\/api-backend/, ''),
    },
  },
}
```

Это обязательно для локальной разработки: в текущей `SecurityConfig`
CORS не включён. В production frontend и backend следует публиковать под
одним origin через reverse proxy либо отдельно добавить CORS на backend.

Swagger backend:

```text
http://localhost:8080/swagger-ui/index.html
http://localhost:8080/v3/api-docs
```

## 4. Роли и доступ

Роли:

- `ROLE_USER` — обычный пользователь;
- `ROLE_ADMIN` — администратор.

Фактические правила backend:

| Область | Доступ |
|---|---|
| `/auth/**` | без авторизации |
| Swagger и `/actuator/health` | без авторизации |
| `/admin/**` | только `ROLE_ADMIN` |
| изменение/удаление/импорт книг | только `ROLE_ADMIN` |
| чтение каталога `/api/v1/books/**` | любой авторизованный пользователь |
| `/user/**` | авторизованный пользователь; удаление — только администратор |

Важно: каталог сейчас **не публичный**. Неавторизованного посетителя нужно
перенаправлять на `/login`.

JWT содержит:

```ts
interface JwtPayload {
  sub: string;    // username
  userId: number;
  iat: number;
  exp: number;
}
```

JWT не содержит роли. После login/register frontend должен:

1. сохранить токены;
2. декодировать `userId` из access token;
3. вызвать `GET /user/{userId}`;
4. сохранить профиль с `roles` в состоянии сессии;
5. показывать административные маршруты только при наличии `ROLE_ADMIN`.

Проверка роли на frontend нужна для UX, но реальную безопасность всегда
обеспечивает backend.

## 5. Маршруты и экраны

### Публичные маршруты

| URL frontend | Экран |
|---|---|
| `/login` | Вход |
| `/register` | Регистрация |

Если пользователь уже вошёл, эти страницы перенаправляют на `/books`.

### Маршруты авторизованного пользователя

| URL frontend | Экран |
|---|---|
| `/books` | Каталог с пагинацией, сортировкой и поиском |
| `/books/:id` | Детальная карточка книги |
| `/profile` | Профиль текущего пользователя |
| `/profile/edit` | Редактирование профиля |
| `/change-password` | Смена пароля |
| `/403` | Нет прав |
| `*` | Страница 404 |

### Маршруты администратора

| URL frontend | Экран |
|---|---|
| `/admin` | Dashboard |
| `/admin/users` | Таблица пользователей |
| `/admin/users/:id` | Пользователь: данные, роли, статус, удаление |
| `/admin/books` | Управление книгами |
| `/admin/books/new` | Создание книги |
| `/admin/books/:id/edit` | Редактирование книги |
| `/admin/import` | Импорт книг |

Нужны два route guard:

- `RequireAuth` — проверяет наличие сессии;
- `RequireRole('ROLE_ADMIN')` — проверяет загруженный профиль и роль.

Во время восстановления сессии показывать loader, а не кратковременно
рендерить страницу входа или ошибку 403.

## 6. Авторизация

### Регистрация

```http
POST /auth/register
Content-Type: application/json
```

```ts
interface RegistrationRequest {
  email: string;       // required, email
  username: string;    // required, 3..50 символов
  password: string;    // required, минимум 8 символов
  firstName?: string;
  lastName?: string;
}
```

Успех: `201 Created` и `AuthResponse`. Регистрация сразу авторизует
пользователя.

### Вход

```http
POST /auth/login
```

```ts
interface LoginRequest {
  username: string;
  password: string;
}

interface AuthResponse {
  accessToken: string;
  refreshToken: string;
  tokenType: string; // "Bearer"
  expiresIn: number; // сейчас миллисекунды, 86400000
}
```

Поле входа называется `username`: backend не поддерживает вход по email.

### Обновление access token

```http
POST /auth/refresh
```

```json
{
  "refreshToken": "<refresh-token>"
}
```

Успех: `200 OK`, новый `accessToken` и прежний `refreshToken`.

Axios interceptor должен:

- добавлять `Authorization: Bearer <accessToken>`;
- до защищённого запроса проверять `exp` и заранее обновлять истёкший или
  почти истёкший access token;
- при первом `401` один раз вызвать `/auth/refresh`;
- поставить параллельные запросы в очередь на время refresh;
- повторить исходный запрос только один раз;
- не пытаться refresh-ить запросы `/auth/login`, `/auth/register` и
  `/auth/refresh`;
- при неуспешном refresh очистить сессию и открыть `/login`.

Проактивная проверка `exp` важна для текущего backend: отдельный
`AuthenticationEntryPoint` не настроен, поэтому запрос с отсутствующим или
просроченным токеном в некоторых случаях может завершиться `403`, а не
ожидаемым `401`. Нельзя автоматически делать refresh при каждом `403`,
иначе запрос пользователя без роли администратора зациклится. При `403`
сначала проверить срок JWT: refresh допустим только для истёкшего токена,
а обычный отказ в роли нужно показать как «Нет доступа».

### Выход

```http
POST /auth/logout
Authorization: Bearer <access-token>
```

Успех: `204 No Content`. После ответа или сетевой ошибки frontend всё
равно очищает локальную сессию.

### Смена пароля

```http
POST /auth/change-password
Authorization: Bearer <access-token>
```

```ts
interface ChangePasswordRequest {
  currentPassword: string;
  newPassword: string; // минимум 8 символов
}
```

Успех: `204 No Content`. Backend отзывает refresh token. После успешной
смены пароля нужно очистить сессию и отправить пользователя на login.

### Хранение токенов

Текущий backend возвращает оба токена в JSON и не устанавливает httpOnly
cookie. Для совместимости допустимо хранить их в `localStorage`, понимая
риск XSS. Предпочтительный production-вариант потребует изменения backend:
refresh token в `Secure; HttpOnly; SameSite` cookie, access token в памяти.

## 7. Профиль пользователя

```ts
interface User {
  id: number;
  email: string;
  username: string;
  firstName: string | null;
  lastName: string | null;
  isActive: boolean;
  isEmailVerified: boolean;
  roles: string[];
}
```

### Получение

```http
GET /user/{id}
```

Для обычного интерфейса всегда использовать `userId` из JWT. Текущий
backend технически не проверяет, что пользователь читает только свой ID;
frontend не должен предоставлять переход к чужим профилям.

### Редактирование

```http
PUT /user/{id}
```

```ts
interface UserUpdateRequest {
  email?: string;
  username?: string;       // 3..50 символов
  password?: string;
  currentPassword: string; // обязателен при любом обновлении профиля
  firstName?: string;
  lastName?: string;
}
```

Форма должна явно объяснять, что текущий пароль нужен для подтверждения
любых изменений. После смены username нужно обязательно выполнить logout:
уже выданный JWT содержит прежний username в `sub` и больше не сможет
аутентифицировать запросы. После смены пароля через этот endpoint также
безопаснее выполнить logout.

## 8. Каталог книг

### Основной тип книги

```ts
interface Book {
  id: number;
  title: string;
  description: string | null;
  isbn: string | null;
  pricePurchase: number;
  priceRental: number | null;
  depositAmount: number | null;
  hasPhysical: boolean;
  hasDigital: boolean;
  physicalInventory: number;
  digitalLicenses: number;
  isAvailableForRent: boolean;
  isAvailableForPurchase: boolean;
  publishedYear: number | null;
  coverImageUrl: string | null;
  totalRentalsCount: number;
  totalPurchasesCount: number;
}
```

JSON-числа цен приходят как `number`. Для вывода денег использовать
`Intl.NumberFormat`; валюту согласовать с владельцем продукта. Backend
сейчас не возвращает код валюты.

`digitalLicenses === -1` означает неограниченное количество лицензий.

### Каталог с пагинацией

```http
GET /api/v1/books?page=0&size=10&sort=title,asc
```

Нумерация страниц backend начинается с нуля. Формат Spring Page:

```ts
interface PageResponse<T> {
  content: T[];
  totalElements: number;
  totalPages: number;
  size: number;
  number: number;
  first: boolean;
  last: boolean;
  numberOfElements: number;
  empty: boolean;
  sort?: unknown;
  pageable?: unknown;
}
```

На экране каталога нужны:

- карточки или таблица книг;
- skeleton во время загрузки;
- empty state;
- пагинация;
- выбор размера страницы: 10, 20, 50;
- сортировка хотя бы по `title`, `publishedYear`, `pricePurchase`;
- поиск по названию;
- синхронизация `page`, `size`, `sort`, `title` с URL.

Примечание backend: пагинированный метод сейчас может возвращать также
мягко удалённые книги, потому что использует `findAll(pageable)` без
фильтра `isActive`. Это нужно исправить на backend; frontend не получает
`isActive` в `Book` и самостоятельно отфильтровать их не может.

### Другие операции чтения

| Метод | Endpoint | Ответ |
|---|---|---|
| GET | `/api/v1/books/{id}` | `Book` |
| GET | `/api/v1/books/isbn/{isbn}` | `Book` |
| GET | `/api/v1/books/all` | `Book[]` |
| GET | `/api/v1/books/search?title=java` | `Book[]` |
| GET | `/api/v1/books/genre/{genre}` | `BookPreview[]` |
| GET | `/api/v1/books/author/{author}` | `BookPreview[]` |
| GET | `/api/v1/books/category/{category}` | `BookPreview[]` |

```ts
interface BookPreview {
  title: string;
  description: string | null;
  pricePurchase: number;
  publishedYear: number | null;
  coverImageUrl: string | null;
}
```

Для параметров пути использовать `encodeURIComponent`.

Ограничение API: `BookPreview` не содержит `id`, поэтому результат
фильтра по автору/жанру/категории нельзя надёжно открыть как детальную
страницу. Основной рабочий поиск следует строить через `/search`; для
полноценных фильтров backend должен добавить `id`.

Ещё одно ограничение: основной `Book` не возвращает авторов, жанры и
категории. Пока их нельзя показать в детальной карточке без доработки API.

### Карточка книги

Показывать:

- обложку или локальный placeholder при пустом/ошибочном URL;
- название, описание, ISBN, год;
- цену покупки, цену аренды и залог;
- физический и цифровой форматы;
- остаток физических копий;
- число цифровых лицензий или «Без ограничений» для `-1`;
- доступность аренды и покупки;
- счётчики аренд и покупок.

Не считать книгу доступной только по остатку: использовать возвращаемые
backend флаги `isAvailableForRent` и `isAvailableForPurchase`.

## 9. Управление книгами для администратора

### Создание

```http
POST /api/v1/books
```

```ts
interface CreateBookRequest {
  title: string;                 // required, максимум 255
  description?: string;
  isbn?: string;                 // максимум 50
  author: string;                // required, максимум 255
  genre: string;                 // required
  category?: string;
  pricePurchase: number;         // required, >= 0
  priceRental?: number;          // >= 0
  depositAmount?: number;        // >= 0
  physicalInventory: number;     // required, integer >= 0
  digitalLicenses?: number;      // -1 означает unlimited
  hasPhysical?: boolean;         // default backend: true
  hasDigital?: boolean;          // default backend: true
  publishedYear?: number;        // 1000..2099
  coverImageUrl?: string;
}
```

Успех: `201 Created`, тело `Book`.

Frontend-правила формы:

- HTML input с `type="number"` преобразовывать в number до отправки;
- запретить отрицательные цены и остаток;
- для unlimited digital licenses использовать checkbox, отправляющий `-1`;
- если `hasPhysical === false`, предложить установить
  `physicalInventory = 0`;
- показать preview обложки и fallback при ошибке загрузки;
- ISBN можно вводить с пробелами/дефисами — backend нормализует его.

Backend не предоставляет справочники авторов, жанров и категорий.
Автор создаётся автоматически, если его ещё нет. Жанр и категория,
напротив, должны уже существовать в БД, иначе возможна ошибка 500.
До появления reference API поля `genre` и `category` должны быть
согласованными текстовыми значениями; не обещать пользователю создание
нового жанра через эту форму.

В текущих миграциях созданы категории `Programming`, `Java`, `Spring`,
`Backend`, `Software Engineering` и жанры `Fantasy`, `Science Fiction`,
`Detective`, `Romance`, `Thriller`, `Horror`, `Adventure`, `Mystery`,
`Historical Fiction`, `Non-Fiction`, `Biography`, `Self-Help`,
`Philosophy`, `Poetry`, `Drama`, `Comedy`, `Dystopia`, `Young Adult`,
`Children`, `Classic`. Эти значения можно временно использовать в
select-компонентах, но лучше вынести их в конфигурацию, пока backend не
предоставляет справочник.

### Редактирование

```http
PUT /api/v1/books/{id}
PATCH /api/v1/books/{id}
```

Оба endpoint принимают один тип и фактически применяют только переданные
не-null поля:

```ts
interface UpdateBookRequest {
  title?: string;
  description?: string;
  isbn?: string;
  pricePurchase?: number;
  priceRental?: number;
  depositAmount?: number;
  physicalInventory?: number;
  digitalLicenses?: number;
  hasPhysical?: boolean;
  hasDigital?: boolean;
  isAvailableForRent?: boolean;
  isAvailableForPurchase?: boolean;
  publishedYear?: number;
  coverImageUrl?: string;
  isActive?: boolean;
}
```

Для frontend рекомендуется использовать `PATCH`. Автор, жанр и категория
через update API не изменяются.

После успешной мутации инвалидировать query keys списка и конкретной
книги.

### Изменение цен

```http
PATCH /api/v1/books/{id}/prices
```

```ts
interface BookPricesRequest {
  pricePurchase: number;
  priceRental?: number;
  depositAmount?: number;
}
```

### Удаление

```http
DELETE /api/v1/books/{id}
DELETE /api/v1/books/hard-delete/{id}
```

- обычное удаление — мягкая деактивация;
- hard delete — необратимое удаление.

Перед обоими действиями нужен confirm dialog. Hard delete должен иметь
особо заметное предупреждение и подтверждение названием книги.
Успешный ответ: `204 No Content`.

## 10. Импорт книг

```http
POST /api/v1/books/import
```

```ts
interface BookImportRequest {
  query?: string;
  title?: string;
  author?: string;
  limit?: number; // default backend: 20
}
```

Хотя бы одно из полей `query`, `title`, `author` должно быть непустым.
Успех: `202 Accepted` без тела.

Импорт асинхронный через RabbitMQ. `202` означает, что задача принята, а
не что все книги уже импортированы. Показывать уведомление «Импорт
запущен» и кнопку обновления списка. API статуса/прогресса задачи сейчас
нет, поэтому progress bar с реальным процентом реализовать невозможно.

## 11. Администрирование пользователей

### Dashboard

```http
GET /admin/dashboard
```

```ts
interface AdminDashboard {
  totalUsers: number;
  activeUsers: number;
  totalBooks: number;
  activeBooks: number;
  usersByRole: Record<string, number>;
}
```

Показать четыре summary card и распределение по ролям.

### Список пользователей

```http
GET /admin/users?page=0&size=20&sort=createdAt,desc
```

Ответ: `PageResponse<AdminUser>`.

```ts
interface AdminUser extends User {
  createdAt: string;
  updatedAt: string;
}
```

Таблица:

- username, email, имя;
- active/inactive;
- email verified/not verified;
- роли;
- даты создания и обновления;
- переход к деталям;
- пагинация и сортировка.

Backend пока не поддерживает серверный поиск пользователей.

### Детали пользователя

```http
GET /admin/users/{id}
```

### Изменение ролей

```http
PUT /admin/users/{id}/roles
```

```json
{
  "roles": ["ROLE_USER", "ROLE_ADMIN"]
}
```

Массив не может быть пустым. В UI доступны только существующие роли
`ROLE_USER` и `ROLE_ADMIN`.

### Активация и деактивация

```http
PATCH /admin/users/{id}/status
```

```json
{
  "isActive": false
}
```

При деактивации показать подтверждение. Если текущий администратор
деактивировал себя, после следующего запроса/refresh его сессия перестанет
работать; frontend должен корректно перейти на login.

### Удаление

```http
DELETE /user/{id}
```

Только для администратора, успех `204`. Backend может запретить удаление,
например если у пользователя есть платёжные методы; сообщение backend
нужно показать в диалоге/уведомлении.

## 12. Ошибки и состояния интерфейса

Backend возвращает ошибки в нескольких форматах.

Обычная ошибка:

```ts
interface ApiError {
  status: number;
  message: string;
  timestamp: string;
}
```

Ошибка user-модуля:

```ts
interface UserApiError extends ApiError {
  path: string;
}
```

Ошибка Bean Validation:

```json
{
  "email": "must be a well-formed email address",
  "password": "size must be between 8 and 2147483647"
}
```

Нужна единая функция `normalizeApiError(error)`, которая различает
объект с `message` и map ошибок полей.

Обработка статусов:

| Статус | Поведение |
|---|---|
| 400 | ошибки формы возле полей или общее сообщение |
| 401 | refresh; при неуспехе — очистка сессии и login |
| 403 | если JWT истёк — один refresh; иначе `/403` или бизнес-ошибка рядом с действием |
| 404 | empty/not-found state |
| 409 | показать конфликт: username, email или ISBN уже существует |
| 410 | показать, что токен/ресурс истёк |
| 500 | общее сообщение и возможность повторить запрос |

На каждой странице должны быть состояния loading, error, empty и success.
Для мутаций блокировать повторное нажатие, пока запрос выполняется.
Уведомления не должны раскрывать access/refresh token или внутренний stack
trace.

## 13. Рекомендуемая структура frontend

```text
src/
  app/
    providers/
    router/
    queryClient.ts
  api/
    httpClient.ts
    auth.api.ts
    books.api.ts
    users.api.ts
    admin.api.ts
    contracts.ts
  features/
    auth/
    books/
    profile/
    admin/
  components/
    ui/
    layout/
    feedback/
  hooks/
  lib/
    authStorage.ts
    jwt.ts
    apiError.ts
    formatters.ts
  pages/
  test/
  main.tsx
```

Не обращаться к Axios прямо из визуальных компонентов. HTTP-функции
держать в `api`, query/mutation hooks — внутри соответствующей feature.

Пример query keys:

```ts
const bookKeys = {
  all: ['books'] as const,
  lists: () => [...bookKeys.all, 'list'] as const,
  list: (params: BookListParams) => [...bookKeys.lists(), params] as const,
  detail: (id: number) => [...bookKeys.all, 'detail', id] as const,
};
```

## 14. UX и нефункциональные требования

- Адаптивность: desktop, tablet, mobile от 360 px.
- Семантический HTML и управление с клавиатуры.
- Видимый focus state.
- Label для каждого поля; ошибки связаны с полем через ARIA.
- Контрастность интерфейса не ниже WCAG AA.
- Все пользовательские тексты интерфейса — на русском; API-имена в коде
  остаются английскими.
- Не рендерить HTML из `description`; выводить как обычный текст.
- Для внешней обложки предусмотреть lazy loading и обработку ошибки.
- Не логировать токены и пароли.
- Не включать секреты в `VITE_*`: такие переменные видны браузеру.
- Даты backend без timezone отображать как локальные, не добавляя
  произвольное UTC-преобразование.
- Денежные вычисления на frontend не выполнять через накопление float;
  frontend здесь только отображает и отправляет значения.

## 15. Тестирование

Минимально покрыть:

- login/register validation;
- восстановление сессии;
- добавление Bearer token;
- один refresh при нескольких одновременных `401`;
- logout при неуспешном refresh;
- `RequireAuth` и `RequireRole`;
- каталог: loading, данные, empty, error, смена страницы;
- создание/редактирование книги и отображение backend validation;
- смену ролей и статуса пользователя;
- подтверждение мягкого и полного удаления;
- нормализацию всех трёх форматов ошибок.

Для API-mock в тестах рекомендуется `msw`. E2E можно добавить на Playwright
для сценариев login → каталог и admin → управление книгами.

## 16. Этапы реализации

### Этап 1 — каркас и авторизация

- Vite + React + TypeScript;
- router, providers, layout;
- HTTP client, token storage, refresh interceptor;
- login, register, logout, guards;
- загрузка текущего пользователя.

### Этап 2 — пользовательская часть

- каталог, поиск, сортировка, пагинация;
- детальная карточка;
- профиль, редактирование, смена пароля;
- общая обработка loading/error/empty.

### Этап 3 — административная часть

- dashboard;
- список и детали пользователей;
- роли, статус, удаление;
- CRUD книг, цены, soft/hard delete;
- импорт книг.

### Этап 4 — качество

- responsive UI и accessibility;
- unit/component tests;
- E2E основных сценариев;
- production build и инструкция запуска.

## 17. Критерии готовности

Frontend считается готовым, если:

- приложение запускается одной документированной командой;
- production build выполняется без ошибок TypeScript и ESLint;
- регистрация, login, refresh, logout и смена пароля работают с backend;
- после перезагрузки страницы сессия восстанавливается;
- пользователь видит каталог и свой профиль;
- пользователь без роли администратора не открывает admin routes;
- администратор управляет книгами и пользователями;
- формы повторяют ограничения backend;
- ошибки backend отображаются понятным текстом;
- реализованы loading, empty и error states;
- приложение работает на мобильном и desktop;
- токены и пароли не попадают в логи;
- основные сценарии покрыты автоматическими тестами.

## 18. Что нужно доработать на backend для полноценного frontend

Это не блокирует базовую SPA, но должно попасть в backend backlog:

1. Включить и настроить CORS либо официально закрепить same-origin proxy.
2. Настроить `AuthenticationEntryPoint`, возвращающий единый JSON и `401`
   для отсутствующего, некорректного или истёкшего access token.
3. Добавить endpoint `/auth/me` или включить роли в access token.
4. Ограничить `/user/{id}` и `PUT /user/{id}` текущим пользователем либо
   администратором.
5. Исключить неактивные книги из пагинированного каталога.
6. Добавить `id` в `BookPreview`.
7. Возвращать авторов, жанры и категории в `Book`.
8. Добавить API справочников авторов, жанров и категорий.
9. Унифицировать формат ошибок, включая validation errors.
10. Добавить endpoint статуса асинхронного импорта.
11. Добавить валюту к денежным полям.
12. Перенести refresh token в httpOnly cookie для production.

До реализации этих пунктов frontend должен следовать ограничениям,
описанным в соответствующих разделах выше.
