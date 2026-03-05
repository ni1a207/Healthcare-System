# Kafka — реестр событий

## Назначение
Распределённый брокер сообщений, обеспечивающий асинхронное взаимодействие между сервисами системы. В текущей реализации используется для передачи событий жизненного цикла пациентов от `patient-service` к `analytics-service`.

---

## Конфигурация брокера

| Параметр | Docker (local) | Infra (AWS MSK / LocalStack) |
| :--- | :--- | :--- |
| **Image** | bitnami/kafka | AWS MSK |
| **Kafka Version** | — | 2.8.0 |
| **Broker Nodes** | 1 | 1 |
| **Instance Type** | — | kafka.m5.xlarge |
| **Cluster Name** | — | kafa-cluster |
| **PLAINTEXT Port** | 9092 | — |
| **External Port** | 9094 | — |
| **Controller Port** | 9093 | — |
| **Bootstrap Servers** | `kafka:9092` | `localhost.localstack.cloud:4510, 4511, 4512` |

> **Примечание:** Имя кластера `kafa-cluster` содержит опечатку в исходном коде (`LocalStack.java`). Указано as-is.

---

## Конфигурация listeners (Docker)

Из переменных окружения Kafka-контейнера:

| Listener | Протокол | Адрес | Назначение |
| :--- | :--- | :--- | :--- |
| `PLAINTEXT` | PLAINTEXT | `kafka:9092` | Внутренний трафик (между сервисами в Docker-сети) |
| `CONTROLLER` | PLAINTEXT | `kafka:9093` | KRaft controller (внутренний, не для клиентов) |
| `EXTERNAL` | PLAINTEXT | `localhost:9094` | Внешний доступ (с хост-машины) |

---

## Реестр топиков

| Топик | Partitions | Replication Factor | Формат | Retention | Описание |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `patient` | 1 (default) | 1 (default) | Protobuf | 7 дней (default) | События жизненного цикла пациентов |

> **Примечание:** Топик создаётся автоматически при первой публикации. Явная конфигурация топика (`num.partitions`, `retention.ms`) в коде отсутствует — используются значения по умолчанию брокера.

---

## Schema Registry

### PatientEvent

Файл: `patient_event.proto`
Пакет: `patient.events`

```protobuf
syntax = "proto3";

package patient.events;
option java_multiple_files = true;

message PatientEvent {
  string patientId = 1;
  string name      = 2;
  string email     = 3;
  string event_type = 4;
}
```

#### Поля сообщения

| Поле | Тип | Field Number | Обязательность | Описание | Пример |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `patientId` | string | 1 | Да | UUID пациента | `550e8400-e29b-41d4-a716-446655440000` |
| `name` | string | 2 | Да | ФИО пациента | `John Doe` |
| `email` | string | 3 | Да | Email пациента | `john.doe@example.com` |
| `event_type` | string | 4 | Да | Тип события | `PATIENT_CREATED` |

#### Producers и Consumers

| Роль | Сервис | Класс | Топик | Ключ (Key) |
| :--- | :--- | :--- | :--- | :--- |
| **Producer** | `patient-service` | `KafkaProducer` | `patient` | `null` |
| **Consumer** | `analytics-service` | `KafkaConsumer` | `patient` | не обрабатывается |

---

## Матрица трассировки событий

| Event Type | Топик | Producer | Consumer | Формат | Триггер | Спецификация                                                                                                             |
| :--- | :--- | :--- | :--- | :--- | :--- |:-------------------------------------------------------------------------------------------------------------------------|
| `PATIENT_CREATED` | `patient` | `patient-service` | `analytics-service` | Protobuf | Успешное создание пациента в БД + gRPC-вызов к billing-service | [PatientEventProducer](..%2FAPI%2FPatientEventProducer.md)<br/>[PatientEventConsumer](..%2FAPI%2FPatientEventConsumer.md)|

---

## Известные проблемы

| # | Проблема | Затронутый сервис | Описание |
| :--- | :--- | :--- | :--- |
| **1** | **Silent Data Loss (Producer)** | `patient-service` | При недоступности Kafka `kafkaTemplate.send()` выбрасывает исключение, которое поглощается блоком `catch`. Клиент получает `200 OK`. Данные в `patient_db` сохранены, gRPC-вызов выполнен. Сообщение в Kafka безвозвратно теряется — возникает Data Inconsistency. |
| **2** | **Silent Data Loss (Consumer)** | `analytics-service` | При ошибке десериализации `InvalidProtocolBufferException` перехватывается, логируется только текст ошибки без stacktrace. Spring Kafka выполняет авто-коммит offset — сообщение считается обработанным и безвозвратно теряется. |
| **3** | **No DLQ** | `patient-service`, `analytics-service` | Dead Letter Queue не настроен. Потерянные сообщения не сохраняются и не могут быть воспроизведены. |
| **4** | **No Retry Policy** | `patient-service` | Механизм повторной отправки (`retries`, `retry.backoff.ms`) не настроен в конфигурации producer. |
