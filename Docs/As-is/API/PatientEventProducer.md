# Kafka Point (Producer) - PatientEvent

## Назначение

Событие публикуется сервисом `patient-service` при успешном создании новой карточки пациента в базе данных и после создания биллингового аккаунта через gRPC. Событие уведомляет заинтересованные сервисы о появлении нового пользователя.

---

## Общие сведения
**Producer:** `patient-service`

**Topic:** `patient`

**Protocol:** `TCP`

**Format message:** `Protobuf` 

---

## Структура сообщения (Схема Protobuf)
Параметры файла `PatientEvent.proto`:

| Параметр | Тип данных | Обязательность | Описание | Значение/пример| Маппинг с patient_db |
| --- | --- | --- | --- |-|-----------------|
| patientId | string | Да | Уникальный идентификатор пациента (UUID) |550e8400-e29b-41d4-a716-446655440000| patient.id                |
| name | string | Да | ФИО пациента |John Doe| patient.name                |
| email | string | Да | Электронная почта пациента |john.doe@example.com| patient.email              |
| event_type | string | Да | Тип события |PATIENT_CREATED| Hardcoded value                |

---

## Пример сообщения (JSON до сериализации)
~~~json 
{
"patientId": "550e8400-e29b-41d4-a716-446655440000",
"name": "John Doe",
"email": "john.doe@example.com",
"event_type": "PATIENT_CREATED"
}   
~~~ 

---

## Алгоритм публикации события
Публикация происходит в процессе выполнения метода [POST /patients](POST.patients.md) .

1. Триггер: успешное сохранение записи в `patient_db` и успешный gRPC-вызов к `billing-service`. `PatientService` вызывает метод `sendEvent(patient)` в `KafkaProducer`, передавая уже готовый объект `Patient`.


2. `KafkaProducer` формирует объект `PatientEvent` через `Protobuf builder`: `PatientEvent.newBuilder().setPatientId(...).setName(...).setEmail(...).setEventType("PATIENT_CREATED").build()`. Данные берутся напрямую из переданного объекта `Patient` — обращения к БД не происходит.


3. `KafkaProducer` вызывает `event.toByteArray()` — `protobuf-java` сериализует объект `PatientEvent` в массив байтов `byte[]`.


4. `KafkaProducer` вызывает `kafkaTemplate.send("patient", event.toByteArray())` — `KafkaTemplate` публикует сообщение в топик `patient`. Ключ сообщения не задан `(null)`.


5. `KafkaTemplate` передаёт сообщение брокеру Kafka по протоколу TCP. Брокер возвращает `RecordMetadata (Ack)`. Метод `sendEvent()` завершается.


6. Управление возвращается в `PatientService`.

### Обработка исключительных ситуаций

#### Kafka недоступен или возник таймаут

1. `kafkaTemplate.send()` выбрасывает исключение, которое перехватывается блоком `catch(Exception e)` внутри `KafkaProducer`.


2. Ошибка записывается в лог уровня `ERROR`. В `{}` подставляется `event.toString()` — Protobuf однострочный текстовый формат:


    ERROR com.pm.patientservice.kafka.KafkaProducer - Error sending PatientCreated event: patientId: "550e8400-e29b-41d4-a716-446655440000" name: "John Doe" email: "john.doe@example.com" event_type: "PATIENT_CREATED"


3. Метод `sendEvent()` завершается нормально (исключение поглощено). `PatientService` продолжает выполнение и возвращает клиенту `200 OK`. Данные в `patient_db` сохранены, gRPC-вызов выполнен. Сообщение в Kafka считается потерянным (Data Inconsistency).

![PatientEventProducer.svg](..%2FDiagrams%2FPatientEventProducer.svg)

---
## Логирование

| Шаг в алгоритме | Уровень | Класс | Сообщение                                   |
|:----------------|:--------|:------|:--------------------------------------------|
| Исключительная ситуация                | ERROR        | KafkaProducer      |Error sending PatientCreated event: {event}|