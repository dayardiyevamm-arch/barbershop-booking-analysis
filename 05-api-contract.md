# 5. API-контракт

Сервис реализует **REST API**, описанный в формате **OpenAPI 3.0** (см. [`openapi.yaml`](openapi.yaml)).

Этот документ — человекочитаемое описание. Для интерактивной документации скопируйте `openapi.yaml` в [Swagger Editor](https://editor.swagger.io/).

---

## 5.1. Общие принципы

**Базовый URL:** `https://api.barberbook.kz/v1`

**Аутентификация:** JWT-токен в заголовке.
```
Authorization: Bearer <token>
```

**Формат данных:** JSON (`Content-Type: application/json`).

**Даты и время:** ISO 8601 в UTC.
```
"start_at": "2026-04-24T09:00:00Z"
```

**Версионирование:** через URL (`/v1/`).

**Пагинация:** query-параметры `limit` (по умолчанию 20, макс. 100) и `offset`.

---

## 5.2. Эндпоинты

### 🔹 GET /slots — получить свободные слоты

**Описание:** возвращает список свободных слотов у мастеров на выбранную дату с учётом длительности услуги.

**Параметры запроса:**

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `branch_id` | UUID | ✓ | ID филиала |
| `service_id` | UUID | ✓ | ID услуги (определяет длительность слота) |
| `master_id` | UUID | — | Фильтр по мастеру; если не указан — все мастера филиала |
| `date` | date (YYYY-MM-DD) | ✓ | Дата в часовом поясе филиала |

**Пример запроса:**
```http
GET /v1/slots?branch_id=b1a2...&service_id=s1...&date=2026-04-24
Authorization: Bearer eyJhbGc...
```

**Пример ответа (200 OK):**
```json
{
  "date": "2026-04-24",
  "timezone": "Asia/Almaty",
  "slots": [
    {
      "master_id": "m1...",
      "master_name": "Алихан А.",
      "start_at": "2026-04-24T04:00:00Z",
      "end_at": "2026-04-24T04:30:00Z",
      "price": 5000
    },
    {
      "master_id": "m1...",
      "master_name": "Алихан А.",
      "start_at": "2026-04-24T04:30:00Z",
      "end_at": "2026-04-24T05:00:00Z",
      "price": 5000
    }
  ]
}
```

---

### 🔹 POST /bookings — создать запись

**Описание:** бронирует выбранный слот.

**Тело запроса:**
```json
{
  "branch_id": "b1a2...",
  "master_id": "m1...",
  "service_id": "s1...",
  "start_at": "2026-04-24T09:00:00Z"
}
```

**Пример успешного ответа (201 Created):**
```json
{
  "id": "bk_9f3e...",
  "status": "CONFIRMED",
  "client_id": "c1...",
  "master_id": "m1...",
  "master_name": "Алихан А.",
  "service_id": "s1...",
  "service_name": "Мужская стрижка",
  "branch_id": "b1a2...",
  "start_at": "2026-04-24T09:00:00Z",
  "end_at": "2026-04-24T09:30:00Z",
  "price": 5000,
  "created_at": "2026-04-22T14:32:10Z"
}
```

**Возможные ошибки:**
| Код | Описание |
|---|---|
| 400 | Невалидный запрос (например, `start_at` не ISO 8601) |
| 401 | Токен невалиден или истёк |
| 409 | `SLOT_ALREADY_TAKEN` — слот занят другим клиентом |
| 409 | `BOOKING_OVERLAP` — у клиента уже есть запись в это время |
| 422 | `BOOKING_IN_PAST` — время бронирования в прошлом |

Подробнее — см. [06-errors.md](06-errors.md).

---

### 🔹 GET /bookings — список моих записей

**Параметры:**
| Параметр | Тип | Описание |
|---|---|---|
| `status` | enum | `upcoming`, `past`, `cancelled` — фильтр по категории |
| `limit` | int | Размер страницы (default 20, max 100) |
| `offset` | int | Смещение |

**Пример:**
```http
GET /v1/bookings?status=upcoming&limit=10
```

**Ответ (200 OK):**
```json
{
  "total": 2,
  "limit": 10,
  "offset": 0,
  "items": [
    {
      "id": "bk_9f3e...",
      "status": "CONFIRMED",
      "master_name": "Алихан А.",
      "service_name": "Мужская стрижка",
      "start_at": "2026-04-24T09:00:00Z",
      "end_at": "2026-04-24T09:30:00Z",
      "price": 5000
    }
  ]
}
```

---

### 🔹 GET /bookings/{id} — получить запись

**Пример:**
```http
GET /v1/bookings/bk_9f3e...
```

**Ответ:** та же структура, что в POST, + `history` (краткая история изменений).

**Ошибки:** 404 (не найдена), 403 (не твоя запись).

---

### 🔹 PATCH /bookings/{id}/cancel — отменить запись

**Тело запроса:** пустое (или опциональная причина).
```json
{
  "reason": "Планы изменились"
}
```

**Ответ (200 OK):**
```json
{
  "id": "bk_9f3e...",
  "status": "CANCELLED",
  "cancelled_at": "2026-04-22T15:00:00Z"
}
```

**Ошибки:**
| Код | Название | Описание |
|---|---|---|
| 403 | `FORBIDDEN` | Запись принадлежит другому клиенту |
| 404 | `BOOKING_NOT_FOUND` | Запись не существует |
| 422 | `INVALID_STATE` | Статус не `CONFIRMED` |
| 422 | `CANCELLATION_WINDOW_EXPIRED` | До визита меньше 2 часов |

---

### 🔹 PATCH /bookings/{id}/reschedule — перенести запись

**Тело запроса:**
```json
{
  "new_start_at": "2026-04-25T11:00:00Z",
  "master_id": "m2...",
  "service_id": "s1..."
}
```

> `master_id` и `service_id` можно не передавать, если они не меняются.

**Ответ (200 OK):** обновлённая запись.

**Ошибки:**
| Код | Название | Описание |
|---|---|---|
| 403 | `FORBIDDEN` | Не твоя запись |
| 404 | `BOOKING_NOT_FOUND` | Записи нет |
| 409 | `SLOT_ALREADY_TAKEN` | Новый слот занят |
| 422 | `RESCHEDULE_WINDOW_EXPIRED` | До визита меньше 2 часов |
| 422 | `INVALID_STATE` | Запись уже отменена или завершена |

---

## 5.3. Формат ошибок

Все ошибки возвращаются в унифицированном формате по мотивам **RFC 7807 (Problem Details)**:

```json
{
  "error": {
    "code": "SLOT_ALREADY_TAKEN",
    "message": "Выбранный слот только что был забронирован другим клиентом",
    "details": {
      "slot_start_at": "2026-04-24T09:00:00Z",
      "master_id": "m1..."
    },
    "request_id": "req_abc123"
  }
}
```

Поле `request_id` помогает быстро найти запрос в логах для дебага.

---

## 5.4. Заголовки и коды ответов — сводка

| HTTP-код | Когда возвращается |
|---|---|
| 200 OK | Успешный GET/PATCH |
| 201 Created | Успешное создание ресурса (POST) |
| 400 Bad Request | Невалидные параметры (типы, обязательные поля) |
| 401 Unauthorized | Токен отсутствует / невалиден / истёк |
| 403 Forbidden | Нет прав на ресурс |
| 404 Not Found | Ресурс не существует |
| 409 Conflict | Конфликт состояния (слот занят, пересечение) |
| 422 Unprocessable Entity | Бизнес-правило нарушено (окно отмены, статус) |
| 500 Internal Server Error | Непредвиденная ошибка сервера |

---

## 5.5. Примеры использования в Postman

1. Скачайте `openapi.yaml`.
2. В Postman: **Import → Upload Files → openapi.yaml** — создастся коллекция с готовыми запросами.
3. В переменных коллекции установите `baseUrl` и `token`.
4. Тестируйте эндпоинты в порядке: GET /slots → POST /bookings → GET /bookings → PATCH /bookings/{id}/reschedule → PATCH /bookings/{id}/cancel.
