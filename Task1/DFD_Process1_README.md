# DFD Process 1: Приём пациента и запись на приём

## Обзор процесса

**Название:** Приём пациента и запись на приём к врачу  
**Тип:** Основной бизнес-процесс (core business process)  
**Участники:** Пациент → Сотрудник ресепшена → Врач-специалист  
**Частота:** Ежедневно, множество раз в день

---

## Участники (External Entities)

| Участник | Роль | Действия |
|----------|------|---------|
| ** Пациент** | External entity | Предоставляет данные при первом визите или звонке; уточняет информацию |
| ** Ресепшен (Reception)** | Internal actor | Принимает информацию; ищет пациента в системе; назначает врача и время |

---

## Основные процессы (Processes)

### P1.1: Ввод данных пациента

**Входные данные (Input):**
- ФИО (фамилия, имя, отчество) — базовые ПИИ
- Дата рождения — базовые ПИИ
- Телефон контактный — базовые ПИИ
- Email — базовые ПИИ
- Расширенные данные (опционально):
  - Адрес проживания (город, улица, дом)
  - Место работы/учёбы
  - Наличие хронических заболеваний

**Обработка:**
1. Ручной ввод в форму (Excel или портал)
2. Валидация базовых полей (проверка формата телефона, email)

**Выходные данные (Output):**
- Валидированные данные пациента → передача в P1.2

---

### P1.2: Проверка дубликатов

**Входные данные (Input):**
- Валидированные данные пациента из P1.1
- Данные из D1 (база пациентов)

**Обработка:**
1. Поиск пациента по ФИО + ДР в базе (fuzzy matching)
2. Если найден → пациент существует (update контакты)
3. Если не найден → новый пациент (create)

**Выходные данные (Output):**
- Patient_ID (существующий или новый)
- Статус: "найден" или "новый"
- → передача в P1.3

---

### P1.3: Назначение врача и расписание

**Входные данные (Input):**
- Patient_ID из P1.2
- Информация о пациенте (причина визита, предпочтения)
- Расписание врачей (из внутренней системы)

**Обработка:**
1. Сотрудник ресепшена смотрит расписание врачей
2. Выбирает подходящего врача и временной слот
3. Проверяет наличие свободного времени

**Выходные данные (Output):**
- Запись на приём (Appointment record):
  - Patient_ID
  - Doctor_ID
  - Date & Time
  - Status: "scheduled"
- → передача в D2 (Журнал записей)

---

## Хранилища данных (Data Stores)

### D1: Пациенты

**Текущее состояние (As-Is):**
```
Excel файл: Пациенты.xlsx
Расположение: \\fileserver\Medikamente\Patients\
Формат: 
  | ФИО | Дата рождения | Телефон | Email | Адрес | Место работы | Хронические болезни |
  |-----|---------------|---------|-------|-------|--------------|-------------------|
  | Иванов И.И. | 1990-05-15 | +7(925)123-45-67 | ivan@email.ru | М. Тверская, д.5 | ООО "Рога и копыта" | Диабет |
  | Петрова А.В. | 1985-03-20 | +7(912)987-65-43 | petr@email.ru | Ленинский пр., д.10 | ИП Рогова | Гипертония |
```

**Проблемы:**
- — **Открытый текст** — все ПИИ видны
- — **Нет шифрования** — любой с доступом к диску читает все
- — **Нет контроля версий** — конфликты при одновременном редактировании
- — **Дублирование** — один пациент может быть добавлен несколько раз
- — **Нет RBAC** — все сотрудники видят всех пациентов

**Целевое состояние (To-Be):**
```
🗄️ PostgreSQL таблица: patients
Структура:
  CREATE TABLE patients (
    id BIGSERIAL PRIMARY KEY,
    fio_encrypted BYTEA NOT NULL,  -- Шифровано AES-256
    dob_encrypted BYTEA NOT NULL,  -- Шифровано AES-256
    phone_encrypted BYTEA NOT NULL, -- Шифровано AES-256; видны только последние 4 цифры
    email_encrypted BYTEA NOT NULL, -- Шифровано AES-256
    address_encrypted BYTEA,         -- Шифровано AES-256
    workplace TEXT,                  -- Не шифровано (низкая чувствительность)
    chronic_diseases_encrypted BYTEA, -- Шифровано AES-256
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    created_by INT REFERENCES users(id),  -- Для аудита
    updated_by INT REFERENCES users(id)   -- Для аудита
  );
  
  -- Индекс для быстрого поиска (по hash от ФИО+ДР)
  CREATE INDEX idx_patient_hash ON patients USING HASH (
    pgcrypto.digest(fio_encrypted || dob_encrypted, 'sha256')
  );
```

**Классификация:** CONFIDENTIAL (Level 3)  
**Шифрование:** AES-256 GCM (PostgreSQL pgcrypto)  
**Доступ (RBAC):** Reception (read/write own), Specialist (read all), Admin (all)  
**Аудит:**  Логирование всех операций

---

### D2: Журнал записей к врачам

**Текущее состояние (As-Is):**
```
Excel файлы по врачам: Journal-Doctor-Ivanov.xlsx, Journal-Doctor-Petrov.xlsx
Расположение: \\fileserver\Medikamente\Journals\
Формат:
  | Дата | Время | ФИО пациента | Телефон | Причина | Статус |
  |------|------|--------------|---------|---------|--------|
  | 2026-03-09 | 10:00 | Иванов И.И. | +7(925)123-45-67 | Прием | scheduled |
  | 2026-03-09 | 11:00 | Петрова А.В. | +7(912)987-65-43 | Консультация | completed |
```

**Проблемы:**
- — **Множественные файлы** — сложно синхронизировать
- — **ФИО и телефон в открытом виде** — утечка при доступе
- — **Нет истории отмен** — нельзя отследить отмены
- — **Нет интеграции** — врач может не видеть новую запись сразу

**Целевое состояние (To-Be):**
```
🗄️ PostgreSQL таблица: appointments
Структура:
  CREATE TABLE appointments (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL REFERENCES patients(id),
    doctor_id INT NOT NULL REFERENCES staff(id),
    scheduled_date DATE NOT NULL,
    scheduled_time TIME NOT NULL,
    reason_encrypted BYTEA,  -- Шифровано (может содержать симптомы)
    status ENUM ('scheduled', 'completed', 'cancelled', 'no_show'),
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    created_by INT REFERENCES users(id),
    cancelled_at TIMESTAMP,
    cancelled_by INT REFERENCES users(id),
    created_reason_for_cancel TEXT
  );
  
  -- История изменений (аудит)
  CREATE TABLE appointments_history (
    id BIGSERIAL PRIMARY KEY,
    appointment_id BIGINT NOT NULL,
    old_status ENUM,
    new_status ENUM,
    changed_at TIMESTAMP DEFAULT NOW(),
    changed_by INT REFERENCES users(id)
  );
```

**Классификация:** CONFIDENTIAL (Level 3)  
**Шифрование:** AES-256 для причины визита  
**Доступ (RBAC):** Reception (write), Doctor (read own), Admin (all)  
**Аудит:**  История всех изменений

---

## Потоки данных (Data Flows)

### Flow 1: Пациент → Ресепшен → P1.1

```
Пациент 
  │
  │  запись к специалисту
  │
  ▼
Ресепшен → Form Input
  │
  ├─ Вводит ФИО: Иванов И.И.
  ├─ Вводит ДР: 1990-05-15
  ├─ Вводит телефон: +7(925)123-45-67
  ├─ Вводит email: ivan@email.ru
  └─ Опционально: адрес, место работы, хронические болезни
  
  ▼
Валидация:
     ├─ ФИО: не пусто? ✓
     ├─ ДР: формат YYYY-MM-DD? ✓
     ├─ Телефон: формат +7(XXX)XXX-XX-XX? ✓
     ├─ Email: формат xxx@xxx.xx? ✓
     └─ Возраст: 18-120 лет? ✓
  
  ▼
Output: валидированные данные
```

**Классификация:** CONFIDENTIAL (базовые ПИИ)  
**Защита in-transit:** — Текущее (открытый текст в Excel) → To-Be (TLS 1.3)  
**Защита at-rest:** — Текущее (открытый текст) → To-Be (AES-256)  

---

### Flow 2: P1.1 → P1.2 (Поиск дубликата)

```
P1.2: Проверка дубликатов
  │
  ├─ Поиск в D1 (Patients):
  │  ├─ SELECT * WHERE fio = "Иванов И.И." AND dob = "1990-05-15"
  │  │ Result: ✓ Найден (patient_id = 42)
  │  └─ Обновляем контакты (телефон, email)
  │
  └─ Или (если не найден):
     ├─ INSERT новую запись
     └─ patient_id = 12345 (новый)

OUTPUT:
  ├─ patient_id: 42 (или 12345)
  ├─ status: "found" (или "new")
  └─ → P1.3
```

**Уязвимость:** — Если в P1.2 ошибка, один пациент может быть дублирован  
**Решение:**  UNIQUE constraint на (fio_encrypted, dob_encrypted, phone_encrypted)

---

### Flow 3: P1.3 → D2 (Запись на приём)

```
P1.3: Назначение врача
  │
  ├─ Ресепшен смотрит расписание врачей:
  │  ├─ Dr. Петров (Кардиолог)
  │  │  ├─ 2026-03-09 09:00 — занято
  │  │  ├─ 2026-03-09 10:00 — свободно ✓
  │  │  └─ 2026-03-09 11:00 — свободно ✓
  │  │
  │  └─ Dr. Иванов (Терапевт)
  │     ├─ 2026-03-09 14:00 — свободно ✓
  │     └─ 2026-03-09 15:00 — занято
  │
  ├─ Сотрудник выбирает:
  │  ├─ Doctor: Петров (doctor_id = 3)
  │  ├─ Date: 2026-03-09
  │  └─ Time: 10:00
  │
  └─ Создание записи (INSERT в D2):
     ├─ patient_id: 42
     ├─ doctor_id: 3
     ├─ scheduled_date: 2026-03-09
     ├─ scheduled_time: 10:00
     ├─ reason: "Консультация кардиолога"
     └─ status: "scheduled"

OUTPUT:
  ├─ appointment_id: 5678
  ├─ confirmation: "Запись создана на 2026-03-09 в 10:00 к Dr. Петрову"
  └─ → D2 (Журнал записей)
```

---

## Текущие уязвимости (As-Is)

| # | Уязвимость | Классификация | Риск |
|---|-----------|----------------|------|
| 1 | **Отсутствие шифрования at-rest** | P1.1 | КРИТИЧНЫЙ |
| 2 | **Открытый текст на файловом сервере** | P1.1 | КРИТИЧНЫЙ |
| 3 | **Отсутствие RBAC** | P1.2 | КРИТИЧНЫЙ |
| 4 | **Нет логирования** | P1.3 | КРИТИЧНЫЙ |
| 5 | **Ручной ввод → опечатки** | P1.1 | ВЫСОКИЙ |
| 6 | **Дублирование пациентов** | P1.2 | ВЫСОКИЙ |
| 7 | **Нет истории отмен** | P1.3 | ВЫСОКИЙ |
| 8 | **Конфликты версий Excel** | D2 | ВЫСОКИЙ |

---

##  Рекомендуемые контроли (To-Be)

### Контроль 1: Шифрование in-transit

**Что:** TLS 1.3 для всех HTTP запросов  
**Где:** Portal → Backend API  
**Как:**
```
GET https://api.medikamente.local/patients/search?fio=Иванов&dob=1990-05-15
  ├─ Protocol: TLS 1.3 (ECDHE для PFS)
  ├─ Certificate: Signed by internal CA (для on-prem)
  └─ Headers:
     ├─ Authorization: Bearer <JWT>
     └─ X-Request-ID: <uuid> (для трассировки)

Response: 200 OK
  ├─ Body (encrypted in TLS tunnel):
  │  ├─ patient_id: 42
  │  ├─ status: "found"
  │  └─ phone_masked: "+7(XXX)XXX-56-67" (видны только последние 4)
  └─ Headers:
     └─ X-Response-ID: <uuid>
```

---

### Контроль 2: RBAC на уровне API

**Что:** Role-based access control  
**Где:** API Gateway → Авторизация  
**Как:**
```
Пользователь: reception@medikamente.local (Role = Reception)
  ├─ Может: READ пациентов (поиск)
  ├─ Может: CREATE appointments
  ├─ Не может: DELETE пациентов
  └─ Не может: READ медицинских карт других врачей

Policy (OPA):
  allow {
    input.user.role == "Reception"
    input.action == "read"
    input.resource == "appointments"
  }
```

---

### Контроль 3: Логирование всех операций

**Что:** Immutable audit log  
**Где:** Elasticsearch + Kibana  
**Как:**
```
Event 1:
  timestamp: 2026-03-09T09:30:00Z
  user_id: 5 (reception@medikamente.local)
  action: "search_patient"
  query: { fio: "Иванов", dob: "1990-05-15" }
  result: "found" (patient_id: 42)
  ip_address: 192.168.1.45
  user_agent: "Mozilla/5.0..."
  
Event 2:
  timestamp: 2026-03-09T09:35:00Z
  user_id: 5
  action: "create_appointment"
  patient_id: 42
  doctor_id: 3
  scheduled_time: "2026-03-09T10:00:00Z"
  result: "success" (appointment_id: 5678)
  ip_address: 192.168.1.45

Query (для forensics):
  GET /elasticsearch/audit_log/_search
    {
      "query": {
        "bool": {
          "must": [
            { "match": { "user_id": 5 } },
            { "range": { "timestamp": { "gte": "2026-03-09T09:00:00Z" } } }
          ]
        }
      }
    }
```

---