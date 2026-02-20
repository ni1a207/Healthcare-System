# Kafka Point (Consumer) - PatientEvent

## Назначение

Событие потребляется сервисом analytics-service. Метод предназначен для получения данных о создании новых пациентов из Kafka для их последующей регистрации в системе аналитики. Текущая реализация выполняет роль логирующей заглушки.

## Общие сведения

**Consumer:** analytics-service

**Topic:** patient

**Protocol:** TCP

**Format message:** Protobuf

**Group ID:** analytics-service

## Структура сообщения (Схема Protobuf)

Параметры файла `PatientEvent.proto`:

| Параметр | Тип данных | Обязательность | Описание | Значение/пример| Маппинг с db|
| --- | --- | --- | --- |-|-|
| patientId | string | Да | Уникальный идентификатор пациента (UUID) |550e8400-e29b-41d4-a716-446655440000|-|
| name | string | Да | ФИО пациента |John Doe|-|
| email | string | Да | Электронная почта пациента |john.doe@example.com|-|
| event_type | string | Да | Тип события |PATIENT_CREATED|-|

## Алгоритм обработки события

Обработка происходит в классе `KafkaConsumer` при поступлении сообщения в топик `patient`.

1. Триггер: Получение массива байтов (`byte[]`) из Kafka.


2. Десериализация: Вызов метода `PatientEvent.parseFrom(event)`, библиотека `protobuf-java` преобразует бинарные данные в Java-объект.


3. Извлечение данных: Из объекта `patientEvent` извлекаются поля `patientId`, `name` и `email`.
 
 
4. Логирование: С помощью `SLF4J` выполняется запись в лог уровня INFO: 


5. Фиксация смещения (Ack): После завершения метода consumeEvent (независимо от успеха десериализации, так как исключение поймано), сервис автоматически подтверждает обработку сообщения (Auto-commit), сдвигая offset в Kafka.
   

    INFO [имя_потока] com.pm.analyticsservice.kafka.KafkaConsumer - Received Patient Event: [PatientId=550e8400-e29b-41d4-a716-446655440000, PatientName=John Doe, PatientEmail=john.doe@example.com]

### Обработка исключительных ситуаций

**В топик поступило сообщение, не соответствующее схеме `PatientEvent` / поврежденные данные**

1. Выбрасывается исключение `InvalidProtocolBufferException`. Ошибка записывается в лог приложения: `Error deserializing event {message}`.


    ERROR [имя_потока] c.p.a.k.KafkaConsumer - Error deserializing event
    [Stacktrace: com.google.protobuf.InvalidProtocolBufferException: <Причина ошибки>]


2. Сообщение считается обработанным с ошибкой, выполнение метода завершается.



> **Важно**: Текущая реализация не сохраняет данные в базу данных и не выполняет агрегацию метрик. Ключ сообщения (Kafka Key) сервисом не обрабатывается.

![PatientEventConsumer.svg](..%2FDiagrams%2FPatientEventConsumer.svg)