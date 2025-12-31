# Auth Service

## Назначение
Сервис аутентификации и авторизации. Выполняет проверку учетных данных пользователей, генерацию подписанных JWT-токенов и верификацию токенов для внешних компонентов системы (api-gateway).

## Архитектурная схема (C4 Container)
![AuthServiceC4.svg](../Diagrams/AuthServiceC4.svg)

## Стек технологий
* **Runtime**: Java 21 (OpenJDK).
* **Framework**: Spring Boot 3.4.1 (Security, Data JPA, Validation).
* **Token Management**: JJWT 0.12.6.
* **Database**: PostgreSQL (Runtime), H2 (In-memory/Test).
* **Build Tool**: Maven 3.9.9.
* **Infrastructure**: Docker (Port 4005).
* **Documentation**: Springdoc OpenAPI 2.6.0.
* **Logging**: SLF4J.

## Безопасность и контроль доступа

**Шифрование паролей**: Используется BCrypt. Хранение осуществляется в виде хэшей.

**JWT**:
Алгоритм подписи: HMAC SHA (HS256). Ключ считывается из переменной ${jwt.secret}.

**Payload:**
* **sub**: Email пользователя.
* **role**: Роль пользователя (строка).
* **iat**: Время выпуска токена (Unix Timestamp).
* **exp**: Время истечения срока действия (10 часов с момента выдачи).

**Контроль доступа**: Сервис настроен на permitAll(). Ограничение доступа к эндпоинтам авторизации не требуется, так как проверка прав и валидация токена реализованы внутри бизнес-логики (AuthService).

### Ролевая модель
В сервисе реализована ролевая модель. Тип данных в JWT и БД — строка.

| Роль | Область доступа |
|:---|:---|
| **USER** | Базовый доступ. |
| **ADMIN** | Расширенный доступ (предусмотрен для тестового аккаунта в data.sql). |

### База данных: auth_db
#### Таблица: users

| Название | Обяз. | Тип | Ограничения | Описание | Пример |
|:---|:---|:---|:---|:---|:---|
| **id** | Да | UUID | PK | Идентификатор пользователя | 223e4567-e89b... |
| **email** | Да | VARCHAR(255) | Unique, Not Null | Логин пользователя (Email) | testuser@test.com |
| **password** | Да | VARCHAR(255) | Not Null | Хэш пароля (BCrypt) | $2b$12$7hoR... |
| **role** | Да | VARCHAR(50) | Not Null | Роль пользователя | ADMIN |

## API
### REST

| Метод и путь                                      | Описание |
|:--------------------------------------------------|:-|
| [POST /login ](..%2FAPI%2FPOST.api.auth.login.md) | Аутентификация пользователя. Генерация JWT. |
| [GET /validate](..%2FAPI%2FGET.validate.md)       | Валидация JWT при внешних запросах к системе. |

## Переменные окружения
### Application (auth-service)
Параметры считываются из application.properties. Для работы с PostgreSQL и JWT необходима передача следующих переменных через окружение (Docker):

| Переменная | Значение (по умолчанию) | Описание |
| :--- |:---|:---|
| **SERVER_PORT** | 4005 | Порт сервиса |
| **SPRING_DATASOURCE_URL** | jdbc:postgresql://auth-service-db:5432/db | URL для подключения к PostgreSQL |
| **SPRING_DATASOURCE_USERNAME** | admin_user | Имя пользователя базы данных |
| **SPRING_DATASOURCE_PASSWORD** | password | Пароль пользователя базы данных |
| **SPRING_JPA_HIBERNATE_DDL_AUTO** | update | Автоматическое обновление схемы базы данных |
| **SPRING_SQL_INIT_MODE** | always | Выполнение data.sql при каждом запуске |
| **JWT_SECRET** | - | Секретный ключ для подписи токенов (Base64) |

### Database (auth-service-db)
| Переменная | Значение | Описание |
| :--- |:---|:---|
| **POSTGRES_DB** | db | Название создаваемой схемы |
| **POSTGRES_USER** | admin_user | Корневой пользователь базы данных |
| **POSTGRES_PASSWORD** | password | Пароль корневого пользователя |

## Сетевые параметры

| Параметр | Значение | Описание |
| :--- |:---|:---|
| **Service Name** | auth-service | Имя хоста в Docker-сети |
| **Internal Port** | 4005 | Входящий HTTP порт (server.port) |
| **Target DB Host** | auth-service-db:5432 | Адрес подключения к PostgreSQL |

## Swagger
Интерфейс доступен напрямую при обращении к сервису:

| Интерфейс | URL | Описание |
| :--- |:---|:---|
| **Swagger UI** | /swagger-ui.html | UI документации |
| **OpenAPI JSON** | /v3/api-docs | Спецификация API в формате JSON |