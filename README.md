# Backend Architecture for Hybrid Photo School Platform

> Основано на SRS (IsProject.pdf), с учетом требований гибридных курсов, аренды оборудования, коммуникаций и платежей.

## 1. Архитектура и микросервисы

### 1.1 Состав микросервисов
- **api-gateway** (опциональный, но рекомендованный): единая точка входа, rate limiting, CORS, маршрутизация, TLS offload.
- **auth-service**: JWT-аутентификация, refresh-токены, выдача/ревокация, 2FA для преподавателей.
- **user-service**: профили, роли (student, teacher, admin, equipment_manager), permissions, предпочтения уведомлений.
- **course-service**: курсы (онлайн/оффлайн/гибрид), модули, учебные материалы (метаданные), статусы курсов.
- **schedule-service**: расписание занятий, слоты (онлайн/оффлайн), запись на слот, предотвращение двойных бронирований.
- **equipment-service**: каталог оборудования, статусы доступности, бронирования/возвраты, обслуживание.
- **communication-service**: чат, групповые беседы, файлы, системные уведомления, интеграция с email/видеозвонками (через внешние провайдеры).
- **payment-service**: платежи за курсы/аренду, счета, статусы, webhooks от провайдеров.
- **notification-service** (выделенный воркер): email/sms/push/system notifications, шаблоны, ретраи.
- **report-service** (опциональный): агрегации, аналитика посещаемости/платежей.

### 1.2 Текстовая диаграмма взаимодействий
- БД:
  - auth-service: PostgreSQL `auth_db` (refresh tokens, 2FA secrets, revoked tokens);
  - user-service: PostgreSQL `user_db`;
  - course-service: PostgreSQL `course_db`;
  - schedule-service: PostgreSQL `schedule_db`;
  - equipment-service: PostgreSQL `equipment_db`;
  - communication-service: PostgreSQL `comm_db` (чат, сообщения);
  - payment-service: PostgreSQL `payment_db`;
  - notification-service: PostgreSQL `notification_db` (шаблоны, логи).
- REST (sync):
  - api-gateway → все сервисы.
  - auth-service ↔ user-service (валидация пользователей при выдаче токена).
  - course-service ↔ user-service (информация о преподавателях/студентах).
  - schedule-service ↔ course-service (слоты привязаны к курсам/преподавателям).
  - equipment-service ↔ schedule-service (проверка занятости для слотов офлайн).
  - payment-service → course-service / equipment-service (проверка суммы/элемента).
- Kafka (async):
  - `user.registered` (auth-service → notification-service, user-service).
  - `course.created` (course-service → schedule-service, notification-service).
  - `course.enrollment.created` (course-service → payment-service, notification-service).
  - `schedule.slot.booked` (schedule-service → notification-service, equipment-service).
  - `equipment.booking.created` (equipment-service → payment-service, notification-service).
  - `payment.completed` (payment-service → course-service, equipment-service, notification-service).
  - `notification.send-requested` (producers: course/schedule/equipment/payment-service; consumer: notification-service).

### 1.3 Bounded Contexts
- **Identity & Access (auth-service, user-service)**: учетные записи, роли, 2FA, токены. Четко отделены для безопасности и масштабирования.
- **Learning (course-service, schedule-service)**: курсы, материалы, слоты; совместный контекст из-за сильной связи расписания с курсами.
- **Equipment Rental (equipment-service)**: каталог, доступность, бронирования, обслуживание; независимая модель инвентаря.
- **Communication (communication-service, notification-service)**: чат (реальное время) отделен от асинхронных уведомлений (email/system), что упрощает масштабирование воркеров.
- **Payments & Billing (payment-service)**: эквайринг, статусы, счета; держит инварианты денег отдельно.
- **Reporting (report-service)**: только чтение/агрегации, потребляет события.

### 1.4 Потоки данных (ключевые сценарии)
1. **Регистрация/логин**:
   - UI → api-gateway → auth-service (POST /register) → user-service (создать профиль) → Kafka `user.registered` → notification-service (email подтверждение).
   - Логин: UI → auth-service (POST /login) → валидация пользователя в user-service → выдача JWT + refresh; опционально 2FA challenge.
2. **Создание курса и запись**:
   - Teacher UI → course-service (POST /courses) → Kafka `course.created` → notification-service.
   - Student UI → course-service (POST /courses/{id}/enroll) → создает Enrollment (pending) → Kafka `course.enrollment.created` → payment-service создает invoice → после `payment.completed` course-service подтверждает enrollment и шлет `notification.send-requested`.
3. **Бронирование оборудования**:
   - UI → equipment-service (POST /equipment/{id}/book) → проверка доступности/пересечений → резерв → Kafka `equipment.booking.created` → payment-service (счет) и notification-service.
4. **Бронирование слота занятия**:
   - UI → schedule-service (POST /slots/{id}/book) → проверка курса/преподавателя через course-service → блокировка слота → событие `schedule.slot.booked` → notification-service; при офлайн-слоте equipment-service может зарезервировать аудиторию/оборудование.
5. **Платеж**:
   - payment-service создает Payment/Invoice → редирект/checkout → webhook провайдера → payment-service обновляет статус → Kafka `payment.completed`/`payment.failed` → course-service/equipment-service подтверждают/отменяют брони, notification-service шлет письма.
6. **Уведомления**:
   - Любой сервис публикует `notification.send-requested` с типом (email/system/push) → notification-service рендерит шаблон, отправляет через провайдер, логирует результат.

### 1.5 Соответствие ключевым требованиям SRS
- **FR-001/FR-002 (регистрация/аутентификация, 2FA для преподавателей)** → auth-service (JWT + refresh + optional TOTP), user-service (email verification, парольный сброс), Kafka `user.registered` для email-подтверждения.
- **FR-010/FR-011 (курсы, запись)** → course-service (CRUD, материалы, статусы pending/confirmed/completed), schedule-service (слоты), payment-service (взимание платы за курсы), notification-service (напоминания о занятиях).
- **FR-020/FR-021 (каталог и бронирование оборудования)** → equipment-service (фильтры по типу, интеграция с расписанием офлайн-слотов), payment-service (аренда), notification-service (напоминания о возврате, EI-001).
- **FR-030/FR-031 (чат и видеоконференции до 50 участников)** → communication-service (чат, файлы, групповые беседы, интеграция с внешним video provider; хранит ссылки на сессии/записи), notification-service (уведомления о запланированных звонках).
- **FR-040/FR-041 (календарь доступности и бронирование занятий)** → schedule-service (timeslot + booking с защитой от двойного бронирования), связка с course-service и equipment-service.
- **UI-001/UI-002** → публичный каталог курсов/оборудования, личный кабинет (user-service + course-service + schedule-service + equipment-service + communication-service виджеты), кэширование популярного каталога.
- **CI-002 (платежи)** → payment-service с защищенным webhook’ом, интеграцией с course/equipment, invoice/receipt.
- **PER/REL/SEC/USA/MAN**: 
  - Производительность: кэш Redis для публичных каталогов и расписаний, пагинация, CDN для статики/материалов, горизонтальное масштабирование сервисов и Kafka.
  - Надежность: multi-AZ PostgreSQL, Kubernetes HPA, readiness/liveness, retry + DLQ в Kafka, резервное копирование.
  - Безопасность: TLS 1.2+, парольный хэш (BCrypt/Argon2), RBAC, аудит (audit log таблицы/события), GDPR/152-ФЗ (опциональный masking/consent), 2FA.
  - Доступность/удобство: WCAG AA закладывается на фронте; на бэкенде — адаптивные DTO, мультиязычные сообщения ошибок.

## 2. Модель данных (ER по сервисам)

### 2.1 auth-service
- `user_auth(id, user_id, password_hash, two_factor_enabled, two_factor_secret, last_login_at, status)`
- `refresh_token(id, user_id, token, expires_at, revoked, created_at)`
- `revoked_token(jti, user_id, revoked_at)`
- Связи: user_auth 1:1 user (в user-service), refresh_token ManyToOne user.

### 2.2 user-service
- `user(id, email, phone, first_name, last_name, avatar_url, status, created_at, updated_at)`
- `role(id, code, description)`
- `permission(id, code, description)`
- `user_role(user_id, role_id)` (ManyToMany)
- `role_permission(role_id, permission_id)` (ManyToMany)
- Индексы: email unique, phone unique, status, role.code.

### 2.3 course-service
- `course(id, title, description, format, level, price, status, teacher_id, start_date, end_date, created_at)`
- `module(id, course_id, title, order_index, content_url)`
- `lesson(id, module_id, title, duration_minutes, video_url, material_url)`
- `enrollment(id, course_id, student_id, status, created_at, updated_at, payment_id)`
- Связи: course 1:N module; module 1:N lesson; course 1:N enrollment; user (teacher_id/student_id) через user-service (foreign key logical).
- Индексы: course.status, course.format, enrollment.status, teacher_id, student_id.

### 2.4 schedule-service
- `timeslot(id, course_id, teacher_id, location_type, location_value, start_at, end_at, capacity, status)`
- `slot_booking(id, timeslot_id, student_id, status, created_at, payment_id)`
- Связи: timeslot 1:N slot_booking; timeslot → course_service (course_id), teacher_id logical FK.
- Индексы: timeslot.start_at, end_at, status; slot_booking.status.

### 2.5 equipment-service
- `equipment(id, name, type, condition, location, hourly_price, status, available_from, available_to)`
- `equipment_booking(id, equipment_id, user_id, start_at, end_at, status, payment_id)`
- `maintenance(id, equipment_id, start_at, end_at, description, status)`
- Связи: equipment 1:N equipment_booking; equipment 1:N maintenance.
- Индексы: equipment.type, status; booking(equipment_id, start_at, end_at), maintenance windows.

### 2.6 communication-service
- `chat(id, type, name, created_by)`
- `chat_member(chat_id, user_id, role)`
- `message(id, chat_id, sender_id, content, content_type, created_at, attachment_url)`
- Индексы: chat_member(chat_id, user_id), message.chat_id + created_at.

### 2.7 payment-service
- `payment(id, user_id, amount, currency, status, purpose_type, purpose_id, provider, provider_ref, created_at, updated_at)`
- `invoice(id, payment_id, due_date, status, pdf_url)`
- `refund(id, payment_id, amount, status, created_at)`
- Индексы: payment.status, purpose_type/purpose_id, provider_ref.

### 2.8 notification-service
- `notification_template(id, code, channel, subject, body)`
- `notification_log(id, user_id, channel, payload, status, error, created_at)`
- Индексы: code unique, status, user_id.

## 3. REST API (ключевые эндпоинты)

### 3.1 auth-service
- `POST /api/v1/auth/register` (anonymous): {email, password, role} → 201; ошибки 400/409. Триггер `user.registered`.
- `POST /api/v1/auth/login` (anonymous): {email, password, otp?} → {accessToken, refreshToken} 200; 401/403.
- `POST /api/v1/auth/refresh` (authenticated refresh): {refreshToken} → new tokens; 401/403.
- `POST /api/v1/auth/logout` → revoke refresh/JTI.
- `POST /api/v1/auth/2fa/enable` (teacher/admin) → QR secret.

### 3.2 user-service
- `GET /api/v1/users/me` (any auth): профиль.
- `PATCH /api/v1/users/me` (auth): обновление профиля.
- `GET /api/v1/users/{id}` (admin/teacher self for students?)
- `GET /api/v1/users` (admin) с фильтрами status/role.
- `POST /api/v1/users/{id}/roles` (admin) → назначение ролей.

### 3.3 course-service
- `POST /api/v1/courses` (teacher/admin): создание курса.
- `GET /api/v1/courses` (anonymous): каталог с фильтрами format/level/status, кэшируемый.
- `GET /api/v1/courses/{id}` (anonymous/auth): детали.
- `PATCH /api/v1/courses/{id}` (teacher-owner/admin): обновление.
- `POST /api/v1/courses/{id}/enroll` (student): создает enrollment (pending).
- `POST /api/v1/courses/{id}/materials` (teacher): загрузка метаданных файла.
- `GET /api/v1/enrollments` (student/teacher/admin): фильтр по статусу.

### 3.4 schedule-service
- `POST /api/v1/slots` (teacher/admin): создать слот (онлайн/оффлайн, capacity).
- `GET /api/v1/slots` (auth, anonymous ограниченно): список по дате/курсу.
- `POST /api/v1/slots/{id}/book` (student): бронь слота.
- `PATCH /api/v1/slots/{id}` (teacher/admin): изменение слота.
- `GET /api/v1/slots/{id}/bookings` (teacher/admin): список бронирований.

### 3.5 equipment-service
- `GET /api/v1/equipment` (anonymous): каталог, фильтры type/status.
- `GET /api/v1/equipment/{id}/availability` (auth): проверка по диапазону дат.
- `POST /api/v1/equipment/{id}/book` (auth, equipment_manager/admin approve): создает бронь.
- `PATCH /api/v1/equipment-bookings/{id}` (equipment_manager/admin): approve/cancel/return.
- `POST /api/v1/equipment` (equipment_manager/admin): CRUD оборудования.

### 3.6 communication-service
- `GET /api/v1/chats` (auth): списки чатов.
- `POST /api/v1/chats` (auth): создать приват/групповой.
- `POST /api/v1/chats/{id}/messages` (auth member): отправка сообщения.
- `GET /api/v1/chats/{id}/messages` (auth member): история с пагинацией.
- `POST /api/v1/video-sessions` (teacher/admin): создание видеосессии (courseId/slotId, startAt, maxParticipants≤50), возвращает joinUrl/recordingFlag.
- `POST /api/v1/video-sessions/{id}/join` (auth member): выдает одноразовый токен в провайдер, фиксирует audit log.
- `GET /api/v1/video-sessions/{id}/recordings` (teacher/admin): ссылки на записи (если включена запись), метаданные.

### 3.7 payment-service
- `POST /api/v1/payments` (auth): {purposeType: COURSE|EQUIPMENT|SLOT, purposeId, amount} → Payment + redirect url.
- `GET /api/v1/payments/{id}` (auth): статус.
- `POST /api/v1/payments/webhook/{provider}` (public, signed): обработка статусов.
- `POST /api/v1/refunds` (admin): возвраты.

### 3.8 notification-service
- `POST /api/v1/notifications/test` (admin): отправка теста.
- `GET /api/v1/notifications/logs` (admin): фильтр по статусу/пользователю.

## 4. Kafka и событийная модель

Топики:
- `user.registered`
- `course.created`
- `course.enrollment.created`
- `schedule.slot.booked`
- `equipment.booking.created`
- `communication.video-session.scheduled`
- `communication.video-session.recording-ready`
- `payment.completed`
- `payment.failed`
- `notification.send-requested`

Пример события `course.enrollment.created`:
```json
{
  "eventId": "uuid",
  "occurredAt": "2025-09-22T10:00:00Z",
  "courseId": "uuid",
  "enrollmentId": "uuid",
  "studentId": "uuid",
  "amount": 100.00,
  "currency": "RUB",
  "idempotencyKey": "enroll-uuid",
  "correlationId": "req-uuid"
}
```
- Публикатор: course-service. Подписчики: payment-service (создание счета), notification-service (письмо), report-service.
- Инварианты: idempotencyKey для защиты от повторов; correlationId для трассировки.

`payment.completed`:
```json
{
  "eventId": "uuid",
  "paymentId": "uuid",
  "purposeType": "COURSE",
  "purposeId": "uuid",
  "status": "COMPLETED",
  "amount": 100.00,
  "currency": "RUB",
  "userId": "uuid",
  "provider": "tinkoff",
  "occurredAt": "2025-09-22T10:05:00Z",
  "correlationId": "req-uuid"
}
```
- Публикатор: payment-service. Подписчики: course-service, equipment-service, schedule-service (если оплачиваем слот), notification-service.
- Инварианты: идемпотентность по paymentId + status, trace через correlationId.

`communication.video-session.scheduled`:
```json
{
  "eventId": "uuid",
  "sessionId": "uuid",
  "courseId": "uuid",
  "slotId": "uuid",
  "startAt": "2025-09-22T11:00:00Z",
  "maxParticipants": 50,
  "recordingEnabled": true,
  "createdBy": "teacher-uuid",
  "correlationId": "req-uuid"
}
```
- Публикатор: communication-service при создании видеосессии (FR-031). Подписчики: notification-service (рассылка ссылок/напоминаний), schedule-service (синхронизация со слотом), report-service.

`communication.video-session.recording-ready`:
```json
{
  "eventId": "uuid",
  "sessionId": "uuid",
  "recordingUrl": "https://.../recording.mp4",
  "durationSeconds": 3600,
  "availableUntil": "2025-10-01T00:00:00Z",
  "correlationId": "provider-callback"
}
```
- Публикатор: communication-service по webhook’у от video-провайдера. Подписчики: notification-service (отправка ссылок студентам), course-service (сохранить ссылку в материалах), report-service (аналитика посещаемости).

`notification.send-requested`:
```json
{
  "eventId": "uuid",
  "channel": "EMAIL|SYSTEM",
  "templateCode": "ENROLLMENT_CONFIRMED",
  "userId": "uuid",
  "data": {"courseTitle": "...", "startAt": "..."},
  "priority": "NORMAL|HIGH",
  "correlationId": "req-uuid"
}
```
- Публикатор: бизнес-сервисы. Подписчик: notification-service.

События используются для синхронизации статусов (оплаты → подтверждение), триггера уведомлений и фоновых задач (рендер писем, отчетность). Все слушатели должны быть идемпотентными.

## 5. Безопасность и JWT

- JWT access token: клеймы `sub` (userId), `email`, `roles`, `permissions`, `jti`, `iat`, `exp`, `iss`, `aud`, `cid` (correlation/request id). TTL access: 15 мин. Refresh token: 14 дней, хранится в БД auth-service; revocation list по `jti`.
- 2FA (TOTP) для teacher/admin: при login выдача `mfa_required` и временный challenge; отдельный endpoint для подтверждения OTP.

### Пример конфигурации Spring Security (auth-service)
```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http, JwtAuthFilter jwtFilter) throws Exception {
    http.csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/v1/auth/login", "/api/v1/auth/register", "/api/v1/auth/refresh", "/api/v1/auth/webhook/**").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/v1/auth/2fa/enable").hasAnyRole("TEACHER", "ADMIN")
            .anyRequest().authenticated())
        .exceptionHandling(ex -> ex.authenticationEntryPoint(new JwtAuthEntryPoint())
            .accessDeniedHandler(new JwtAccessDeniedHandler()))
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
}
```
- Генерация токена: с использованием `io.jsonwebtoken` (HS512/RS256); `jti` сохраняется для трекинга.
- Валидация: фильтр читает Authorization Bearer, проверяет подпись, срок, наличие в blacklist.
- Ошибки 401/403 обрабатываются custom handlers, возвращающие JSON.

## 6. Кэширование (Redis)

- Кэшируются публичные каталоги: список курсов `cache:course:list:{filters}` TTL 10 мин; каталог оборудования `cache:equipment:list:{filters}` TTL 10 мин; расписание слотов по дате `cache:slots:{date}` TTL 5 мин.
- Инвалидация: при CRUD курс/оборудование/слот публикуется событие (Kafka) и сервис чистит кэш через `@CacheEvict` или listener.

Пример конфигурации:
```java
@EnableCaching
@Configuration
public class RedisConfig {
  @Bean
  public RedisConnectionFactory redisConnectionFactory() {
      return new LettuceConnectionFactory();
  }
  @Bean
  public CacheManager cacheManager(RedisConnectionFactory cf) {
      return RedisCacheManager.builder(cf)
        .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(10)))
        .build();
  }
}
```
Использование:
```java
@Cacheable(value = "course:list", key = "#filter.hashCode()")
public Page<CourseDto> findCourses(CourseFilter filter, Pageable pageable) { ... }

@CacheEvict(value = "course:list", allEntries = true)
public CourseDto updateCourse(UUID id, CourseUpdateRequest req) { ... }
```

## 7. Инфраструктура

### 7.1 Структура репозитория
- Multi-repo по сервисам или mono-repo с каталогами:
```
root/
  auth-service/
  user-service/
  course-service/
  schedule-service/
  equipment-service/
  communication-service/
  payment-service/
  notification-service/
  report-service/
  common-libs/ (DTO, util, security)
```

### 7.2 Пример Dockerfile (для course-service)
```Dockerfile
# Build stage
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./mvnw -DskipTests clean package

# Runtime stage
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/course-service.jar ./course-service.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/course-service.jar"]
```

### 7.3 Примеры Kubernetes манифестов
`Deployment` + `Service` (course-service):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: course-service
spec:
  replicas: 3
  selector:
    matchLabels: { app: course-service }
  template:
    metadata:
      labels: { app: course-service }
    spec:
      containers:
      - name: course-service
        image: registry.example.com/course-service:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: prod
        - name: DB_URL
          valueFrom:
            secretKeyRef:
              name: course-db-secret
              key: url
        - name: JWT_PUBLIC_KEY
          valueFrom:
            secretKeyRef:
              name: jwt-keys
              key: public
        readinessProbe:
          httpGet: { path: /actuator/health, port: 8080 }
        livenessProbe:
          httpGet: { path: /actuator/health/liveness, port: 8080 }
---
apiVersion: v1
kind: Service
metadata:
  name: course-service
spec:
  selector: { app: course-service }
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```
`ConfigMap`/`Secret` пример:
```yaml
apiVersion: v1
kind: Secret
metadata: { name: jwt-keys }
type: Opaque
data:
  public: <base64-key>
  private: <base64-key>
---
apiVersion: v1
kind: ConfigMap
metadata: { name: kafka-config }
data:
  KAFKA_BOOTSTRAP: kafka:9092
```

### 7.4 Логирование и мониторинг
- Structured logging (JSON) с полями `correlationId`, `userId`, `service`.
- Spring Boot Actuator + Micrometer → Prometheus; алерты в Grafana.
- Tracing: OpenTelemetry (OTLP) → Jaeger/Tempo; correlationId прокидывается в заголовках (X-Correlation-Id).

## 8. Примеры кода (course-service)

### 8.1 Application класс
```java
@SpringBootApplication
@EnableJpaRepositories
@EnableCaching
public class CourseServiceApplication {
  public static void main(String[] args) {
    SpringApplication.run(CourseServiceApplication.class, args);
  }
}
```

### 8.2 Entities
```java
@Entity
@Table(name = "course")
public class Course {
  @Id @GeneratedValue(strategy = GenerationType.UUID)
  private UUID id;
  private String title;
  @Enumerated(EnumType.STRING)
  private CourseFormat format; // ONLINE, OFFLINE, HYBRID
  private BigDecimal price;
  private String level;
  @Enumerated(EnumType.STRING)
  private CourseStatus status; // DRAFT, PUBLISHED, ARCHIVED
  private UUID teacherId; // reference to user-service
  private LocalDate startDate;
  private LocalDate endDate;
  @OneToMany(mappedBy = "course", cascade = CascadeType.ALL, orphanRemoval = true)
  private List<Module> modules = new ArrayList<>();
}

@Entity
@Table(name = "enrollment")
public class Enrollment {
  @Id @GeneratedValue(strategy = GenerationType.UUID)
  private UUID id;
  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "course_id")
  private Course course;
  private UUID studentId;
  @Enumerated(EnumType.STRING)
  private EnrollmentStatus status; // PENDING, CONFIRMED, CANCELLED
  private UUID paymentId;
  private Instant createdAt;
}
```

### 8.3 DTO & Controllers
```java
@RestController
@RequestMapping("/api/v1/courses")
@RequiredArgsConstructor
public class CourseController {
  private final CourseService courseService;

  @GetMapping
  public Page<CourseResponse> list(CourseFilter filter, Pageable pageable) {
    return courseService.find(filter, pageable);
  }

  @PostMapping
  @PreAuthorize("hasAnyRole('TEACHER','ADMIN')")
  public ResponseEntity<CourseResponse> create(@Valid @RequestBody CourseCreateRequest req) {
    return ResponseEntity.status(HttpStatus.CREATED).body(courseService.create(req));
  }

  @PostMapping("/{id}/enroll")
  @PreAuthorize("hasRole('STUDENT')")
  public ResponseEntity<EnrollmentResponse> enroll(@PathVariable UUID id) {
    return ResponseEntity.status(HttpStatus.CREATED).body(courseService.enroll(id));
  }
}
```

### 8.4 Service layer
```java
public interface CourseService {
  Page<CourseResponse> find(CourseFilter filter, Pageable pageable);
  CourseResponse create(CourseCreateRequest req);
  EnrollmentResponse enroll(UUID courseId);
}

@Service
@RequiredArgsConstructor
@Transactional
public class CourseServiceImpl implements CourseService {
  private final CourseRepository courseRepository;
  private final EnrollmentRepository enrollmentRepository;
  private final KafkaTemplate<String, Object> kafkaTemplate;
  private final AuthContext auth;

  @Override
  @Transactional(readOnly = true)
  @Cacheable(value = "course:list", key = "#filter.hashCode() + '-' + #pageable.pageNumber")
  public Page<CourseResponse> find(CourseFilter filter, Pageable pageable) {
    return courseRepository.findAll(filter.toSpec(), pageable).map(CourseMapper::toResponse);
  }

  @Override
  public CourseResponse create(CourseCreateRequest req) {
    Course course = CourseMapper.fromRequest(req, auth.currentUserId());
    course.setStatus(CourseStatus.DRAFT);
    Course saved = courseRepository.save(course);
    kafkaTemplate.send("course.created", saved.getId().toString(), CourseEvents.created(saved));
    return CourseMapper.toResponse(saved);
  }

  @Override
  public EnrollmentResponse enroll(UUID courseId) {
    Course course = courseRepository.findById(courseId).orElseThrow(() -> new NotFoundException("course"));
    Enrollment enrollment = new Enrollment();
    enrollment.setCourse(course);
    enrollment.setStudentId(auth.currentUserId());
    enrollment.setStatus(EnrollmentStatus.PENDING);
    enrollment.setCreatedAt(Instant.now());
    Enrollment saved = enrollmentRepository.save(enrollment);
    kafkaTemplate.send("course.enrollment.created", saved.getId().toString(), CourseEvents.enrollmentCreated(saved));
    return EnrollmentMapper.toResponse(saved);
  }
}
```

### 8.5 Repositories
```java
public interface CourseRepository extends JpaRepository<Course, UUID>, JpaSpecificationExecutor<Course> {}
public interface EnrollmentRepository extends JpaRepository<Enrollment, UUID> {}
```

### 8.6 Security (resource server)
```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
  @Bean
  SecurityFilterChain filterChain(HttpSecurity http, JwtAuthFilter jwtFilter) throws Exception {
    http.csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
          .requestMatchers(HttpMethod.GET, "/api/v1/courses/**").permitAll()
          .anyRequest().authenticated())
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
  }
}
```

### 8.7 Kafka конфигурация
```java
@Configuration
@EnableKafka
public class KafkaConfig {
  @Bean
  public ProducerFactory<String, Object> producerFactory(KafkaProperties props) {
    Map<String, Object> config = props.buildProducerProperties();
    config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
    return new DefaultKafkaProducerFactory<>(config);
  }

  @Bean
  public KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> pf) {
    return new KafkaTemplate<>(pf);
  }
}
```

### 8.8 Redis использование
```java
@Service
@RequiredArgsConstructor
public class PublicCourseCacheService {
  private final CourseRepository repo;

  @Cacheable(value = "course:list", key = "#filter.hashCode()")
  public Page<CourseResponse> list(CourseFilter filter, Pageable pageable) {
    return repo.findAll(filter.toSpec(), pageable).map(CourseMapper::toResponse);
  }

  @CacheEvict(value = "course:list", allEntries = true)
  public void evictCache() {}
}
```

### 8.9 Миграции (Flyway SQL пример)
```sql
CREATE TABLE course (
  id UUID PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  format VARCHAR(20) NOT NULL,
  price NUMERIC(12,2) NOT NULL,
  level VARCHAR(50),
  status VARCHAR(20) NOT NULL,
  teacher_id UUID NOT NULL,
  start_date DATE,
  end
_date DATE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX idx_course_status ON course(status);
CREATE TABLE enrollment (
  id UUID PRIMARY KEY,
  course_id UUID NOT NULL REFERENCES course(id),
  student_id UUID NOT NULL,
  status VARCHAR(20) NOT NULL,
  payment_id UUID,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX idx_enrollment_status ON enrollment(status);
```

## Допущения
- Видеозвонки реализуются интеграцией с внешним провайдером (WebRTC/SFU), backend хранит только сессионные токены и записи метаданных.
- Платежный провайдер — Tinkoff/Stripe; оплата курсов и аренды идёт одинаковым потоком.
- Хранение учебных материалов/файлов в объектном хранилище (S3-совместимое), ссылки сохраняются в сервисах (course/communication).
- Аутентификация пользователей централизована в auth-service, остальные сервисы выступают как resource servers и доверяют публичному ключу JWT.
