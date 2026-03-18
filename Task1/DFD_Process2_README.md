# DFD Process 2: Медицинский осмотр и диагностика

## Обзор процесса

**Название:** Медицинский осмотр пациента, проведение диагностики, выписка рецептов  
**Тип:** Основной медицинский процесс (core medical process)  
**Участники:** Пациент → Врач-специалист → Лаборатория (внешняя)  
**Классификация данных:** RESTRICTED (Level 4) — медицинская тайна  
**Частота:** Ежедневно, множество осмотров в день  

---

## Участники (External Entities)

| Участник | Роль | Действия | Доступ к данным |
|----------|------|---------|-----------------|
| ** Пациент** | External entity | Приходит на приём; описывает симптомы; получает диагноз и рецепт | Свои данные (RBAC) |
| ** Врач-специалист** | Internal actor | Проводит осмотр; ставит диагноз; выписывает рецепт; подписывает диагноз | Пациентов своей специальности (RBAC) |
| ** Лаборатория** | External system | Получает направление; проводит анализы; возвращает результаты | Только результаты своих анализов (API) |

---

## Основные процессы (Processes)

### P2.1: Проверка медицинской истории

**Входные данные (Input):**
- Patient_ID (из процесса 1)

**Обработка:**
1. Врач запускает портал
2. Вводит/сканирует Patient_ID
3. Система загружает всю историю пациента (предыдущие визиты, диагнозы, анализы)
4. Врач просматривает историю

**Выходные данные (Output):**
- Полная медицинская история → P2.2

**Уязвимости:**
- — История может быть видна другим врачам
- — Нет логирования доступа ("кто просмотрел историю?")
- — Нет шифрования при передаче по сети

---

### P2.2: Проведение осмотра и сбор симптомов

**Входные данные (Input):**
- Пациент
- История из P2.1

**Обработка:**
1. Врач проводит физический осмотр
2. Собирает жалобы пациента (текстовое описание)
3. Измеряет параметры (температура, давление, пульс)
4. Пальпирует, перкуссирует, аускультирует

**Выходные данные (Output):**
- Результаты осмотра:
  - Жалобы: "Головная боль, тошнота"
  - Объективные данные: "T=37.8°C, BP=140/90, пульс=95"
  - Предварительный диагноз: "Грипп?"
  - Необходимые анализы: "ОАК, ОАМ, УЗИ"
  - → передача в P2.3

---

### P2.3: Ввод диагноза и МКБ-коды

**Входные данные (Input):**
- Результаты осмотра из P2.2
- История из P2.1

**Обработка:**
1. Врач открывает форму диагноза
2. Вводит описание диагноза (текстовое поле)
3. Выбирает МКБ-код из справочника (International Classification of Diseases)
   - МКБ-10 код: "J11.1" (Грипп с поражением нижних дыхательных путей)
4. Указывает степень тяжести: "лёгкая" / "средняя" / "тяжёлая"
5. **Подписывает диагноз** (цифровая подпись врача)

**Выходные данные (Output):**
- Медицинская запись:
  - Диагноз (текст): "Острый грипп с осложнением (бронхит)"
  - МКБ-код: "J11.1"
  - Дата: 2026-03-09
  - Врач (подпись): Dr. Петров (цифровая подпись)
  - → передача в P2.4 и D1 (медицинские карты)

---

### P2.4: Создание рецепта и назначений

**Входные данные (Input):**
- Диагноз из P2.3

**Обработка:**
1. Врач открывает форму рецепта
2. Добавляет лекарства:
   - Название: "Амоксициллин"
   - Дозировка: "500 мг"
   - Форма: "капсулы"
   - Количество: "20 капсул"
   - Частота приёма: "3 раза в день"
   - Длительность: "7 дней"
3. Добавляет рекомендации:
   - "Постельный режим"
   - "Обильное тёплое питьё"
   - "Повторный приём через 7 дней"
4. Генерирует рецепт (печать или электронный QR-код)

**Выходные данные (Output):**
- Рецепт (структурированные данные):
  - Пациент: Patient_ID 42
  - Врач: Doctor_ID 3
  - Лекарства: [Амоксициллин 500мг x3 на 7 дней]
  - Дата выписки: 2026-03-09
  - Подпись врача: Dr. Петров
  - Рецепт_ID: 7890
  - → передача в D3 (рецепты) + печать пациенту

---

### P2.5 (опционально): Направление на анализы

**Входные данные (Input):**
- Диагноз из P2.3
- Решение врача о необходимости анализов

**Обработка:**
1. Врач создаёт направление на анализы:
   - Пациент: Patient_ID 42
   - Анализы: "ОАК, ОАМ, Биохимия"
   - Приоритет: "стандартный"
   - Сроки: "в течение 3 дней"
2. Система генерирует направление (печать или QR-код)

**Выходные данные (Output):**
- Направление на анализы:
  - Направление_ID: 9999
  - Patient_ID: 42
  - Анализы: ["ОАК", "ОАМ", "Биохимия"]
  - → передача в D4 (направления) и потом в лабораторию

---

## Хранилища данных (Data Stores)

### D1: Медицинские карты (Диагнозы)

**Текущее состояние (As-Is):**
```
Excel / PDF / JPG: Карта-Пациент42.xlsx
Расположение: \\fileserver\Medikamente\MedicalCards\Patient42\
Формат:
  | Дата | Врач | Диагноз | МКБ-код | Рецепт | Примечания |
  |------|------|---------|---------|--------|-----------|
  | 2026-03-09 | Dr. Петров | Грипп с бронхитом | J11.1 | Амокс... | Постельный режим |
```

**Проблемы:**
- — **Открытый текст** — любой видит диагноз
- — **Нет шифрования** — утечка при доступе к диску
- — **Нет цифровой подписи** — врач может отрицать диагноз
- — **Можно редактировать** — врач может изменить диагноз задним числом
- — **Нет версионирования** — нельзя восстановить старую версию
- — **Нет RBAC** — все врачи видят всех пациентов

**Целевое состояние (To-Be):**
```
PostgreSQL таблица: medical_records
Структура:
  CREATE TABLE medical_records (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL REFERENCES patients(id),
    doctor_id INT NOT NULL REFERENCES staff(id),
    visit_date DATE NOT NULL,
    
    -- Шифрованные поля
    diagnosis_encrypted BYTEA NOT NULL,  -- Шифровано AES-256
    symptoms_encrypted BYTEA,
    examination_findings_encrypted BYTEA,
    
    -- Структурированные данные
    icd10_code VARCHAR(10),  -- МКБ-10 код (не шифруется, нужен для поиска)
    severity ENUM ('light', 'moderate', 'severe'),
    
    -- Цифровая подпись
    doctor_signature BYTEA NOT NULL,  -- PKCS#1 подпись
    signature_timestamp TIMESTAMP NOT NULL,
    certificate_fingerprint VARCHAR(64),  -- Отпечаток сертификата врача
    
    -- Контроль версий
    is_finalized BOOLEAN DEFAULT FALSE,  -- После подписания = TRUE (нельзя редактировать)
    created_at TIMESTAMP DEFAULT NOW(),
    created_by INT REFERENCES users(id),
    
    -- Для immutable history
    version INT DEFAULT 1,
    UNIQUE(patient_id, visit_date, doctor_id)
  );
  
  -- История версий (полный аудит)
  CREATE TABLE medical_records_history (
    id BIGSERIAL PRIMARY KEY,
    medical_record_id BIGINT NOT NULL REFERENCES medical_records(id),
    version INT,
    diagnosis_encrypted BYTEA,
    changed_at TIMESTAMP DEFAULT NOW(),
    changed_by INT,
    change_reason TEXT,
    is_cancelled BOOLEAN DEFAULT FALSE
  );
  
  -- Сигнатуры врачей (для верификации)
  CREATE TABLE doctor_signatures (
    id SERIAL PRIMARY KEY,
    doctor_id INT NOT NULL REFERENCES staff(id),
    certificate_fingerprint VARCHAR(64) UNIQUE,
    public_key TEXT,  -- PEM format
    valid_from DATE NOT NULL,
    valid_until DATE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
  );
```

**Классификация:** RESTRICTED (Level 4) — медицинская тайна  
**Шифрование:** AES-256 GCM для диагнозов и симптомов  
**Цифровая подпись:** УКЭП (Усиленная квалифицированная ЭП) или PKCS#1 RSA  
**Доступ (RBAC):** Doctor (read/write own), Patient (read own), Admin (all), Auditor (read logs only)  
**Аудит:** Полная история версий + сигнатуры

---

### D2: Результаты анализов (внешняя интеграция)

**Текущее состояние (As-Is):**
``` 
Сейчас: результаты приходят на бумаге или в Excel
```

**Целевое состояние (To-Be):**
```
PostgreSQL таблица: lab_results
Структура:
  CREATE TABLE lab_results (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL REFERENCES patients(id),
    order_id BIGINT REFERENCES lab_orders(id),
    lab_id INT NOT NULL REFERENCES external_labs(id),
    
    test_code VARCHAR(20),  -- "OAK" (общий анализ крови)
    test_name VARCHAR(100),
    result_value NUMERIC,
    unit VARCHAR(20),  -- "г/л", "млн/мкл"
    normal_range VARCHAR(50),
    is_normal BOOLEAN,
    
    result_date DATE NOT NULL,
    result_time TIME,
    
    -- Проверка целостности
    result_hash BYTEA,  -- HMAC от result_value
    received_signature BYTEA,  -- Подпись от лаборатории
    
    received_at TIMESTAMP DEFAULT NOW(),
    encrypted_at TIMESTAMP  -- Когда был зашифрован
  );
  
  -- Для шифрованного хранилища
  CREATE TABLE lab_results_encrypted (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT,
    data_encrypted BYTEA,  -- Весь результат в одном зашифрованном поле
    created_at TIMESTAMP DEFAULT NOW()
  );
```

**Классификация:** RESTRICTED (Level 4)  
**Шифрование:** AES-256 для результатов  
**Целостность:** HMAC-SHA256 от лаборатории  
**Доступ (RBAC):** Doctor (read), Patient (read own), Lab (write own tests)

---

### D3: Рецепты

**Целевое состояние (To-Be):**
```
PostgreSQL таблица: prescriptions
Структура:
  CREATE TABLE prescriptions (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL REFERENCES patients(id),
    doctor_id INT NOT NULL REFERENCES staff(id),
    
    prescription_date DATE NOT NULL,
    valid_until DATE,  -- Рецепт действителен XX дней
    
    -- Шифрованные данные
    medications_encrypted BYTEA NOT NULL,
    recommendations_encrypted BYTEA,
    
    -- Структурированные данные
    total_medications INT,  -- количество лекарств в рецепте
    
    -- Цифровая подпись
    doctor_signature BYTEA NOT NULL,
    signature_timestamp TIMESTAMP NOT NULL,
    
    -- Для печати / QR-кода
    qr_code TEXT,  -- encrypted patient_id + recipe_id
    print_count INT DEFAULT 0,
    last_printed_at TIMESTAMP,
    
    -- Статус
    status ENUM ('active', 'used', 'expired', 'cancelled'),
    is_finalized BOOLEAN DEFAULT TRUE,  -- После выписки
    
    created_at TIMESTAMP DEFAULT NOW(),
    created_by INT
  );
  
  -- Детали рецепта (каждое лекарство)
  CREATE TABLE prescription_items (
    id BIGSERIAL PRIMARY KEY,
    prescription_id BIGINT NOT NULL REFERENCES prescriptions(id),
    medication_name VARCHAR(100),  -- AES-256 encrypted
    dosage VARCHAR(50),             -- AES-256 encrypted
    frequency VARCHAR(50),          -- "3 раза в день"
    duration INT,                   -- в днях
    quantity INT,                   -- количество таблеток/капсул
    pharmacy_code VARCHAR(20),      -- для связи с аптекой
    created_at TIMESTAMP DEFAULT NOW()
  );
```

**Классификация:** RESTRICTED (Level 4)  
**Шифрование:** AES-256 для названий лекарств и рекомендаций  
**Цифровая подпись:** УКЭП для юридической значимости  
**Доступ (RBAC):** Doctor (write), Patient (read own), Pharmacist (verify + dispense)

---

### D4: Направления на анализы

**Целевое состояние (To-Be):**
```
PostgreSQL таблица: lab_orders
Структура:
  CREATE TABLE lab_orders (
    id BIGSERIAL PRIMARY KEY,
    patient_id BIGINT NOT NULL REFERENCES patients(id),
    doctor_id INT NOT NULL REFERENCES staff(id),
    
    order_date DATE NOT NULL,
    requested_tests_encrypted BYTEA NOT NULL,
    
    -- Структурированные данные
    priority ENUM ('routine', 'urgent'),
    target_completion_date DATE,
    
    -- Для интеграции с лабораторией
    lab_id INT NOT NULL REFERENCES external_labs(id),
    external_order_id VARCHAR(50),  -- ID в системе лаборатории
    
    -- Цифровая подпись
    doctor_signature BYTEA NOT NULL,
    
    -- Статус
    status ENUM ('created', 'sent_to_lab', 'in_progress', 'completed', 'cancelled'),
    sent_to_lab_at TIMESTAMP,
    completed_at TIMESTAMP,
    
    created_at TIMESTAMP DEFAULT NOW()
  );
```

**Классификация:** CONFIDENTIAL (Level 3)  
**Шифрование:** AES-256 для названий анализов  
**Доступ (RBAC):** Doctor (create), Lab (read + update status), Patient (read own)

---

## Потоки данных (Data Flows)

### Flow 1: Врач загружает историю пациента (P2.1)

```
Врач Dr. Петров
  │
  ├─ Вводит Patient_ID: 42
  │
  ▼
API Call (TLS 1.3 + mTLS):
  │
  GET https://api.medikamente.local/medical-records/patient/42
    ├─ Header: Authorization: Bearer <JWT_doctor_token>
    ├─ Header: X-Doctor-ID: 3
    ├─ RBAC check: Doctor_ID 3 can read records for Patient_ID 42? ✓
    └─ Query: SELECT * FROM medical_records WHERE patient_id = 42 ORDER BY visit_date DESC
  
  ▼
Response (зашифровано в TLS tunnel):
  {
    "records": [
      {
        "id": 5000,
        "visit_date": "2026-03-02",
        "diagnosis": "ORVI (грипп подтвержден тестом)",
        "doctor": "Dr. Ivanov",
        "icd10": "J11.0"
      },
      {
        "id": 4999,
        "visit_date": "2026-02-15",
        "diagnosis": "Гипертония",
        "doctor": "Dr. Petrov",
        "icd10": "I10"
      }
    ]
  }
  
  ▼
Логирование в audit_log:
  {
    "timestamp": "2026-03-09T10:00:00Z",
    "user_id": 3,
    "action": "read_medical_records",
    "patient_id": 42,
    "records_count": 2,
    "ip_address": "192.168.1.50",
    "result": "success"
  }
```

**Уязвимости:**
- As-Is: Врач может читать истории других пациентов без логирования
- To-Be: RBAC + логирование + шифрование in-transit

---

### Flow 2: Врач вводит диагноз и подписывает (P2.3)

```
Врач Dr. Петров
  │
  ├─ Открывает форму диагноза (HTTPS + TLS 1.3)
  │
  ├─ Вводит:
  │  ├─ Диагноз: "Острый грипп с осложнением (бронхит)"
  │  ├─ МКБ-код: "J11.1"
  │  ├─ Степень тяжести: "средняя"
  │  └─ Клинические находки: "Тахипноэ, хрипы при аускультации"
  │
  ├─ Нажимает "Подписать диагноз" (требуется УКЭП)
  │
  ▼
Цифровая подпись (УКЭП):
  │
  ├─ Система запрашивает сертификат врача
  ├─ Врач вводит пин-код от сертификата
  ├─ Подпись вычисляется:
  │  └─ signature = SIGN_WITH_PRIVATE_KEY(
  │      diagnosis_text + patient_id + timestamp
  │    )
  └─ Отпечаток сертификата сохраняется
  
  ▼
Сохранение в БД:
  INSERT INTO medical_records (
    patient_id, doctor_id, visit_date,
    diagnosis_encrypted,      -- AES-256
    icd10_code, severity,
    doctor_signature,         -- PKCS#1 RSA
    signature_timestamp,
    certificate_fingerprint,
    is_finalized,
    created_at, created_by
  ) VALUES (
    42, 3, '2026-03-09',
    pgp_sym_encrypt('Острый грипп с осложнением (бронхит)', 'key'),
    'J11.1', 'moderate',
    <binary_signature>,
    NOW(),
    'a1b2c3d4...',
    TRUE,  -- Финализирован (нельзя редактировать)
    NOW(), 5
  );
  
  ▼
Логирование:
  INSERT INTO audit_log VALUES (
    user_id: 3,
    action: 'create_medical_record',
    resource_id: <new_record_id>,
    before: NULL,
    after: json_build_object(...),
    timestamp: NOW()
  );
```

---

### Flow 3: Врач выписывает рецепт (P2.4)

```
Врач Dr. Петров
  │
  ├─ Открывает форму рецепта
  ├─ Добавляет лекарства:
  │  ├─ Амоксициллин 500мг x3 на 7 дней
  │  ├─ Парацетамол 500мг x3 на 3 дня
  │  └─ Витамин C 1000мг x1 на 10 дней
  │
  ├─ Добавляет рекомендации:
  │  ├─ Постельный режим 2-3 дня
  │  ├─ Обильное питьё
  │  └─ Повторный приём через 7 дней
  │
  └─ Подписывает и выписывает рецепт
  
  ▼
Создание рецепта (с цифровой подписью):
  
  INSERT INTO prescriptions (
    patient_id, doctor_id,
    prescription_date, valid_until,
    medications_encrypted,  -- ["Амоксициллин...", "Парацетамол...", "Витамин C..."]
    recommendations_encrypted,
    doctor_signature,
    signature_timestamp,
    status, is_finalized
  ) VALUES (
    42, 3,
    '2026-03-09', '2026-04-09',  -- действителен 1 месяц
    pgp_sym_encrypt(json_array(...), 'key'),
    pgp_sym_encrypt('Постельный режим...', 'key'),
    <signature>,
    NOW(),
    'active', TRUE
  ) RETURNING id INTO recipe_id;
  
  ▼
Генерация QR-кода:
  
  qr_code_data = {
    recipe_id: 7890,
    patient_id: 42,
    doctor_id: 3,
    timestamp: NOW(),
    signature: hash(recipe_id + patient_id + timestamp)
  }
  
  qr_code = ENCODE_TO_QR(ENCRYPT(qr_code_data, 'key'))
  
  -- Сохраняем QR в БД
  UPDATE prescriptions SET qr_code = '<qr_binary>' WHERE id = recipe_id;
  
  ▼
Выдача рецепта пациенту:
  
  ├─ Печать: A4 с QR-кодом + расшифрованный текст
  │  "Амоксициллин 500мг х3 на 7 дней"
  │  "Парацетамол 500мг х3 на 3 дня"
  │  "Витамин C 1000мг х1 на 10 дней"
  │
  ├─ SMS/Email пациенту:
  │  "Ваш рецепт готов. QR-код: [image]"
  │
  └─ Сохранение в портале пациента
     (шифрованный доступ)
```

---

### Flow 4: Направление на анализы (P2.5)

```
Врач Dr. Петров
  │
  ├─ Решает: пациенту нужны анализы
  ├─ Заполняет форму направления:
  │  ├─ Пациент: Patient_ID 42
  │  ├─ Анализы: ["ОАК", "ОАМ", "Биохимия"]
  │  ├─ Показания: "Грипп подтвержен; исключить осложнения"
  │  ├─ Приоритет: "стандартный"
  │  └─ Лаборатория: "ООО Инвитро" (LAB_ID 1)
  │
  └─ Подписывает направление (УКЭП)
  
  ▼
Создание в БД:
  INSERT INTO lab_orders (
    patient_id, doctor_id, order_date,
    requested_tests_encrypted,
    priority, target_completion_date,
    lab_id, doctor_signature
  ) VALUES (
    42, 3, '2026-03-09',
    pgp_sym_encrypt(['OAK', 'OAM', 'Biochemistry'], 'key'),
    'routine', '2026-03-12',
    1, <signature>
  ) RETURNING id INTO order_id;
  
  ▼
Отправка в лабораторию (API Integration):
  
  -- Асинхронный процесс
  POST https://lab.invitro.local/orders/create
    ├─ mTLS: сертификат Медикаменте
    ├─ Body (зашифровано):
    │  {
    │    "order_id": 999,
    │    "patient_id": "<encrypted>",  -- Token, не реальный ID
    │    "tests": ["OAK", "OAM"],
    │    "priority": "routine",
    │    "target_date": "2026-03-12"
    │  }
    └─ Signature: HMAC-SHA256(body, shared_secret)
  
  ▼
Логирование:
  INSERT INTO audit_log VALUES (
    action: 'create_lab_order',
    resource_id: order_id,
    patient_id: 42,
    lab_id: 1,
    timestamp: NOW()
  );
```

---

## Текущие уязвимости (As-Is)

| # | Уязвимость | Риск | Классификация |
|---|-----------|------|----------------|
| 1 | **Диагнозы в открытом виде** | КРИТИЧНЫЙ | P1.1 |
| 2 | **Отсутствие цифровой подписи** | КРИТИЧНЫЙ | P1.2 |
| 3 | **Возможна подделка диагноза** | КРИТИЧНЫЙ | P1.3 |
| 4 | **Нет шифрования при передаче** | КРИТИЧНЫЙ | P1.4 |
| 5 | **Нет контроля целостности** | ВЫСОКИЙ | P2.3 |
| 6 | **Рецепты в открытом виде** | ВЫСОКИЙ | P2.1 |
| 7 | **Можно редактировать рецепт** | ВЫСОКИЙ | P2.2 |
| 8 | **Нет версионирования** | ВЫСОКИЙ | P2.4 |
| 9 | **Отсутствие RBAC** | ВЫСОКИЙ | P2.5 |
| 10 | **Все врачи видят всех пациентов** | ВЫСОКИЙ | P2.6 |

---

## Рекомендуемые контроли (To-Be)

### Контроль 1: Цифровая подпись (УКЭП)

**Что:** Усиленная квалифицированная электронная подпись  
**Где:** На все диагнозы, рецепты, направления  
**Как:**
```
Врач получает УКЭП сертификат:
  ├─ От удостоверяющего центра (УЦ)
  ├─ Сертификат содержит:
  │  ├─ Public Key (RSA 2048)
  │  ├─ ФИО врача
  │  ├─ СНИЛС
  │  ├─ Лицензия на врачебную деятельность
  │  └─ Валидность: 1 год (обновляется)
  │
  └─ Private Key хранится:
     ├─ На смарт-карте (криптографический токен)
     ├─ Защищена ПИН-кодом
     └─ Никогда не экспортируется

Подпись диагноза:
  1. Врач открывает форму диагноза
  2. Нажимает "Подписать"
  3. Система запрашивает ПИН
  4. Вычисляется подпись:
     signature = RSA_SIGN(private_key, SHA256(diagnosis_text + timestamp))
  5. Подпись прилагается к диагнозу
  6. Диагноз становится неизменяемым (IMMUTABLE)

Верификация подписи:
  system.verify_signature(
    signature = <binary>,
    message = diagnosis_text + timestamp,
    public_key = doctor_certificate.public_key
  ) → TRUE/FALSE
```

---

### Контроль 2: Versioning и Immutable History

**Что:** История всех версий диагноза + невозможность редактирования после подписания  
**Где:** medical_records_history таблица  
**Как:**
```
-- Попытка редактировать подписанный диагноз:
UPDATE medical_records SET diagnosis_encrypted = '...' 
WHERE id = 5000 AND is_finalized = TRUE;
→ ERROR: "Cannot update finalized medical record"

-- Если нужно исправить (только врач + причина):
INSERT INTO medical_records_history (
  medical_record_id, version, diagnosis_encrypted,
  change_reason, changed_by
) VALUES (
  5000, 2, pgp_sym_encrypt('Исправленный диагноз', 'key'),
  'Исправление опечатки в диагнозе',
  doctor_id
);

-- Аудит:
SELECT * FROM medical_records_history WHERE medical_record_id = 5000;
  version 1: "Грипп" (2026-03-09 10:00, Dr. Petrov)
  version 2: "ОРВИ, грипп подтвержен" (2026-03-09 10:15, Dr. Petrov)
    ↑ Причина: "Исправление опечатки в диагнозе"
```

---

### Контроль 3: RBAC на уровне специальности

**Что:** Врач видит только пациентов своей специальности  
**Где:** API-layer + Database layer  
**Как:**
```
Dr. Петrov (Cardiology):
  ├─ Может прочитать: пациентов с кардиологическими диагнозами
  ├─ Может создать: диагнозы по кардиологии (МКБ-коды I**)
  └─ Не может прочитать: пациентов гастроэнтеролога (К**)

Policy (OPA):
  allow {
    input.user.role == "Specialist"
    input.user.specialty == "Cardiology"
    input.action == "read"
    input.resource == "medical_records"
    input.record.icd10 ~ "^I"  -- МКБ коды кардиологии
  }

Database layer:
  SELECT mr.* FROM medical_records mr
  WHERE mr.doctor_id = 3  -- только свои пациенты
    AND SUBSTRING(mr.icd10_code, 1, 1) IN ('I')  -- только свои диагнозы
```

---

### Контроль 4: Логирование доступа с HMAC

**Что:** Неизменяемое логирование + проверка целостности  
**Где:** audit_log таблица  
**Как:**
```
-- Логирование события:
INSERT INTO audit_log (
  user_id, action, patient_id, resource_type, resource_id,
  before_state, after_state, ip_address, timestamp
) VALUES (
  3, 'read_medical_record', 42, 'medical_record', 5000,
  NULL,  -- No "before" for read
  json_build_object('diagnosis', '<encrypted>', 'doctor', 'Petrov'),
  '192.168.1.50',
  NOW()
);

-- Вычисление HMAC для проверки целостности:
hmac_value = HMAC_SHA256(
  secret_key = 'audit_log_secret_key_from_HSM',
  message = json_encode({
    user_id: 3,
    action: 'read_medical_record',
    timestamp: NOW(),
    ...
  })
);

-- Сохраняем HMAC:
UPDATE audit_log SET hmac = hmac_value WHERE id = <latest>;

-- Верификация целостности логов:
SELECT COUNT(*) FROM audit_log WHERE 
  HMAC_SHA256('audit_log_secret_key', log_entry) != log.hmac;
  
If COUNT > 0 → Logs were tampered with!
```

---
