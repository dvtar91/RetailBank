## Задание 3 – Разработка архитектуры AI‑сервиса для RetailBank  

---

### Общее содержание  

| Инициатива | Тип решения | Краткое описание | Статус |
|------------|------------|------------------|-----------------|
| **F** – Выявление подозрительных операций (транзит, обналичивание, дробление) | Табличный скоринг + правила AML | Конвейер **признаки → модель → калибровка/порог → детерминированные правила → human‑in‑the‑loop** | Готово (C4‑диаграммы, таблицы, контракт, fallback, мониторинг, угрозы) |
| **A** – OCR‑оцифровка документов к кредитной заявке | OCR + NLP‑extraction + валидация | **Документ → OCR → NLP‑extraction → проверка схемы → human‑review (при низкой уверенности/ошибке)** | Готово (Container‑диаграмма, контракт, fallback, мониторинг) |
| **B** – Прогноз оттока клиентов | Текстовый классификатор + правила роутинга | **Текст → предобработка → классификатор → правила → human‑review, если необходимо** | Готово (Container‑диаграмма, контракт, fallback, мониторинг) |

---

### Контейнерные и компонентные C4‑диаграммы  

#### Флагман — Выявление подозрительных операций (табличный скоринг)  

```mermaid
C4Context
title RetailBank – Флагман: Выявление подозрительных операций (AML)

Person(customer, "Клиент")
System_Boundary(rb, "RetailBank Core") {
    Container(Db, "Операционная БД", "PostgreSQL", "Транзакции, справочники, KYC‑данные")
    Container(KYC, "Внешняя KYC‑платформа", "REST (TLS)", "Риски клиента, статусы")
    Container(API, "Frontend / API", "Spring Boot / Node", "Внутренний API мониторинга")
}
System_Boundary(ai, "AI Service") {
    Container(Queue, "Message Queue", "Kafka", "Буфер запросов на проверку")
    Container(FeatureSrv, "Feature Extraction Service", "Java/Scala", "Формирование детерминированных признаков")
    Container(ScoreModel, "ML Scoring Model", "Python (LightGBM)", "Расчёт `riskScore` (0‑1) и `priorityScore`")
    Container(Rules, "Deterministic Rules Engine", "Drools", "Бизнес‑правила AML (не используют пороги модели) ")
    Container(HumanUI, "Human Review UI", "React", "Экран аналитика")
}
Rel(customer, API, "Отправка операции")
Rel(API, Queue, "POST /risk‑check")
Rel(Queue, FeatureSrv, "Consume")
Rel(FeatureSrv, Db, "Read ops + справочники")
Rel(FeatureSrv, KYC, "Pull risk signals")
Rel(FeatureSrv, ScoreModel, "JSON‑features")
Rel(ScoreModel, Rules, "riskScore + priorityScore")
Rel(Rules, API, "finalDecision (accept / reject / hold)")
Rel(Rules, HumanUI, "hold / low‑confidence → запрос проверки")
Rel(HumanUI, API, "humanVerdict")
Rel(Rules, Db, "store decision history")
```

```mermaid
C4Container
title RetailBank – Компоненты AI‑сервиса (Флагман)

Container_Boundary(ai, "AI Service") {
    Component(API, "REST API `/risk‑check`", "Spring/Node", "Приём запросов, валидация JSON")
    Component(Queue, "Message Queue", "Kafka", "Буфер запросов")
    Component(FeatureExtractor, "Feature Extraction", "Java/Scala", "Сбор и агрегирование признаков")
    Component(ScoreModel, "ML Scoring Model", "Python (LightGBM)", "Выдаёт `riskScore` (0‑1) и `priorityScore` (0‑100)")
    Component(RulesEngine, "Rules Engine", "Drools", "Бизнес‑правила AML (полностью детерминированы)")
    Component(HumanRouter, "Human Review Trigger", "Java", "Определяет необходимость human‑in‑the‑loop (low‑confidence, OOD, priority > 80)")
    Component(HumanUI, "Human Review UI", "React", "Экран аналитика")
    Component(Monitoring, "Monitoring", "Prometheus/Loki", "Метрики, алерты, трассировка")
}
Rel(API, Queue, "enqueue request")
Rel(Queue, FeatureExtractor, "consume")
Rel(FeatureExtractor, ScoreModel, "features JSON")
Rel(ScoreModel, RulesEngine, "riskScore + priorityScore")
Rel(RulesEngine, HumanRouter, "hold? / low‑confidence / priority")
Rel(HumanRouter, HumanUI, "display for review")
Rel(HumanUI, RulesEngine, "humanVerdict")
Rel(RulesEngine, API, "finalDecision")
Rel(Monitoring, FeatureExtractor, "metrics")
Rel(Monitoring, ScoreModel, "metrics")
Rel(Monitoring, RulesEngine, "metrics")
Rel(Monitoring, HumanUI, "metrics")
```

#### Инициатива A – OCR‑оцифровка документов к кредитной заявке  

```mermaid
C4Context
title RetailBank – OCR‑оцифровка документов (инициатива A)

Person(customer, "Клиент")
System_Boundary(rb, "RetailBank Core") {
    Container(Db, "Операционная БД", "PostgreSQL", "Метаданные заявки, справочники")
    Container(API, "Frontend / API", "Spring / Node", "Приём файлов")
}
System_Boundary(aiDocs, "AI Service – Документы") {
    Container(DocQueue, "Message Queue – Docs", "Kafka", "Очередь файлов")
    Container(OCRSrv, "OCR Service", "Tesseract (GPU)", "Распознавание текста")
    Container(NLPExtract, "NLP Extraction", "Python (BERT)", "Извлечение полей из текста")
    Container(Validator, "Deterministic Validation", "Java", "Проверка схемы, бизнес‑правила")
    Container(HumanDocUI, "Human Review UI – Docs", "React", "Экран операторов")
}
Rel(customer, API, "POST /documents (PDF/JPEG)")
Rel(API, DocQueue, "enqueue file")
Rel(DocQueue, OCRSrv, "consume")
Rel(OCRSrv, NLPExtract, "plain text")
Rel(NLPExtract, Validator, "extracted fields")
Rel(Validator, API, "auto‑accept (if OK)")
Rel(Validator, HumanDocUI, "hold / low confidence")
Rel(HumanDocUI, API, "approved / corrected")
Rel(OCRSrv, Db, "optional metadata")
Rel(NLPExtract, Db, "store extracted fields")
```

#### Инициатива B – Прогноз оттока клиентов (Churn)  

```mermaid
C4Context
title RetailBank – Прогноз оттока клиентов (инициатива B)

Person(customer, "Клиент")
System_Boundary(rb, "RetailBank Core") {
    Container(API, "Frontend / API", "Spring / Node", "Приём события/профиля клиента")
}
System_Boundary(aiChurn, "AI Service – Churn") {
    Container(ChurnQueue, "Message Queue – Churn", "Kafka", "Очередь запросов на прогноз оттока")
    Container(Preproc, "Feature Service – Churn", "Java", "Подготовка признаков (агр., нормал.)")
    Container(ChurnModel, "ML Churn Model", "Python (XGBoost)", "Прогноз вероятности оттока")
    Container(ChurnRules, "Deterministic Churn Rules", "Drools", "Правила бизнес‑приоритетов (VIP, регуляция)")
    Container(ChurnHumanUI, "Human Review UI – Churn", "React", "Экран аналитика при low‑confidence")
}
Rel(customer, API, "POST /customer‑event")
Rel(API, ChurnQueue, "enqueue request")
Rel(ChurnQueue, Preproc, "consume")
Rel(Preproc, ChurnModel, "features JSON")
Rel(ChurnModel, ChurnRules, "prediction + confidence")
Rel(ChurnRules, API, "auto‑route")
Rel(ChurnRules, ChurnHumanUI, "hold / low confidence")
Rel(ChurnHumanUI, API, "final decision")
```

---

### Таблица ответственности компонентов (RACI)

| Компонент | R (Ответственный) | C (Консультируемый) | I (Информируемый) | A (Выполняющий) |
|-----------|-------------------|---------------------|-------------------|-----------------|
| **API (Frontend / API)** | Team API (Backend) | Product Owner, AML Team, Compliance | Аналитики, SRE | Team API |
| **Message Queue (Kafka)** | Platform Ops | Team API, Data Engineering | Все команды | Platform Ops |
| **Feature Extraction Service** | Data Engineering | AML Team, Compliance | Product Owner | Data Engineering |
| **ML Scoring Model** | Data Science – AML | Feature Service, Compliance | Product Owner | Data Science |
| **Calibration & Threshold Service** | Data Science – AML | Business Analyst | Product Owner | Data Science |
| **Deterministic Rules Engine** | Business Rules Team | Compliance, Risk | Product Owner | Business Rules Team |
| **Human Review UI** | UX / Frontend Team | Business Analyst, Compliance | Product Owner | UX / Frontend Team |
| **OCR Service** | Document Engineering | Compliance, Legal | Product Owner | Document Engineering |
| **NLP Extraction** | NLP Team (Data Science) | Document Engineering | Product Owner | NLP Team |
| **Doc Validation Rules** | Business Rules Team | Compliance | Product Owner | Business Rules Team |
| **Churn Feature Service** | Data Engineering – Churn | Marketing, Risk | Product Owner | Data Engineering |
| **Churn Model** | Data Science – Churn | Feature Service | Product Owner | Data Science |
| **Churn Rules Engine** | Business Rules Team – Churn | Marketing, Compliance | Product Owner | Business Rules Team |
| **Churn Human Review UI** | UX / Frontend Team – Churn | Business Analyst, Marketing | Product Owner | UX / Frontend Team |
| **Monitoring (Prometheus + Loki)** | SRE / Platform Ops | Все команды | Руководство | SRE |

---

### Выходные контракты (Response Schemas)

#### Флагман – Выявление подозрительных операций  

```json
{
  "requestId": "string (uuid)",
  "operationId": "string",
  "decision": "enum[queued, accept, reject, hold]",
  "riskScore": "number (0‑1)",               // вывод модели – используется только для аналитики
  "priorityScore": "number (0‑100)",          // скалированный риск, определяет порядок в очереди
  "modelVersion": "string",
  "featuresUsed": ["list of feature names"],
  "ruleTriggers": ["list of rule identifiers (optional)"],
  "humanReviewRequired": "boolean",
  "humanVerdict": "enum[accept, reject, pending] | null",
  "timestamp": "ISO‑8601",
  "source": {
    "operationRecordId": "string",
    "kcySignalIds": ["array of ids"]
  },
  "validation": {
    "featuresValid": "boolean",
    "missingFeatures": ["array (optional)"]
  },
  "audit": {
    "processingTimeMs": "integer",
    "queueDelayMs": "integer"
  }
}

```

*Обязательные поля:* `requestId, operationId, decision, score, modelVersion, timestamp, source`.  
Если один из обязательных признаков недоступен → `decision: hold`, `humanReviewRequired: true`.

#### OCR‑оцифровка документов  

```json
{
  "requestId": "string (uuid)",
  "documentId": "string",
  "extractedFields": {
    "fieldName": "value",
    "...": "..."
  },
  "confidence": "number (0‑1)",
  "modelVersion": "string",
  "validation": {
    "schemaValid": "boolean",
    "missingFields": ["array (optional)"]
  },
  "humanReviewRequired": "boolean",
  "humanVerdict": "enum[accept, reject, needsCorrection] | null",
  "timestamp": "ISO‑8601"
}
```

*Если `confidence < 0.8` **или** обнаружены конфликты полей → `humanReviewRequired: true`.*

#### Прогноз оттока (Churn)  

```json
{
  "requestId": "string (uuid)",
  "customerId": "string",
  "predictedProbability": "number (0‑1)",
  "confidence": "number (0‑1)",
  "modelVersion": "string",
  "ruleApplied": "boolean",
  "humanReviewRequired": "boolean",
  "humanDecision": "enum[retain, ignore, escalated] | null",
  "timestamp": "ISO‑8601"
}
```

*Если `confidence < 0.6` **или** правило «VIP‑клиент» конфликтует с прогнозом → `humanReviewRequired: true`.*

---

### Уровень автоматизации  

| Сценарий | Что делает модель | Что делают правила | Где обязателен человек |
|----------|------------------|--------------------|------------------------|
| **Флагман – обычный поток** | Выдаёт `score` (0‑1) | Порог `accept < 0.45`, `reject > 0.90`, + AML‑правила | Если `score ∈ [0.45‑0.55]` **или** `confidence < 0.7` → *Human Review* |
| **Флагман – экстремальный‑порог** | `score > 0.95` → автоматический `reject` | Запись в журнал, уведомление | Нет |
| **OCR** | Распознаёт текст, извлекает поля | Проверка формата, контрольные суммы, бизнес‑правила | `confidence < 0.8` **или** конфликт → *Human Review* |
| **Churn** | Прогнозирует вероятность оттока | Если вероятность > 0.8 → «приоритет удержания», иначе «стандарт»; дополнительные правила (VIP, регуляция) | `confidence < 0.6` **или** правило‑override → *Human Review* |

---

### Fallback‑стратегии  

| № | Признак отклонения | Действие | Куда направляется |
|---|--------------------|----------|-------------------|
| **1** | **Отсутствие/устаревание признаков** (FeatureValidator) | `humanReviewRequired = true` → **Human Review UI** | Human Review UI |
| **2** | **Out‑of‑distribution (OOD)‑детекция** (признаки за пределами обучающего диапазона) | Пометить `humanReviewRequired = true`, но сохранить `priorityScore` | Human Review UI |
| **3** | **Новый клиент без истории** | Увеличить порог `priorityScore` = 80 → Human Review | Human Review UI |
| **4** | **Зона «зазор»** (`priorityScore` ∈ [70‑80]) | Требовать подтверждения человека | Human Review UI |
| **5** | **Разногласие модели и детерминированного правила** | Приоритет правил → **Rule‑Only** (не зависимо от `riskScore`) | Rules Engine (автоматическое `accept`/`reject`/`hold`) |
| **6** | **Недоступность модели / сервиса** | Переключить в **Rule‑Only** режим | Rules Engine |
| **7** | **Неверный JSON‑ответ модели** | `400 Bad Request`, логировать | API‑клиент |
| **8** | **OCR‑схема не прошла валидацию** | Human Review (показ оригинала + извлечённые поля) | Doc Human Review UI |
| **9** | **Противоречивые документы** | Human Review с пометкой «конфликт» | Doc Human Review UI |
| **10** | **Запрос вне области применимости (Churn)** | `403 Forbidden` + ссылка на справку | API‑клиент |
| **11** | **Отказ внешнего сервиса (KYC, OCR SaaS)** | Перейти в **offline‑mode** → только локальные правила | Rules Engine |
| **12** | **Превышение лимитов очереди** | Дрейн‑режим: новые запросы сразу в Human Review | Human Review UI |
| **13** | **Обнаружена XSS/малициозный файл (OCR)** | Блокировать, вернуть `415 Unsupported Media Type` | API‑клиент |

---

### План мониторинга  

| Уровень | Метрика | Порог тревоги | Действие |
|----------|---------|---------------|----------|
| **API** | `request_rate`, `error_rate (4xx/5xx)` | > 200 req/s, > 2 % 5xx | Авто‑скейлинг, алерт SRE |
| **Очереди** | `queue_length`, `lag_ms` | > 10 000 сообщений, `lag_ms > 5 000` | Увеличить concurrency, алерт |
| **Feature Extraction** | `extraction_time_ms`, `missing_features_rate` | > 200 ms, > 5 % missing | Перезапуск, проверка источников |
| **FeatureValidator / OOD** | `featureValidatorErrors`, `oodDetectionRate` | > 2 % всего | Алерт, эскалация к Data‑Science |
| **ML Model** | `model_latency_ms`, `priorityScoreDistribution`, `drift_score (PSI)` | > 150 ms, priorityScore > 80 % запросов, PSI > 0.1 | Перевести модель в **read‑only**, запустить пере‑обучение |
| **Rules Engine** | `rule_trigger_rate`, `rule_reject_rate` | > 30 % hold‑rate, > 40 % reject‑rate без human‑review | Пересмотр бизнес‑правил |
| **Human Review** | `avg_review_time`, `queue_backlog_human`, `escalation_rate` | > 15 min, backlog > 500, escalations > 10 % | Поднять staffing, пересмотреть SLA |
| **Бизнес‑метрики** | `falsePositiveRate`, `falseNegativeRate`, `blocked_transactions_per_month` | FP > 3 %, FN > 1 % | Пересчитать ROI, откат модели |
| **Безопасность** | `sensitive_data_leak_events`, `unauthorized_access_attempts` | > 0 | Инцидент‑процесс, немедленная блокировка |

*Автоматический откат* – если `drift_score` превышает порог, система переводит модель в **read‑only**, генерирует задачу пере‑обучения и **переключает поток** на **Rule‑Only** режим (см. fallback № 6).  

---

### Границы доверия + таблица угроз  

| № | Граница (на схеме) | Тип данных | Угроза | Мера снижения |
|---|----------------------|------------|--------|----------------|
| **1** | Frontend → API (внешний запрос) | Неаутентифицированный JSON | Инъекция, DoS, подделка | JWT‑подпись, rate‑limiting, JSON‑schema validation |
| **2** | FeatureSrv ← KYC (REST) | Риск‑сигналы KYC | Подмена / перехват | TLS 1.3, HMAC подпись запросов, контрольные суммы |
| **3** | Model ← FeatureSrv (признаки) | Признаки операции | Adversarial‑атака, подмена | HMAC подпись payload, whitelist признаков, контроль целостности |
| **4** | RulesEngine ← Model (riskScore + priorityScore) | Вывод модели | Rule‑poisoning, конфликт правил | Тест‑suite правил, ограничение прав изменения |
| **5** | Client → API (документ) | PDF/JPEG (неструктурированный) | Вредоносный файл, steganography, XSS | Проверка MIME‑type, антивирус‑скан, лимит ≤ 10 MB |
| **6** | OCRSrv ← документ | Необработанное изображение | Оффлайн‑атака, скрытый код | Sandbox‑окружение, ограничение прав процесса OCR |
| **7** | Classifier ← текст (Churn) | Свободный текст | Prompt‑injection, языковые атаки | Очистка/экранирование, whitelist‑словарей, ограничение длины |
| **8** | External AML/KYC API → FeatureSrv | Внешний API‑ответ | Перехват / модификация | TLS, подпись API‑ключей, проверка хэша ответа |

---

### Краткие выводы и рекомендации бизнесу  

| Пункт | Вывод |
|-------|-------|
| **Модель** | **Только приоритет** (riskScore + priorityScore).  Нет автоматических `reject`/`accept`. |
| **Бизнес‑правила** | Остаются **единственным** источником окончательного решения; они могут переопределять модельный риск. |
| **Human‑review** | Срабатывает при **низкой уверенности**, **OOD‑детекции**, **высоком приоритете** или **отсутствии признаков**. |
| **Мониторинг** | Добавлен контроль `priorityScoreDistribution` и `oodDetectionRate`. |
| **Риски** | Привязаны к границам доверия (пунктирные линии).  Меры (TLS, HMAC, sandbox, валидация) уже реализованы. |
| **Тестирование** | Пилотный запуск в режиме **shadow‑mode**: модель генерирует приоритет, но решения принимаются только правилами и human‑review.  После подтверждения KPI (корректность приоритета, отсутствие роста FN) – полное включение модели в прод. |
