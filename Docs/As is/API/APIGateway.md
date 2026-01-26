## Алгоритм работы при запросе к защищенным ресурсам системы

1. `api-gateway` получает запрос, через `Route Configuration` анализирует URL запроса: ищет совпадение в `application.yml`. Если для найденного маршрута в списке `filters` указан `JwtValidation`.


2. Управление передается классу-обработчику `JwtValidationGatewayFilterFactory`.


3. `JwtValidationGatewayFilterFactory` извлекает из заголовка `Authorization` значение `Bearer <token>`. Если токена нет или он не начинается с `Bearer `  шлюз возвращает HTTP статус-код `401 UNAUTHORIZED`.


4. `api-gateway`, используя реактивный `WebClient`, инициирует проверку сессии: выполняется HTTP-запрос `GET /validate` в `auth-service` с заголовком `Authorization`.


5. `auth-service` проверяет токен и возвращает ответ:
   
   **5.1.** Если сервис не отвечает или произошла ошибка на сервере — `api-gateway` возвращает HTTP статус-код `500 INTERNAL SERVER ERROR`.
    
   **5.2.** Если токен не прошел проверку, сервис возвращает HTTP статус-код `401 UNAUTHORIZED`.
    
   **5.3.** Если токен прошел проверку, сервис возвращает HTTP статус-код `200 OK`.


6. `api-gateway` получает ответ `200 OK` и проксирует запрос к целевому сервису.


7. `api-gateway` получает ответ от целевого сервиса.


8. `api-gateway` проксирует ответ клиенту.
 
![APIGateway.validate.svg](..%2FDiagrams%2FAPIGateway.validate.svg)

## Алгоритм работы при запросе к публичным ресурсам системы

1. `api-gateway` принимает HTTP-запрос от `Single Page Application`. Компонент `Route Configuration` сопоставляет URI запроса с правилами в `application.yml`. Если для найденного маршрута в списке `filters` не указан `JwtValidation`, `JwtValidationGatewayFilterFactory` и `WebClient` (для вызова auth-service) в обработке не участвуют.
2. `api-gateway`  проксирует запрос к целевому сервису.


3. `api-gateway` получает ответ от целевого сервиса.


4. `api-gateway` проксирует ответ клиенту.

![APIGateway.routing.svg](..%2FDiagrams%2FAPIGateway.routing.svg)