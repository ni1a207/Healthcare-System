## Алгоритм работы при запросе к защищенным ресурсам системы

1. `api-gateway` принимает HTTP-запрос на порту 4004. Компонент `Route Configuration` сопоставляет URI запроса с правилами маршрутизации в `application.yml`. Для маршрута `/api/patients/**` в списке `filters` указан `JwtValidation`.


2. `Route Configuration` передаёт управление фильтру `JwtValidationGatewayFilterFactory`.


3. `JwtValidationGatewayFilterFactory` извлекает значение заголовка `Authorization` из входящего запроса. Если заголовок отсутствует или не начинается с префикса `Bearer ` — `api-gateway` немедленно возвращает клиенту HTTP статус-код `401 UNAUTHORIZED`, запрос не проксируется.


4. `JwtValidationGatewayFilterFactory` через реактивный `WebClient` инициирует HTTP-запрос [GET /validate](GET.validate.md) к `auth-service` с заголовком `Authorization: Bearer <token>`.


5. `auth-service` выполняет валидацию токена и возвращает ответ. Если `auth-service` недоступен или вернул любой не-2xx статус — `WebClient` выбрасывает `WebClientResponseException`, которое не перехватывается — `api-gateway` возвращает клиенту HTTP статус-код `500 INTERNAL SERVER ERROR`.


>**Важно:** В текущей реализации JwtValidationGatewayFilterFactory отсутствует явная обработка 401 от auth-service через .onStatus(). Если токен невалиден и auth-service возвращает 401 — клиент получит 500, а не 401.


6. При получении `200 OK` от `auth-service` — `WebClient` передаёт управление следующему фильтру в цепочке `chain.filter(exchange)`. `api-gateway` проксирует запрос к целевому сервису. Префикс `/api` удаляется через `StripPrefix=1` — запрос `/api/patients/123` преобразуется в `/patients/123`.


7. `api-gateway` получает ответ от целевого сервиса. Если сервис недоступен — возвращается HTTP статус-код `500 INTERNAL SERVER ERROR`.


8. `api-gateway` проксирует ответ клиенту.
 
![APIGateway.validate.svg](..%2FDiagrams%2FAPIGateway.validate.svg)

## Логирование

| Шаг в алгоритме       | Уровень | Класс | Сообщение                                   |
|:----------------------|:--------|:----|:--------------------------------------------|
|-|-|-| Логирование в данном методе не предусмотрено|


---


## Алгоритм работы при запросе к публичным ресурсам системы

1. `api-gateway` принимает HTTP-запрос на порту 4004. Компонент `Route Configuration` сопоставляет URI запроса с правилами в `application.yml`. Для маршрута `/auth/**` фильтр `JwtValidation` не указан — `JwtValidationGatewayFilterFactory` и `WebClient` в обработке не участвуют. Префикс `/auth`удаляется через `StripPrefix=1`.


2. `api-gateway` проксирует запрос к целевому сервису.


3. `api-gateway` получает ответ от целевого сервиса. Если сервис недоступен — возвращается HTTP статус-код `500 INTERNAL SERVER ERROR`.


4. `api-gateway` проксирует ответ клиенту.

![APIGateway.routing.svg](..%2FDiagrams%2FAPIGateway.routing.svg)


## Логирование

| Шаг в алгоритме       | Уровень | Класс | Сообщение                                   |
|:----------------------|:--------|:----|:--------------------------------------------|
|-|-|-| Логирование в данном методе не предусмотрено|