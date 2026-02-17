# billing-service

## Назначение
Сервис управления финансовыми счетами пациентов. Отвечает за создание и обслуживание биллинговых аккаунтов. В текущей реализации обрабатывает запросы на регистрацию счетов, поступающие от patient-service по протоколу gRPC.

## Архитектурная схема (C4 Component)

![billing-serviceC4.svg](..%2FDiagrams%2Fbilling-serviceC4.svg)

## Стек технологий
* **Runtime**: Java 21 (Eclipse Temurin / OpenJDK)).
* **Framework**: Spring Boot 3.4.1.
* **Protocols**: gRPC (v1.69.0), Protobuf (v4.29.1).
* **Library**: grpc-spring-boot-starter (v3.1.0.RELEASE).
* **Build Tool**: Maven 3.9.9.
* **Infrastructure**: Docker (Multi-stage build).
* **Logging**: SLF4J (Logback).

## Безопасность и контроль доступа
В текущей конфигурации механизмы авторизации и аутентификации (Spring Security) не подключены. Доступ к gRPC-методам открыт для внутренних компонентов системы (api-gateway не имеет прямого доступа к данному сервису).

## База данных
В текущей версии кода (BillingGrpcService.java) персистентный слой не реализован. Бизнес-логика по сохранению данных в БД имитируется: сервис возвращает статические значения {accountId: "12345", status: "ACTIVE"}.

## API
### gRPC

| Метод                                                      | Тип запроса    | Тип ответа      | Описание |
|:-----------------------------------------------------------|:---------------|:----------------| :--- |
| [CreateBillingAccount](..%2FAPI%2FCreateBillingAccount.md) | BillingRequest | BillingResponse | Создание нового биллингового счета для пациента |


## Переменные окружения

| Переменная | Значение (по умолчанию) | Описание |
| :--- | :--- | :--- |
| **SERVER_PORT** | 4001 | Порт для HTTP (Actuator/Health) |
| **GRPC_SERVER_PORT** | 9001 | Порт для gRPC взаимодействия |

## Сетевые параметры

| Параметр | Значение | Описание |
| :--- | :--- | :--- |
| **Service Name** | billing-service | Имя хоста для внутреннего взаимодействия |
| **Internal HTTP Port** | 4001 | Служебный порт (Spring Web) |
| **Internal gRPC Port** | 9001 | Основной порт приема трафика |

