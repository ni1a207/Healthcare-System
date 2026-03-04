# Kafka Point (Consumer) - PatientEvent

## Назначение

Событие потребляется сервисом `analytics-service`. Метод предназначен для получения данных о создании новых пациентов из Kafka для их последующей регистрации в системе аналитики. Текущая реализация выполняет роль логирующей заглушки.

---

## Общие сведения

**Consumer:** `analytics-service`

**Topic:** `patient`

**Protocol:** `TCP`

**Format message:** `Protobuf`

**Group ID:** `analytics-service`

---

## Структура сообщения (Схема Protobuf)

Параметры файла `PatientEvent.proto`:

| Параметр | Тип данных | Обязательность | Описание | Значение/пример| Маппинг с db|
| --- | --- | --- | --- |-|-|
| patientId | string | Да | Уникальный идентификатор пациента (UUID) |550e8400-e29b-41d4-a716-446655440000|-|
| name | string | Да | ФИО пациента |John Doe|-|
| email | string | Да | Электронная почта пациента |john.doe@example.com|-|
| event_type | string | Да | Тип события |PATIENT_CREATED|-|

---

## Алгоритм обработки события

Обработка происходит в классе `KafkaConsumer` при поступлении сообщения в топик `patient`.

1. Триггер: брокер Kafka доставляет сообщение из топика `patient` в `analytics-service`. Spring Kafka вызывает метод `consumeEvent(byte[] event)` аннотированный `@KafkaListener(topics="patient", groupId="analytics-service")`.


2. `protobuf-java` десериализует полученный массив байтов `byte[]` в Java-объект `PatientEvent` через `PatientEvent.parseFrom(event)`.


3. Из объекта `patientEvent` извлекаются поля `patientId`, `name`, `email`.


4. С помощью SLF4J выполняется запись в лог уровня `INFO`:


    INFO com.pm.analyticsservice.kafka.KafkaConsumer - Received Patient Event: [PatientId=550e8400-e29b-41d4-a716-446655440000,PatientName=John Doe,PatientEmail=john.doe@example.com]

5. Метод `consumeEvent()` завершается. Spring Kafka выполняет авто-коммит offset — сообщение считается обработанным.


### Обработка исключительных ситуаций

#### **Сообщение не соответствует схеме `PatientEvent` / поврежденные данные / ошибка десериализации**

1. `PatientEvent.parseFrom(event)` выбрасывает `InvalidProtocolBufferException`, которое перехватывается блоком catch.


2. Ошибка записывается в лог уровня `ERROR`. В `{}` подставляется `e.getMessage()` — только текст ошибки без `stacktrace`:


    ERROR com.pm.analyticsservice.kafka.KafkaConsumer - Error deserializing event <сообщение об ошибке>


3. Метод `consumeEvent()` завершается нормально. Spring Kafka выполняет авто-коммит offset — сообщение считается обработанным и безвозвратно теряется (DLQ не настроен).

> **Важно**: Текущая реализация не сохраняет данные в базу данных и не выполняет агрегацию метрик. Ключ сообщения (Kafka Key) сервисом не обрабатывается.

![PatientEventConsumer.svg](..%2FDiagrams%2FPatientEventConsumer.svg)

---
## Логирование

| Шаг в алгоритме | Уровень | Класс | Сообщение                                   |
|:----------------|:--------|:------|:--------------------------------------------|
| 4               |INFO| KafkaConsumer      | Received Patient Event: [PatientId={},PatientName={},PatientEmail={}]|
| Исключительная ситуация                |ERROR| KafkaConsumer      | Error deserializing event {e.getMessage()}|