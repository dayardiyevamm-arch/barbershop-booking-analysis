# 4. UML-диаграммы

Все диаграммы выполнены в нотации **UML 2.x** и реализованы через **Mermaid**, который рендерится в GitHub, GitLab, Notion и большинстве современных инструментов.

---

## 4.1. Use Case Diagram — диаграмма вариантов использования

Показывает, кто и что может делать в системе (скоуп клиентской части).

```mermaid
flowchart TB
    Client(("👤 Клиент"))
    Master(("✂️ Мастер"))
    Notifier(("📩 Notification Service"))

    subgraph BarberBook["Система BarberBook"]
        UC1["Просмотреть свободные слоты"]
        UC2["Создать запись"]
        UC3["Отменить запись"]
        UC4["Перенести запись"]
        UC5["Просмотреть мои записи"]
        UC6["Получить уведомление о записи"]
        UC7["Просмотреть своё расписание"]
    end

    Client --- UC1
    Client --- UC2
    Client --- UC3
    Client --- UC4
    Client --- UC5
    Master --- UC7
    UC2 -.->|«include»| UC1
    UC4 -.->|«include»| UC1
    UC2 -.->|«extend»| UC6
    UC3 -.->|«extend»| UC6
    UC4 -.->|«extend»| UC6
    UC6 --- Notifier
```

**Пояснения:**
- **Actor «Клиент»** — первичный актор, все основные сценарии.
- **Actor «Мастер»** — показан для контекста; его функциональность вне скоупа данного ТЗ.
- **Actor «Notification Service»** — внешняя система (SMS/push), вторичный актор.
- **«include»** означает обязательную часть сценария (чтобы создать запись — нужно сначала увидеть слоты).
- **«extend»** означает расширение (уведомление отправляется, но это отдельный шаг).

---

## 4.2. Activity Diagram — процесс бронирования

Показывает пошаговый флоу пользователя при создании записи.

```mermaid
flowchart TD
    Start([Начало]) --> Login{Клиент<br/>авторизован?}
    Login -->|Нет| DoLogin[Авторизация по SMS]
    Login -->|Да| SelectBranch[Выбор филиала]
    DoLogin --> SelectBranch
    SelectBranch --> SelectService[Выбор услуги]
    SelectService --> SelectMaster[Выбор мастера<br/>или «любой»]
    SelectMaster --> SelectDate[Выбор даты]
    SelectDate --> LoadSlots[Загрузка свободных слотов]
    LoadSlots --> HasSlots{Слоты есть?}
    HasSlots -->|Нет| SuggestDate[Предложить другую дату]
    SuggestDate --> SelectDate
    HasSlots -->|Да| PickSlot[Клиент выбирает слот]
    PickSlot --> Confirm[Подтверждение деталей]
    Confirm --> Submit[POST /bookings]
    Submit --> Validate{Слот всё ещё<br/>свободен?}
    Validate -->|Нет| ShowError[Показать «слот занят»,<br/>обновить список]
    ShowError --> LoadSlots
    Validate -->|Да| CreateBooking[Создать запись в БД<br/>статус CONFIRMED]
    CreateBooking --> SendNotification[Отправить уведомление]
    SendNotification --> ShowSuccess[Экран успеха<br/>с ID записи]
    ShowSuccess --> End([Конец])
```

---

## 4.3. Sequence Diagram — создание записи

Показывает взаимодействие компонентов во времени при успешном создании записи.

```mermaid
sequenceDiagram
    participant C as 👤 Клиент<br/>(Mobile App)
    participant API as API Gateway
    participant BS as Booking Service
    participant DB as Database
    participant NS as Notification Service

    C->>API: GET /slots?branch=X&service=Y&date=2026-04-24
    API->>BS: getAvailableSlots(filters)
    BS->>DB: SELECT свободные слоты
    DB-->>BS: Список слотов
    BS-->>API: 200 OK [слоты]
    API-->>C: Показать слоты

    C->>API: POST /bookings {slot, service, master}
    API->>BS: createBooking(request, userId)
    BS->>DB: BEGIN TRANSACTION
    BS->>DB: SELECT ... FOR UPDATE (lock slot)

    alt Слот свободен
        BS->>DB: INSERT booking (status=CONFIRMED)
        BS->>DB: INSERT booking_history (CREATED)
        BS->>DB: COMMIT
        BS->>NS: sendBookingConfirmation(booking)
        NS-->>BS: OK
        BS-->>API: 201 Created {booking}
        API-->>C: Запись создана ✅
    else Слот уже занят
        BS->>DB: ROLLBACK
        BS-->>API: 409 Conflict {SLOT_ALREADY_TAKEN}
        API-->>C: Ошибка: слот занят
    end
```

**Ключевые моменты:**
- Блокировка строки через `SELECT ... FOR UPDATE` защищает от гонки (race condition) при одновременной записи двух клиентов.
- Уведомление отправляется **после** коммита транзакции, чтобы не отправить подтверждение по неудачной записи.

---

## 4.4. Sequence Diagram — отмена записи

```mermaid
sequenceDiagram
    participant C as 👤 Клиент
    participant API as API Gateway
    participant BS as Booking Service
    participant DB as Database
    participant NS as Notification Service

    C->>API: PATCH /bookings/{id}/cancel
    API->>BS: cancelBooking(bookingId, userId)

    BS->>DB: SELECT booking WHERE id = bookingId
    DB-->>BS: booking

    alt Запись не найдена
        BS-->>API: 404 Not Found
        API-->>C: Запись не найдена
    else Запись принадлежит другому клиенту
        BS-->>API: 403 Forbidden
        API-->>C: Нет прав
    else Статус не CONFIRMED
        BS-->>API: 422 INVALID_STATE
        API-->>C: Запись уже отменена/завершена
    else До визита < 2 часов
        BS-->>API: 422 CANCELLATION_WINDOW_EXPIRED
        API-->>C: Поздно отменять, свяжитесь с салоном
    else Всё ок
        BS->>DB: UPDATE booking SET status = CANCELLED
        BS->>DB: INSERT booking_history (CANCELLED)
        BS->>NS: sendCancellationNotice(booking)
        BS-->>API: 200 OK {booking}
        API-->>C: Запись отменена ✅
    end
```

---

## 4.5. Sequence Diagram — перенос записи

Самый сложный сценарий: нужно атомарно освободить старый слот и занять новый.

```mermaid
sequenceDiagram
    participant C as 👤 Клиент
    participant API as API Gateway
    participant BS as Booking Service
    participant DB as Database
    participant NS as Notification Service

    C->>API: PATCH /bookings/{id}/reschedule<br/>{new_start_at, master_id?, service_id?}
    API->>BS: rescheduleBooking(bookingId, newSlot, userId)

    BS->>DB: SELECT booking WHERE id = bookingId
    DB-->>BS: booking

    Note over BS: Валидация:<br/>- существует ли запись<br/>- принадлежит ли пользователю<br/>- статус CONFIRMED<br/>- до визита > 2 часов

    alt Валидация не прошла
        BS-->>API: 4xx Error
        API-->>C: Ошибка
    else Валидация прошла
        BS->>DB: BEGIN TRANSACTION
        BS->>DB: SELECT slot FOR UPDATE (новый слот)

        alt Новый слот занят
            BS->>DB: ROLLBACK
            BS-->>API: 409 SLOT_ALREADY_TAKEN
            API-->>C: Слот занят
        else Новый слот свободен
            BS->>DB: UPDATE booking SET start_at=new, end_at=new<br/>(старый слот автоматически освобождается)
            BS->>DB: INSERT booking_history<br/>(RESCHEDULED, old_value, new_value)
            BS->>DB: COMMIT
            BS->>NS: sendRescheduleNotice(booking)
            BS-->>API: 200 OK {updated booking}
            API-->>C: Перенос выполнен ✅
        end
    end
```

**Почему ID записи не меняется при переносе:**
Это упрощает аналитику и клиентскую часть: запись «живёт» как один объект от создания до завершения/отмены. Вся история переносов — в `BOOKING_HISTORY`.

---

## 4.6. Component Diagram — упрощённая архитектура

Контекстная диаграмма: как Booking Service встраивается в общую систему.

```mermaid
flowchart LR
    subgraph Client["Клиентская часть"]
        Mobile[Mobile App iOS/Android]
        Web[Web App]
    end

    subgraph Backend["Backend"]
        Gateway[API Gateway<br/>+ Auth]
        BookingSvc[Booking Service]
        CatalogSvc[Catalog Service<br/>услуги, мастера, филиалы]
        UserSvc[User Service]
        NotifSvc[Notification Service]
    end

    subgraph Data["Хранилища"]
        BookingDB[(Booking DB<br/>PostgreSQL)]
        CatalogDB[(Catalog DB)]
        UserDB[(User DB)]
    end

    subgraph External["Внешние"]
        SMS[SMS-провайдер]
        Push[Push-провайдер]
    end

    Mobile --> Gateway
    Web --> Gateway
    Gateway --> BookingSvc
    Gateway --> CatalogSvc
    Gateway --> UserSvc
    BookingSvc --> BookingDB
    BookingSvc --> CatalogSvc
    BookingSvc --> UserSvc
    BookingSvc --> NotifSvc
    CatalogSvc --> CatalogDB
    UserSvc --> UserDB
    NotifSvc --> SMS
    NotifSvc --> Push
```

**Скоуп этого ТЗ:** тёмно-синий блок **Booking Service** и его интеграции с соседями.
