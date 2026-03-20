# Task 5: Стратегия миграции и Cutover-план

## ВЫБРАННАЯ СТРАТЕГИЯ: PARALLEL RUN (Параллельный запуск)

### Обоснование выбора

| Критерий | High-level Design | Branch by Abstraction | **Parallel Run** |
|----------|------------------|----------------------|-----------------|
| **Риск потери данных** | Высокий (одноразовая миграция) | Средний (многоэтапно) | **Низкий** (параллельная работа) |
| **Время простоя системы** | 4-8 часов | Недели простоев | **0 часов** (нет простоя) |
| **Сложность синхронизации** | Простая | Сложная | **Управляемая** |
| **Откат при ошибке** | Невозможен | Сложен | **Быстрый** (переход на As-Is) |
| **Тестирование в prod** | Нет | Частичное | **Полное** (2-3 недели) |
| **Сроки** | 3-4 недели | 8-12 недель | **16 недель** |

**Для Медикаменте это оптимально, потому что:**
- Критична непрерывность работы (пациенты не могут ждать)
- Большой объём данных требует валидации
- Сложная интеграция с 1C требует тестирования в разных условиях
- Высокие требования compliance (152-ФЗ) не допускает потерь данных

---

## CUTOVER-ПЛАН: ПОЭТАПНАЯ СТРУКТУРА

### ФАЗА 0: ПОДГОТОВКА (Недели 1-2)

| Этап | Описание действия | Ключевые задачи | Ответственные | Время/Сроки | Риски и меры по снижению |
|------|-------------------|-----------------|---------------|-------------|--------------------------|
| **0.1 Инфраструктура** | Развёртывание серверов и сетей | Upgrade сеть, добавить серверы, хранилище данных, Kubernetes pilot | CTO + Infrastructure | Неделя 1 | *Риск:* Network downtime, *Мера:* Фазовое обновление во время off-peak |
| **0.2 Безопасность** | Дизайн и реализация Vault, шифрования | Настройка Vault, генерация ключей, RBAC в Keycloak | Security Consultant + DBA | Неделя 1-2 | *Риск:* Ключи потеряны, *Мера:* HMAC backup в зашифрованном хранилище |
| **0.3 DB миграция** | Миграция пациентов | ETL pipeline (Python + Airflow), dry-run 10x, валидация | DBA + Dev Lead | Неделя 1-2 | *Риск:* Data corruption, *Мера:* 3x тестирование, rollback script готов |
| **0.4 Команда** | Найм и обучение | обучение микросервисам | HR + CTO | Неделя 1 | *Риск:* Staff не готов, *Мера:* Внешние консультанты на первую неделю |

---

### ФАЗА 1: PILOT (Недели 3-6)

| Этап | Описание действия | Ключевые задачи | Ответственные | Время/Сроки | Риски и меры по снижению |
|------|-------------------|-----------------|---------------|-------------|--------------------------|
| **1.1 K8s Staging** | Развёртывание Kubernetes кластера | 3 master + 3 worker nodes, networking, RBAC, helm charts | DevOps + Architect | Неделя 3 | *Риск:* Cluster unstable, *Мера:* Redundancy тестирование, failover test |
| **1.2 Core Services** | Развёртывание Payment + Lab services | Docker images, service mesh (Istio), health checks, monitoring | Dev Team (2 микросервиса) | Неделя 3-4 | *Риск:* Service startup fail, *Мера:* Auto-rollback, canary deployment |
| **1.3 Data Warehouse** | Развёртывание Analytics Layer | Snowflake schema, ETL pipeline (Airflow), masking rules | Data Engineer | Неделя 4-5 | *Риск:* ETL jobs fail, *Мера:* DLQ (dead letter queue) monitoring |
| **1.4 Testing** | UAT с реальными бизнес-юзерами | Payment flow E2E, Lab order workflow, data validation | QA + Business Users | Неделя 5-6 | *Риск:* Найдены критичные баги, *Мера:* Bugfix + re-test, freeze if P1 bugs |

---

### ФАЗА 2: PARALLEL RUN — PART 1 (Недели 7-8)

| Этап | Описание действия | Ключевые задачи | Ответственные | Время/Сроки | Риски и меры по снижению |
|------|-------------------|-----------------|---------------|-------------|--------------------------|
| **2.1 Dual-write** | Запуск As-Is + To-Be параллельно | Развёртывание Patient + Appointment services, включение синхронизации (Excel ↔ PostgreSQL) | Dev Team + DBA | Неделя 7 | *Риск:* Данные diverge, *Мера:* Reconciliation job каждый час, alarm на lag > 5min |
| **2.2 Sync Validation** | Проверка синхронизации данных | Мониторинг новых пациентов (In Excel и To-Be), двойной контроль приёмов | Operations + QA | Неделя 7-8 | *Риск:* Silent data loss, *Мера:* HMAC checksums на каждой синхронизации |
| **2.3 Medical Records** | Миграция медицинских карт | Deploy Medical Records service, sync от As-Is, encryption validation | Dev Team + DBA | Неделя 7-8 | *Риск:* Потеря истории пациента, *Мера:* Backup исходных файлов, audit log всех операций |
| **2.4 1C Integration** | Запуск real-time sync с 1C Бухгалтерия | API gateway, batch + real-time sync, reconciliation reports | Integration Lead + 1C specialist | Неделя 8 | *Риск:* 1C API недоступна, *Мера:* Fallback на файловый batch, очередь сообщений |

---

### ФАЗА 3: PARALLEL RUN — PART 2 (Недели 9-10)

| Этап | Описание действия | Ключевые задачи | Ответственные | Время/Сроки | Риски и меры по снижению |
|------|-------------------|-----------------|---------------|-------------|--------------------------|
| **3.1 Load Testing** | Тест нагрузки на To-Be систему (5x growth) | Симуляция 5x пациентов, appointment volume, payment processing | QA + Performance team | Неделя 9 | *Риск:* Bottleneck в БД, *Мера:* Query optimization, caching layer |
| **3.2 Compliance Audit** | Проверка 152-ФЗ compliance | Audit logs complete, encryption verified, RBAC tested | Security + Compliance officer | Неделя 9 | *Риск:* Non-compliance найден, *Мера:* Fix gap, re-audit перед cutover |
| **3.3 Staff Training** | Обучение всего персонала на новой системе | Reception на patient portal, doctors на medical records, accountants на payment reconciliation | Training Team + Super users | Неделя 9-10 | *Риск:* Staff не готов, *Мера:* Checklists, hotline support 24/7 during cutover |
| **3.4 Rollback Drill** | Тренировка отката на As-Is | Сценарий: To-Be недоступна, откат на Excel за 30 мин | DevOps + Operations | Неделя 10 | *Риск:* Rollback fail, *Мера:* Dry-run 3x, scripts автоматизированы, tested |

---

### ФАЗА 4: CUTOVER (День X, Неделя 11)

| Этап | Описание действия | Ключевые задачи | Ответственные | Время/Сроки | Риски и меры по снижению |
|------|-------------------|-----------------|---------------|-------------|--------------------------|
| **4.1 Pre-cutover** | Финальная проверка перед переключением | Stop As-Is writes (Excel), final reconciliation, backup As-Is database | DBA + Ops | День X, 08:00 | *Риск:* Последние данные потеряны, *Мера:* WAL-based PITR готов |
| **4.2 Final Sync** | Последняя синхронизация всех данных | Sync остатки (Excel→PostgreSQL), закрытие Excel сессий | Sync Team | День X, 09:00-10:00 | *Риск:* Sync зависает, *Мера:* Kill timeout 5min, manual cleanup |
| **4.3 Switchover** | Переключение трафика на To-Be | DNS switch (если облако) или firewall rules (если on-prem), test health checks | Infrastructure Team | День X, 10:00-10:15 | *Риск:* Partial traffic switch, *Мера:* Blue-green setup, instant rollback button |
| **4.4 Validation** | Проверка что To-Be работает | Первые приёмы, платежи, запросы в 1C, audit logs пишутся | QA + Operations | День X, 10:15-12:00 | *Риск:* Critical error найден, *Мера:* Instant rollback к Phase 3 state |
| **4.5 Go/No-Go** | Решение: остаём на To-Be или откатываемся | Business sign-off, CTO decision | CTO + CEO | День X, 12:00 | *Риск:* Нерешительность, *Мера:* Предсказанные criterea (P0 bugs = rollback) |

---

### ФАЗА 5: STABILIZATION (Недели 12-16)

| Этап | Описание действия | Ключевые задачи | Ответственные | Время/Сроки | Риски и меры по снижению |
|------|-------------------|-----------------|---------------|-------------|--------------------------|
| **5.1 Excel Decommission** | Отключение As-Is системы (Excel) | Архивирование данных, удаление сетевых диск, документирование | Operations | Неделя 12 | *Риск:* Data loss при удалении, *Мера:* 3-месячный архив в S3 |
| **5.2 Monitoring** | 24/7 мониторинг To-Be в production | ELK, Prometheus, Splunk SIEM, on-call rotation | DevOps + Ops | Недели 12+ | *Риск:* Issue не заметили, *Мера:* Automated alerts, chat notifications |
| **5.3 Performance Tuning** | Оптимизация To-Be на основе реальных нагрузок | Query tuning, caching, connection pooling | DB Architect + Dev | Недели 13-14 | *Риск:* Slow queries, *Мера:* Index tuning, schema redesign если needed |
| **5.4 Hardening** | Security hardening и patch management | Vulnerability scanning, SSL cert rotation, password rotation, security training | Security Team | Недели 14-16 | *Риск:* Zero-day exploit, *Мера:* WAF rules updated, incident response tested |

---

## РАСПРЕДЕЛЕНИЕ РОЛЕЙ И ОТВЕТСТВЕННОСТИ

### Ключевые роли:

| Роль | Ответственность | Примеры задач |
|------|------------------|--------------|
| **CTO** | Общее руководство, архитектурные решения, go/no-go | Фазовый переход, rollback decision, staff escalation |
| **DBA** | Data migration, integrity, backup/restore | ETL testing, reconciliation, encryption setup |
| **DevOps Lead** | Infrastructure, Kubernetes, deployment | Server setup, K8s pilot, blue-green switch |
| **Dev Lead** | Microservices development, code review | API contracts, service deployment, testing |
| **Integration Lead** | 1C API, data sync, external systems | 1C API design, sync validation, fallback testing |
| **Security Consultant** | Encryption, RBAC, compliance audit | Vault setup, audit logs, 152-ФЗ checklist |
| **QA Lead** | Testing, UAT, rollback testing | Test cases, load testing, rollback drills |
| **Operations Manager** | Day-to-day monitoring, incident response | On-call rotation, monitoring setup, escalation |

---

## ПЛАН УПРАВЛЕНИЯ РИСКАМИ

### Top 5 рисков и меры снижения:

| Риск | Вероятность | Влияние | Мера снижения | Ответственный |
|------|------------|---------|---------------|--------------|
| **Data loss при миграции** | Средняя | КРИТИЧНОЕ | 3x тестирование, backup, HMAC checksums | DBA |
| **Неудача синхронизации As-Is + To-Be** | Средняя | КРИТИЧНОЕ | Reconciliation job каждый час, alarm < 5min lag | Integration Lead |
| **1C API недоступна в день X** | Низкая | ВЫСОКОЕ | Fallback на batch файлы + queue, 2-часовой буфер | Integration Lead |
| **Critical bug найден после cutover** | Средняя | КРИТИЧНОЕ | Instant rollback скрипт (5 мин), Phase 3 backup | DevOps Lead |
| **Staff не готов к новой системе** | Низкая | СРЕДНЕЕ | Training program недели 9-10, hotline 24/7, checklists | Training Team |

---