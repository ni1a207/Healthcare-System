# API Gateway Service

## Назначение
Единая точка входа для всей микросервисной архитектуры. Обеспечивает маршрутизацию запросов, централизованную проверку безопасности (JWT через внешний сервис) и проксирование документации OpenAPI.
## Архитектурная схема (C4 Container)
![APIGatewayC4.svg](..%2FDiagrams%2FAPIGatewayC4.svg)

*Примечание: Шлюз является посредником между внешними клиентами и внутренними микросервисами.*

## Стек технологий
* **Runtime**: Java 21 (OpenJDK).
* **Framework**: Spring Boot 3.4.1 (Spring Cloud Gateway).
* **Build Tool**: Maven 3.9.9.
* **Infrastructure**: Docker.
* **Logging**: SLF4J.

## Безопасность и контроль доступа
Шлюз делегирует проверку подлинности специализированному сервису авторизации: кастомный фильтр JwtValidation (класс JwtValidationGatewayFilterFactory) извлекает токен из заголовка Authorization и выполняет проверочный запрос GET /validate к auth-service.
При получении ответа 200 OK шлюз пропускает запрос дальше.
## Переменные окружения

| Переменная | Значение                    | Описание                                             |
| :--- |:----------------------------|:-----------------------------------------------------|
| **SERVER_PORT** | 4004                        | Порт шлюза                                           |
| **AUTH_SERVICE_URL** | http://auth-service:4005    | Адрес для перенаправления запросов авторизации       |
| **PATIENT_SERVICE_URL** | http://patient-service:4000 | Адрес для перенаправления запросов сервиса пациентов |

## Сетевые параметры
| Параметр | Значение    | Описание |
| :--- |:------------| :--- |
| **Service Name** | api-gateway | Имя хоста в Docker-сети |
| **Internal Port** | 4004        | Входящий HTTP порт (server.port) |
| **Exposed Port** | 4004        | Порт, доступный извне (localhost)
## Маршрутизация
Шлюз использует статическую маршрутизацию. Имена хостов должны быть разрешимы на уровне сетевой инфраструктуры (Docker DNS).

| Направление         | Префикс пути       | Целевой URI                      | Фильтры                                      |
|:--------------------|:-------------------|:---------------------------------|:---------------------------------------------|
| **Auth Service**    | /api/auth/**       | http://host.docker.internal:4005 | StripPrefix=1                                |
| **Patient Service** | /api/patient/**    | http://host.docker.internal:4000 | StripPrefix=1, JwtValidation                 |
| **Auth Docs**       | /api-docs/auth     |http://host.docker.internal:4005 | RewritePath=/api-docs/auth, /v3/api-docs     |
| **Patient Docs**    | /api-docs/patients | http://host.docker.internal:4000 | RewritePath=/api-docs/patients, /v3/api-docs |


### Особенности конфигурации:
1. **StripPrefix=1**: Удаляет первый сегмент пути перед отправкой запроса конечному микросервису.
2. **JwtValidation**: Применяется только к бизнес-логике (/api/patients/**). Запросы на авторизацию и получение документации проходят без этого фильтра.