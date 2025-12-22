# Analytics Service

## Назначение
Сервис-потребитель событий (Consumer). Предназначен для извлечения данных о пациентах из Kafka и их первичной обработки (десериализации). Текущая версия служит заглушкой для будущего модуля аналитики.
## Архитектурная схема (C4 Container)
![AnalyticsServiceC4.svg](..%2FDiagrams%2FAnalyticsServiceC4.svg)
## Стек технологий

* **Runtime**: Java 21 (OpenJDK).
* **Framework**: Spring Boot 3.4.1.
* **Messaging**: Spring Kafka 3.3.0.
* **Serialization**: Protobuf-java 4.29.1.
* **Build Tool**: Maven 3.9.9.
* **Infrastructure**: Docker.
* **Logging**: SLF4J.

## Безопасность и контроль доступа
Внешние API отсутствуют. Безопасность обеспечивается на уровне сетевой изоляции внутри Docker-сети.
## База данных
Отсутствует. Вся обработка происходит в оперативной памяти (In-memory).
## API
### Kafka
* **Role**: Consumer

* **Topic**: patient
* **Event**: PatientEvent (Protobuf)
* **Group ID**: analytics-service

## Переменные окружения
| Переменная              | Значение   | Описание                    |
|:------------------------|:-----------|:----------------------------|
| **SERVER_PORT** | 4002       |Порт сервиса
|SPRING_KAFKA_BOOTSTRAP_SERVERS| kafka:9092 | Адрес брокера Kafka для Consumer

| Параметр | Значение          | Описание |
| :--- |:------------------| :--- |
| **Service Name** | analytics-service | Имя хоста в Docker-сети |
| **Internal Port** | 4002              | Входящий HTTP порт (server.port) |
| **Target Kafka Host** | kafka:9092        | Адрес подключения к брокеру Kafka |