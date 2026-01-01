# Patient Service

## Назначение
Центральный микросервис системы, отвечающий за управление жизненным циклом данных пациентов (CRUD) и оркестрацию связанных бизнес-процессов: создание финансовых аккаунтов через gRPC и уведомление смежных систем через Kafka.
## Архитектурная схема (C4 Component)

![PatientServiceC4.svg](..%2FDiagrams%2FPatientServiceC4.svg)
*Примечание: На схеме отображены только прямые интеграции сервиса.*
## Стек технологий

* **Runtime**: Java 21 (Eclipse Temurin / OpenJDK).
* **Framework**: Spring Boot 3.4.0. (Spring Data JPA, Spring Web).
* **Database**: PostgreSQL (runtime), H2 (In-memory/Test).
* **Messaging**: Spring Kafka 3.3.0.
* **Protocols**: REST (HTTP), gRPC (Client).
* **Serialization**: Protobuf 4.29.1 (для gRPC и Kafka).
* **Build Tool**: Maven 3.9.9.
* **Infrastructure**: Docker.
* **Logging**: SLF4J/Logback (Level: INFO).
  
 *В коде присутствует закомментированная конфигурация для H2 (In-memory), которая может быть активирована для локальных тестов.*

## Безопасность и контроль доступа

Доступ ко всем эндпоинтам осуществляется через API Gateway после валидации JWT в Auth Service.

## База данных: patient_db
### Таблица: patients

| Название | Обяз. | Тип       | Ограничения | Описание                               | Пример |
| :--- | :--- |:----------| :--- |:---------------------------------------| :--- |
| **id** | Да | UUID      | Primary Key | Уникальный идентификатор записи в БД   | 6ff9700a-040a-4119-a7a9-f6662122e423 |
| **name** | Да | String    | Not Null, Max 100 chars | ФИО пациента                           | Иван Иванов |
| **email** | Да | String    | Unique, Not Null | Электронная почта                      | ivan@mail.com |
| **address** | Да | String    | Not Null | Адрес  проживания                      | г. Москва, ул. Ленина, д. 1 |
| **date_of_birth** | Да | LocalDate | Not Null | Дата рождения                          | 1990-05-15 |
| **registered_date** | Да | LocalDate | Not Null | Дата регистрации (только при создании) | 2025-12-20 |
## API  
### REST 
| Метод и путь                                      | Описание                                           |
|:--------------------------------------------------|:---------------------------------------------------| 
| [POST /patients](..%2FAPI%2FPOST.api.patients.md) | Создание нового  пациента в системе                | 
| GET /patients                                     | Получение списка всех зарегистрированных пациентов | 
| PUT                  /patients/{id}               | Изменение данных пациента в системе                | 
| DELETE               /patients/{id}               | Удаление пациента из системы                       | 


### gRPC  
* **Service**: BillingService
* **Method**: createBillingAccount 

 
**Синхронное создание финансового счета пациента через адаптер BillingServiceGrpcClient.*
### Kafka
* **Topic**: patient
* **Event**: PatientEvent (Protobuf)
* **Role**: Producer
* **Event Type**: PATIENT_CREATED
* **Key-serializer**: StringSerializer 
* **Value-serializer**: ByteArraySerializer
## Переменные окружения
### Application (patient-service)
| Переменная | Значение                                     | Описание                                    |
| :--- |:---------------------------------------------|:--------------------------------------------|
| **SERVER_PORT** | 4000                                         | Порт сервиса                                |
| **SPRING_DATASOURCE_URL** | jdbc:postgresql://patient-service-db:5432/db | URL для подключения к PostgreSQL            |
| **SPRING_DATASOURCE_USERNAME** | admin_user                                   | Имя пользователя базы данных                |
| **SPRING_DATASOURCE_PASSWORD** | password                                     | Пароль пользователя базы данных             |
| **SPRING_KAFKA_BOOTSTRAP_SERVERS**| kafka:9092                                   | Адрес брокера Kafka для Producer            |
| **BILLING_SERVICE_ADDRESS** | billing-service                              | Хост gRPC-Billing Service                   |
| **BILLING_SERVICE_GRPC_PORT** | 9001                                         | Порт gRPC-сервиса биллинга                  |
| **SPRING_JPA_HIBERNATE_DDL_AUTO** | update                                       | Автоматическое обновление схемы базы данных |
| **JAVA_TOOL_OPTIONS** | -agentlib:jdwp=...address=*:5005             | Параметры для удаленной отладки (Debug)     |
### Database (patient-service-db)
| Переменная | Значение   | Описание |
| :--- |:-----------| :--- |
| **POSTGRES_DB** | db         | Название создаваемой схемы |
| **POSTGRES_USER** | admin_user | Логин администратора БД |
| **POSTGRES_PASSWORD** | password   | Пароль администратора БД |

## Сетевые параметры 

| Параметр | Значение                | Описание |
| :--- |:------------------------| :--- |
| **Service Name** | patient-service         | Имя хоста в Docker-сети |
| **Internal Port** | 4000                    | Входящий HTTP порт (server.port) |
| **Debug Port** | 5005                    | Порт для подключения Java Debugger |
| **Target gRPC Host** | billing-service:9001    | Адрес назначения для gRPC вызовов |
| **Target DB Host** | patient-service-db:5432 | Адрес подключения к PostgreSQL |
| **Target Kafka Host** | kafka:9092              | Адрес подключения к брокеру Kafka |

## Swagger
| Интерфейс | URL              | Описание |
| :--- |:-----------------| :--- |
| **Swagger UI** | /swagger-ui.html | Локальный UI документации |
| **OpenAPI JSON** | /v3/api-docs     | Спецификация API в формате JSON |