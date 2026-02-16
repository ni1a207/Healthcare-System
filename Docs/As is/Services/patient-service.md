# patient-service

## Назначение
Сервис управления данными пациентов. Реализует CRUD-операции, синхронное создание платежного аккаунта в Billing Service через gRPC и асинхронную публикацию событий создания пациента в Kafka.
## Архитектурная схема (C4 Component)

![PatientServiceC4.svg](..%2FDiagrams%2FPatientServiceC4.svg)
## Стек технологий

* **Runtime**: Java 21 (Eclipse Temurin / OpenJDK).
* **Framework**: Spring Boot 3.4.0. (Spring Data JPA, Spring Web).
* **Database**: PostgreSQL 17 (runtime), H2 (In-memory/Test).
* **Messaging**: Spring Kafka 3.3.0. (Protobuf serialization).
* **Protocols**: REST (HTTP), gRPC/Kafka (Protobuf 4.29.1).
* **Build Tool**: Maven 3.9.9.
* **Infrastructure**: Docker / AWS CDK (Fargate).
* **Logging**: SLF4J (Logback).
  
 *В коде присутствует закомментированная конфигурация для H2 (In-memory), которая может быть активирована для локальных тестов.*

## Безопасность и контроль доступа

Доступ ко всем эндпоинтам осуществляется через API Gateway после валидации JWT в Auth Service.

## База данных: patient_db
### Таблица: patient

## API  
### REST 
| Метод и путь                                                 | Описание                                           |
|:-------------------------------------------------------------|:---------------------------------------------------| 
| [POST /patients](..%2FAPI%2FPOST.patients.md)                | Создание нового  пациента в системе                | 
| [GET /patients](..%2FAPI%2FGET.patients.md)                  | Получение списка всех зарегистрированных пациентов | 
| [PUT /patients/{id}](..%2FAPI%2FPUT.patients.id.md)          | Изменение данных пациента в системе                | 
| [DELETE /patients/{id}](..%2FAPI%2FDELETE.patients.id.md) | Удаление пациента из системы                       |

### gRPC
Интеграция с `billing-service`. Синхронное создание финансового счета пациента через адаптер BillingServiceGrpcClient.


| Метод | Путь (Service/Method) | Протокол | Формат | Описание |
| :--- | :--- | :--- | :--- | :--- |
| `CreateBillingAccount` | `BillingService/CreateBillingAccount` | gRPC (HTTP/2) | Protobuf | Создание биллингового аккаунта пациента |

### Kafka Producer
Сервис публикует события в брокер сообщений для уведомления других компонентов системы.

| Топик | Событие (eventType) | Протокол | Формат | Ключ (Key) | Описание |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `patient` | `PATIENT_CREATED` | Kafka TCP | Protobuf | `null` | Нотификация о создании нового пациента |

## Переменные окружения
### Application (patient-service)
| Переменная | Значение | Описание |
| :--- | :--- | :--- |
| **SERVER_PORT** | 4000 | Порт сервиса |
| **SPRING_DATASOURCE_URL** | jdbc:postgresql://patient-service-db:5432/db <br> (Infra: jdbc:postgresql://patient-service-db:5432/patient-service-db) | URL для подключения к PostgreSQL |
| **SPRING_DATASOURCE_USERNAME** | admin_user | Имя пользователя базы данных |
| **SPRING_DATASOURCE_PASSWORD** | password | Пароль пользователя базы данных |
| **SPRING_KAFKA_BOOTSTRAP_SERVERS** | kafka:9092 <br> (Infra: localhost.localstack.cloud:4510, 4511, 4512) | Адрес брокера Kafka для Producer |
| **BILLING_SERVICE_ADDRESS** | billing-service <br> (Infra: host.docker.internal) | Хост gRPC-Billing Service |
| **BILLING_SERVICE_GRPC_PORT** | 9005 <br> (Infra: 9001) | Порт gRPC-сервиса биллинга |
| **SPRING_JPA_HIBERNATE_DDL_AUTO** | update | Автоматическое обновление схемы БД |
| **SPRING_SQL_INIT_MODE** | always | Режим выполнения SQL-скриптов (data.sql) |
| **JAVA_TOOL_OPTIONS** | -agentlib:jdwp=...address=*:5005 | Параметры для удаленной отладки (Debug) |

### Database (patient-service-db)
| Переменная | Значение | Описание |
| :--- | :--- | :--- |
| **POSTGRES_DB** | db <br> (Infra: patient-service-db) | Название создаваемой схемы |
| **POSTGRES_USER** | admin_user | Логин администратора БД |
| **POSTGRES_PASSWORD** | password | Пароль администратора БД |

## Сетевые параметры

| Параметр | Значение | Описание |
| :--- | :--- | :--- |
| **Service Name** | patient-service | Имя хоста в Docker-сети |
| **Internal Port** | 4000 | Входящий HTTP порт (server.port) |
| **Debug Port** | 5005 | Порт для подключения Java Debugger |
| **Target gRPC Host** | billing-service:9005 <br> (Infra: host.docker.internal:9001) | Адрес назначения для gRPC вызовов |
| **Target DB Host** | patient-service-db:5432 | Адрес подключения к PostgreSQL |
| **Target Kafka Host** | kafka:9092 <br> (Infra: localhost.localstack.cloud:4510, 4511, 4512) | Адрес подключения к брокеру Kafka |
## Swagger
| Интерфейс | URL              | Описание |
| :--- |:-----------------| :--- |
| **Swagger UI** | /swagger-ui.html | Локальный UI документации |
| **OpenAPI JSON** | /v3/api-docs     | Спецификация API в формате JSON |