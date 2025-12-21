# Patient Service

## Назначение
Центральный микросервис системы, отвечающий за управление жизненным циклом данных пациентов (CRUD) и оркестрацию связанных бизнес-процессов: создание финансовых аккаунтов через gRPC и уведомление смежных систем через Kafka.
## Архитектурная схема (C4 Container)
![PatientServiceC4.svg](..%2FDiagrams%2FPatientServiceC4.svg)
*Примечание: На схеме отображены только прямые интеграции сервиса.*
## Стек технологий
Backend: Java.

Framework: Spring Boot 3.

Data base: PostgreSQL (Spring Data JPA).

Message broker: Kafka 

Service Discovery: Eureka

## Безопасность и контроль доступа

Доступ ко всем эндпоинтам осуществляется через API Gateway после валидации JWT в Auth Service.

## База данных `patient_db`
### Таблица: `patients`

| Название | Обяз. | Тип | Ограничения | Описание | Пример |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **id** | Да | Long | Primary Key, Auto-increment | Уникальный идентификатор записи в БД | 1 |
| **name** | Да | String | Not Null, Max 100 chars | ФИО пациента (соответствует полю в DTO и Entity) | Иван Иванов |
| **email** | Да | String | Unique, Not Null | Электронная почта (используется для проверки уникальности) | ivan@mail.com |
| **address** | Да | String | Not Null | Адрес фактического проживания | г. Москва, ул. Ленина, д. 1 |
| **date_of_birth** | Да | LocalDate | Not Null | Дата рождения (мапится из String через `LocalDate.parse`) | 1990-05-15 |
| **registered_date** | Да | LocalDate | Not Null | Дата регистрации (заполняется только при создании записи) | 2025-12-20 |а рождения пациента | `1990-05-15` |
## API Интерфейсы (OpenAPI)
### REST Эндпоинты
| Метод    | Путь                    | Описание                                           |
|:---------|:------------------------|:---------------------------------------------------| 
| `POST`   | `/api/v1/patients`      | Создание нового  пациента в системе                | 
| `GET`    | `/api/v1/patients`      | Получение списка всех зарегистрированных пациентов | 
| `PUT`    | `/api/v1/patients/{id}` | Изменение данных пациента в системе                | 
| `DELETE` | `/api/v1/patients/{id}` | Удаление пациента из системы                       | 


### gRPC Интерфейсы 
* **Service**: `BillingService`
* **Method**: `createBillingAccount` 
* **Описание**: Синхронное создание финансового счета пациента.

### Событийная модель (Kafka)
* **Topic**: `patient-events`
* **Role**: `Producer`
* **Event** Type: `PATIENT_CREATED`

## Переменные окружения
| Переменная              | Описание                       |
|:------------------------|:-------------------------------|
| SPRING_DATASOURCE_URL   | Адрес подключения к patient_db |
| KAFKA_BOOTSTRAP_SERVERS | Адрес брокера сообщений        |
|BILLING_SERVICE_GRPC_HOST|Хост для связи с Billing Service|

## Сетевые параметры и Discovery
Микросервис регистрируется в реестре сервисов Eureka и доступен внутри сети по следующим параметрам:

| Параметр                | Значение по умолчанию | Описание                                      |
|:------------------------|:----------------------|:----------------------------------------------|
| **Service Name** | `patient-service`     | Идентификатор для Service Discovery (Eureka)  |
| **HTTP Port** | `8081`                | Внутренний порт сервиса                       |
| **Internal Base URL** | `http://patient-service:8081` | URL для межсервисного взаимодействия в Docker |
| **External URL (через GW)**| `http://localhost:8080/api/v1/patients` | Точка входа для Frontend/Postman |

### Реестр сервисов (Service Discovery)
* **Eureka Client**: Сервис автоматически регистрируется в Eureka Server при запуске.
* **Балансировка**: API Gateway использует имя `patient-service` для балансировки запросов между инстансами.