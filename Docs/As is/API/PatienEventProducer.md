# Kafka Point (Producer) - PatientEvent

## Назначение

Событие публикуется сервисом patient-service при успешном создании новой карточки пациента в базе данных и после создания биллингового аккаунта через gRPC. Событие уведомляет заинтересованные сервисы о появлении нового пользователя.
## Общие сведения
**Producer:** patient-service

**Topic:** patient

**Protocol:** TCP

**Format message:** Protobuf 

## Структура сообщения (Схема Protobuf)
Параметры файла `PatientEvent.proto`:

| Параметр | Тип данных | Обязательность | Описание | Значение/пример| Маппинг с patient_db |
| --- | --- | --- | --- |-|-----------------|
| patientId | string | Да | Уникальный идентификатор пациента (UUID) |550e8400-e29b-41d4-a716-446655440000| patient.id                |
| name | string | Да | ФИО пациента |John Doe| patient.name                |
| email | string | Да | Электронная почта пациента |john.doe@example.com| patient.email              |
| event_type | string | Да | Тип события |PATIENT_CREATED| Hardcoded value                |



## Пример сообщения (JSON до сериализации)
~~~ 

{
"patientId": "550e8400-e29b-41d4-a716-446655440000",
"name": "John Doe",
"email": "john.doe@example.com",
"event_type": "PATIENT_CREATED"
}   
~~~ 


## Алгоритм публикации события
Публикация происходит в процессе выполнения метода [POST /patients](POST.patients.md) .

1. Триггер: Успешное завершение бизнес-логики (создание записи в `patient_db` в `patient-service` и успешный gRPC вызов к `billing-service`).


2. Формирование данных: Сервис извлекает данные сущности из БД `patient_db`.


3. Сборка сообщения: Заполнение параметров `PatientEvent`, установка типа события `PATIENT_CREATED` в параметр `event_type`.


4. Сериализация: Трансформация объекта в массив байтов (ByteArraySerializer).


5. Отправка: Публикация в топик `patient`.

### Обработка исключительных ситуаций

#### Kafka недоступен или возник таймаут

1. Ошибка записывается в лог приложения:


    ERROR [имя_потока] com.pm.patientservice.KafkaProducer - Error sending PatientCreated event: patientId: "550e8400-e29b-41d4-a716-446655440000"
    name: "John Doe"
    email: "john.doe@example.com"
    event_type: "PATIENT_CREATED"

2. Клиент получает успешный ответ 200 OK, данные в БД сохранены, gRPC вызов выполнен. **Сообщение в Kafka считается потерянным (Data Inconsistency)**.

![PatientEventProducer.svg](..%2FDiagrams%2FPatientEventProducer.svg)