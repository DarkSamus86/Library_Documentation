# Email Verification & Notification System — Документация

## Обзор

Система асинхронных уведомлений Library отвечает за:
1. Отправку email-уведомлений при регистрации пользователя
2. Верификацию email через ссылку в письме
3. Автоматическое удаление неподтверждённых учётных записей через 24 часа

Архитектура построена на **RabbitMQ** (асинхронная доставка), **JavaMailSender** (отправка писем), **Spring Scheduling** (очистка просроченных аккаунтов).

---

## Архитектура

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Registration Flow                            │
│                                                                     │
│  POST /auth/register                                                │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────┐     ┌──────────────┐     ┌───────────────────┐    │
│  │ AuthController│──▶│ AuthService  │──▶ │ UserService       │    │
│  └─────────────┘     └──────────────┘     └───────────────────┘    │
│                                                  │                  │
│                                                  ▼                  │
│                                    ┌───────────────────────────┐   │
│                                    │ UserEventPublisher        │   │
│                                    │ (RabbitTemplate)          │   │
│                                    └───────────────────────────┘   │
│                                                  │                  │
│                                                  ▼                  │
│                                    ┌───────────────────────────┐   │
│                                    │ RabbitMQ                  │   │
│                                    │ Exchange: library.notification │
│                                    │ Routing Key: user.registered   │
│                                    │ Queue: library.notification.email │
│                                    └───────────────────────────┘   │
│                                                  │                  │
│                                                  ▼                  │
│                                    ┌───────────────────────────┐   │
│                                    │ EmailNotificationConsumer │   │
│                                    │ (@RabbitListener)         │   │
│                                    └───────────────────────────┘   │
│                                                  │                  │
│                                                  ▼                  │
│                                    ┌───────────────────────────┐   │
│                                    │ JavaMailSender            │   │
│                                    │ (SMTP → Gmail/другие)     │   │
│                                    └───────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     Email Verification Flow                         │
│                                                                     │
│  Пользователь кликает ссылку в письме                               │
│       │                                                             │
│       ▼                                                             │
│  GET /auth/verify?token=xxxxx                                       │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ EmailVerificationController                                 │   │
│  │  1. Найти токен в БД                                        │   │
│  │  2. Проверить срок действия (24h)                           │   │
│  │  3. Если валиден → isEmailVerified = true                   │   │
│  │  4. Удалить токен                                           │   │
│  │  5. Редирект на фронтенд с ?verified=true                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                  Unverified User Cleanup (Scheduled)                │
│                                                                     │
│  @Scheduled(cron = "0 0 * * * ?") — каждый час                      │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ UnverifiedUserCleanupService                                │   │
│  │  1. Найти пользователей с isEmailVerified = false           │   │
│  │     и createdAt < (now - 24 hours)                          │   │
│  │  2. Удалить их из БД                                        │   │
│  │  3. Логировать результат                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Компоненты

### 1. RabbitMQ — Асинхронная доставка событий

#### Конфигурация

| Параметр | Значение |
|----------|----------|
| Exchange | `library.notification` (Direct) |
| Queue | `library.notification.email` |
| Routing Key | `user.registered` |
| Message Converter | `JacksonJsonMessageConverter` (JSON) |
| Durable | Да (queue survives broker restart) |

#### Файлы

| Файл | Назначение |
|------|-----------|
| `config/RabbitMQConfig.java` | Объявление exchange, queue, binding, message converter |
| `notification/publisher/UserEventPublisher.java` | Публикация событий в RabbitMQ |
| `notification/consumer/EmailNotificationConsumer.java` | @RabbitListener, обработка событий, отправка email |
| `notification/event/UserRegisteredEvent.java` | DTO события (userId, email, username) |

#### UserEventPublisher

```java
@Component
public class UserEventPublisher {
    private final RabbitTemplate rabbitTemplate;

    public void publishUserRegistered(UserRegisteredEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.NOTIFICATION_EXCHANGE,
            RabbitMQConfig.ROUTING_KEY_USER_REGISTERED,
            event
        );
    }
}
```

#### EmailNotificationConsumer

```java
@Component
public class EmailNotificationConsumer {
    private final JavaMailSender mailSender;

    @RabbitListener(queues = RabbitMQConfig.QUEUE_EMAIL)
    public void handleUserRegistered(UserRegisteredEvent event) {
        // Отправка welcome email с ссылкой верификации
    }
}
```

---

### 2. Email — Отправка писем

#### SMTP Конфигурация

**application-test.yml:**
```yaml
spring:
  mail:
    host: ${MAIL_HOST:localhost}
    port: ${MAIL_PORT:1025}
```

**Для production (Gmail):**
```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${GMAIL_USERNAME}
    password: ${GMAIL_APP_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
          connectiontimeout: 5000
          timeout: 5000
          writetimeout: 5000
```

> **Важно:** Для Gmail необходимо использовать **App Password** (не обычный пароль пароля). Генерируется в Google Account → Security → App Passwords. Требуется включённая 2FA.

#### Для разработки (MailHog)

Docker Compose поднимает MailHog:
- SMTP: `localhost:1025`
- Web UI: `http://localhost:8025`

Все письма intercept-ятся и доступны через веб-интерфейс.

---

### 3. Verification Token — Сущность и хранилище

#### Entity: VerificationToken

```java
@Entity
@Table(name = "verification_tokens")
public class VerificationToken {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String token;          // UUID v4

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(name = "expires_at", nullable = false)
    private LocalDateTime expiresAt;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
        this.expiresAt = this.createdAt.plusHours(24); // 24 часа
    }

    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiresAt);
    }
}
```

#### Repository: VerificationTokenRepo

```java
public interface VerificationTokenRepo extends JpaRepository<VerificationToken, String> {
    Optional<VerificationToken> findByToken(String token);
    Optional<VerificationToken> findByUserId(Long userId);
    void deleteByUserId(Long userId);

    @Query("SELECT vt.user.id FROM VerificationToken vt WHERE vt.expiresAt < :now")
    List<Long> findExpiredTokenUserIds(@Param("now") LocalDateTime now);
}
```

#### Service: VerificationTokenService

```java
@Service
public class VerificationTokenService {

    private final VerificationTokenRepo tokenRepo;

    // Создать токен для пользователя
    public VerificationToken createToken(User user) {
        // Удалить старый токен если есть
        tokenRepo.findByUserId(user.getId())
            .ifPresent(tokenRepo::delete);

        VerificationToken token = new VerificationToken();
        token.setUser(user);
        return tokenRepo.save(token);
    }

    // Верифицировать по токену
    @Transactional
    public void verifyToken(String token) {
        VerificationToken vt = tokenRepo.findByToken(token)
            .orElseThrow(() -> new InvalidTokenException("verification"));

        if (vt.isExpired()) {
            throw new VerificationTokenExpiredException();
        }

        User user = vt.getUser();
        user.setIsEmailVerified(true);
        tokenRepo.delete(vt);
    }

    // Получить просроченные user ID
    public List<Long> findExpiredTokenUserIds() {
        return tokenRepo.findExpiredTokenUserIds(LocalDateTime.now());
    }
}
```

---

### 4. Email Verification Endpoint

#### EmailVerificationController

```java
@RestController
@RequestMapping("/auth")
public class EmailVerificationController {

    private final VerificationTokenService tokenService;

    @GetMapping("/verify")
    public ResponseEntity<Void> verifyEmail(
            @RequestParam String token,
            @RequestParam(defaultValue = "http://localhost:5173") String redirectUrl
    ) {
        tokenService.verifyToken(token);
        // Редирект на фронтенд с флагом успеха
        return ResponseEntity.status(HttpStatus.FOUND)
            .location(URI.create(redirectUrl + "/verify?status=success"))
            .build();
    }
}
```

**Response:**
- `302 Found` → редирект на `http://localhost:5173/verify?status=success`
- `400 Bad Request` → InvalidTokenException (токен не найден или malformed)
- `410 Gone` → VerificationTokenExpiredException (токен истёк)

---

### 5. Scheduled Cleanup — Удаление неподтверждённых аккаунтов

#### UnverifiedUserCleanupService

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UnverifiedUserCleanupService {

    private final UserRepo userRepo;
    private final VerificationTokenService tokenService;

    @Scheduled(cron = "0 0 * * * ?") // Каждый час
    @Transactional
    public void cleanupUnverifiedUsers() {
        LocalDateTime cutoff = LocalDateTime.now().minusHours(24);

        List<User> unverified = userRepo
            .findByIsEmailVerifiedFalseAndCreatedAtBefore(cutoff);

        if (unverified.isEmpty()) {
            log.debug("No unverified users to clean up");
            return;
        }

        List<Long> deletedIds = new ArrayList<>();
        for (User user : unverified) {
            userRepo.delete(user);
            deletedIds.add(user.getId());
            log.info("Deleted unverified user: id={}, email={}, createdAt={}",
                user.getId(), user.getEmail(), user.getCreatedAt());
        }

        log.info("Cleanup completed: deleted {} unverified users", deletedIds.size());
    }
}
```

#### UserRepo — дополнительный метод

```java
public interface UserRepo extends JpaRepository<User, Long> {
    // ... существующие методы ...

    @Query("SELECT u FROM User u WHERE u.isEmailVerified = false AND u.createdAt < :cutoff")
    List<User> findByIsEmailVerifiedFalseAndCreatedAtBefore(@Param("cutoff") LocalDateTime cutoff);
}
```

---

### 6. Email Template — Письмо верификации

#### Формат письма

**Subject:** `Подтвердите ваш аккаунт в Library`

**Body (HTML):**
```html
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background: #2c3e50; color: white; padding: 20px; text-align: center; }
        .content { padding: 30px 20px; background: #f9f9f9; }
        .button {
            display: inline-block;
            padding: 14px 30px;
            background: #3498db;
            color: white;
            text-decoration: none;
            border-radius: 5px;
            font-size: 16px;
            margin: 20px 0;
        }
        .button:hover { background: #2980b9; }
        .footer { padding: 20px; text-align: center; color: #999; font-size: 12px; }
        .warning { color: #e74c3c; font-size: 13px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Library</h1>
        </div>
        <div class="content">
            <p>Здравствуйте, <strong>{username}</strong>!</p>
            <p>Спасибо за регистрацию в Library. Для завершения регистрации подтвердите ваш email, нажав на кнопку ниже:</p>

            <p style="text-align: center;">
                <a href="{verificationUrl}" class="button">Подтвердить email</a>
            </p>

            <p>Или перейдите по ссылке:</p>
            <p style="word-break: break-all; color: #3498db;">{verificationUrl}</p>

            <p class="warning">⚠️ Ссылка действительна 24 часа. Если вы не зарегистрировались в Library, проигнорируйте это письмо.</p>
        </div>
        <div class="footer">
            <p>Library Management System</p>
            <p>Это автоматическое письмо, не отвечайте на него.</p>
        </div>
    </div>
</body>
</html>
```

**Body (plain-text fallback):**
```
Здравствуйте, {username}!

Спасибо за регистрацию в Library. Для завершения регистрации перейдите по ссылке:

{verificationUrl}

Ссылка действительна 24 часа.
Если вы не регистрировались в Library, проигнорируйте это письмо.

Library Management System
```

#### Обновлённый EmailNotificationConsumer

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class EmailNotificationConsumer {

    private final JavaMailSender mailSender;
    private final VerificationTokenService tokenService;

    @Value("${app.frontend.url:http://localhost:5173}")
    private String frontendUrl;

    @Value("${app.base-url:http://localhost:8080}")
    private String baseUrl;

    @RabbitListener(queues = RabbitMQConfig.QUEUE_EMAIL)
    public void handleUserRegistered(UserRegisteredEvent event) {
        log.info("Received UserRegisteredEvent: userId={}, email={}", event.userId(), event.email());

        try {
            // Создаём токен верификации
            User user = ...; // загрузить из БД по userId
            VerificationToken token = tokenService.createToken(user);

            String verificationUrl = baseUrl + "/auth/verify?token=" + token.getToken();

            // HTML письмо
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");

            helper.setTo(event.email());
            helper.setSubject("Подтвердите ваш аккаунт в Library");
            helper.setText(buildHtmlEmail(event.username(), verificationUrl), true);

            mailSender.send(message);
            log.info("Verification email sent to: {}", event.email());
        } catch (Exception e) {
            log.error("Failed to send verification email to {}: {}", event.email(), e.getMessage());
            throw e; // Для retry через RabbitMQ DLQ
        }
    }

    private String buildHtmlEmail(String username, String verificationUrl) {
        // HTML template с подстановкой username и verificationUrl
    }
}
```

---

## Полный процесс регистрации

### Шаг 1: Регистрация

```
POST /auth/register
Content-Type: application/json

{
  "email": "user@gmail.com",
  "username": "johndoe",
  "password": "password123",
  "firstName": "John",
  "lastName": "Doe"
}
```

**Что происходит:**
1. `AuthService.register()` → `UserService.register()`
2. Создаётся пользователь с `isEmailVerified = false`
3. Назначается роль `ROLE_USER`
4. Публикуется `UserRegisteredEvent` в RabbitMQ
5. Возвращаются JWT токены (access + refresh)

**Response `201 Created`:**
```json
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "eyJhbGciOi...",
  "tokenType": "Bearer",
  "expiresIn": 86400000
}
```

### Шаг 2: Отправка email

1. `UserEventPublisher` отправляет событие в `library.notification` exchange
2. `EmailNotificationConsumer` получает событие из очереди
3. Создаётся `VerificationToken` (UUID, expires через 24h)
4. Отправляется HTML-письмо с кнопкой верификации

### Шаг 3: Верификация

Пользователь получает письмо и кликает кнопку **"Подтвердить email"**.

```
GET /auth/verify?token=550e8400-e29b-41d4-a716-446655440000
```

**Что происходит:**
1. Поиск токена в БД
2. Проверка срока действия (24h)
3. Если валиден → `user.isEmailVerified = true`, токен удаляется
4. Редирект `302` на фронтенд: `http://localhost:5173/verify?status=success`

**Response `302 Found`:**
```
Location: http://localhost:5173/verify?status=success
```

**Ошибки:**
| Код | Причина |
|-----|---------|
| `400 Bad Request` | Токен не найден или malformed |
| `410 Gone` | Токен истёк (24h прошло) |

### Шаг 4: Ограничения для неподтверждённых пользователей

Пока `isEmailVerified = false`:
- Пользователь **может** логиниться и получать токены
- Пользователь **может** просматривать книги
- Пользователь **не может** брать книги в аренду (будущий функционал)
- При смене email → `isEmailVerified` сбрасывается в `false`

### Шаг 5: Автоматическая очистка

**Расписание:** каждый час (`0 0 * * * ?`)

**Логика:**
1. Найти всех пользователей где `isEmailVerified = false` И `createdAt < now - 24h`
2. Удалить их из БД (вместе с ролями и токенами)
3. Логировать результат

**Пример лога:**
```
INFO  UnverifiedUserCleanupService - Deleted unverified user: id=42, email=test@test.com, createdAt=2026-07-02T10:00:00
INFO  UnverifiedUserCleanupService - Cleanup completed: deleted 3 unverified users
```

---

## База данных

### Новая таблица: verification_tokens

```sql
CREATE TABLE verification_tokens (
    token VARCHAR(36) PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_verification_tokens_expires_at ON verification_tokens(expires_at);
```

| Поле | Тип | Описание |
|------|-----|----------|
| `token` | UUID (VARCHAR 36) | Уникальный токен верификации |
| `user_id` | BIGINT | Связь с users (1:1) |
| `expires_at` | TIMESTAMP | Срок действия (created_at + 24h) |
| `created_at` | TIMESTAMP | Время создания |

### Flyway миграция

```sql
-- V5__create_verification_tokens.sql
CREATE TABLE IF NOT EXISTS verification_tokens (
    token VARCHAR(36) PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_verification_tokens_expires_at
    ON verification_tokens(expires_at);
```

---

## Конфигурация

### application.yml — новые свойства

```yaml
app:
  frontend:
    url: ${FRONTEND_URL:http://localhost:5173}
  base-url: ${APP_BASE_URL:http://localhost:8080}
  email:
    from: ${EMAIL_FROM:noreply@library.com}
    verification-token-ttl-hours: 24
  cleanup:
    unverified-users-cron: "0 0 * * * ?"
```

### application-test.yml — Gmail SMTP

```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${GMAIL_USERNAME}
    password: ${GMAIL_APP_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
```

### Docker Compose — MailHog для dev

```yaml
services:
  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI
```

---

## Security

### JWT и верификация

- Access/Refresh токены выдаются **сразу** при регистрации (независимо от верификации)
- `isEmailVerified` проверяется при **чувствительных операциях** (смена пароля, аренда книг)
- При смене email → `isEmailVerified` сбрасывается → новый токен → новое письмо

### Токен верификации

- Генерируется как UUID v4 (36 символов)
- Хранится в БД (не в JWT)
- Одноразовый — удаляется после использования
- Срок действия: 24 часа
- Один токен на пользователя (при повторной отправке — старый удаляется)

### Защита от abuse

- Rate limiting на `/auth/verify` (Bucket4j)
- Rate limiting на повторную отправку письма верификации
- Логирование всех попыток верификации

---

## Обработка ошибок

### EmailNotificationConsumer — retry

При ошибке отправки письма:
1. RabbitMQ автоматически retry (message не ack-нут)
2. После N попыток → Dead Letter Queue (`library.notification.email.dlq`)
3. DLQ messages можно обработать вручную или через отдельный consumer

### Верификация — коды ошибок

| HTTP Status | Исключение | Описание |
|-------------|-----------|----------|
| `400 Bad Request` | `InvalidTokenException` | Токен не найден или malformed |
| `410 Gone` | `VerificationTokenExpiredException` | Токен истёк (24h прошло) |
| `403 Forbidden` | `AccountNotVerifiedException` | Аккаунт не подтверждён (при чувствительных операциях) |

---

## Новые файлы (для реализации)

| Файл | Назначение |
|------|-----------|
| `entity/VerificationToken.java` | Entity токена верификации |
| `repository/VerificationTokenRepo.java` | JPA репозиторий |
| `service/VerificationTokenService.java` | Создание, верификация, очистка |
| `service/UnverifiedUserCleanupService.java` | Scheduled job удаления |
| `controller/EmailVerificationController.java` | GET /auth/verify endpoint |
| `config/EmailConfig.java` | Настройки email (from, templates) |
| `dto/VerificationResponse.java` | DTO ответа верификации |
| `db/migration/V5__create_verification_tokens.sql` | Flyway миграция |

---

## Существующие файлы (уже есть)

| Файл | Роль в системе |
|------|---------------|
| `config/RabbitMQConfig.java` | Exchange, queue, binding для уведомлений |
| `notification/publisher/UserEventPublisher.java` | Публикация UserRegisteredEvent |
| `notification/consumer/EmailNotificationConsumer.java` | Приём события, отправка email (требует доработки) |
| `notification/event/UserRegisteredEvent.java` | DTO события |
| `user/entity/User.java` | Поле `isEmailVerified` уже есть |
| `user/common/exceptions/InvalidTokenException.java` | Исключение для invalid токена |
| `user/common/exceptions/VerificationTokenExpiredException.java` | Исключение для истёкшего токена |
| `user/common/exceptions/AccountNotVerifiedException.java` | Исключение для неподтверждённого аккаунта |
| `user/common/handler/UserExceptionHandler.java` | Обработка всех вышеуказанных исключений |

---

## Тестирование

### Unit-тесты

| Класс | Что тестировать |
|-------|----------------|
| `VerificationTokenServiceTest` | Создание токена, верификация, истечение, повторная генерация |
| `UnverifiedUserCleanupServiceTest` | Удаление просроченных, сохранение свежих, пустой список |
| `EmailNotificationConsumerTest` | Отправка письма, создание токена, обработка ошибок |

### Integration-тесты

| Сценарий | Описание |
|----------|----------|
| Полный flow регистрации | Register → Email → Verify → isEmailVerified = true |
| Истечение токена | Register → wait 24h → verify → 410 Gone |
| Очистка неподтверждённых | Register → wait 24h → cleanup → user deleted |
| Повторная регистрация | Register → delete → register again → новый токен |

### Ручное тестирование (MailHog)

1. Запустить `docker compose up`
2. Зарегистрировать пользователя через `/auth/register`
3. Открыть `http://localhost:8025` — письмо должно появиться
4. Кликнуть ссылку верификации из письма
5. Проверить `isEmailVerified = true` в БД

---

## Troubleshooting

### Письма не отправляются

1. Проверить SMTP настройки: `MAIL_HOST`, `MAIL_PORT`, `GMAIL_USERNAME`, `GMAIL_APP_PASSWORD`
2. Для Gmail: убедиться что включена 2FA и создан App Password
3. Проверить RabbitMQ: очередь `library.notification.email` существует?
4. Проверить логи: `EmailNotificationConsumer` получает события?

### Токен не работает

1. Проверить что таблица `verification_tokens` создана (Flyway миграция V5)
2. Проверить что токен не истёк (24h)
3. Проверить что токен не был использован ранее

### Очистка не работает

1. Проверить что `@EnableScheduling` активен (`LibraryApplication.java`)
2. Проверить cron выражение: `0 0 * * * ?` (каждый час)
3. Проверить что в БД есть пользователи с `isEmailVerified = false` и `createdAt < 24h ago`
