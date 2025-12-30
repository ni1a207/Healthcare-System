## Алгоритм работы при запросе к защищенным ресурсам системы

1. `api-gateway` получает запрос, анализирует URL запроса: ищет совпадение в `application.yml`. Если для найденного маршрута в списке `filters` указан - `JwtValidation`, управление передается классу-обработчику `JwtValidationGatewayFilterFactory`.


2. `api-gateway` (в рамках логики фильтра) извлекает из заголовка `Authorization` `Bearer <token>`, если токена нет или он не начинается с Bearer - шлюз возвращает HTTP статус-код `401 UNAUTHORIZED`.


3. `api-gateway`, используя реактивный `WebClient`, инициирует проверку сессии: выполняется HTTP-запрос  [GET /validate](GET.validate)  в `auth-service` с заголовком `Authorization`.


4. `auth-service` проверяет токен и возвращает ответ:

    4.1. Если сервис неотвечает или произошла ошибка на сервере - `api-gateway` возвращет HTTP статус-код `500 INTERNAL SERVER ERROR`.

    4.1. Если токен не прошел проверку, сервис возвращает HTTP статус-код `401 UNAUTHORIZED`.

    4.2. Если токен прошел проверку, сервис возвращает HTTP статус-код `200 OK`.



5. `api-gateway` получает ответ `200 OK`и проксирует запрос к целевому сервису.


6. `api-gateway` получает ответ от целевого сервиса.
 

7. `api-gateway` проксирует ответ клиенту.
 
![APIGateway.validate.svg](..%2FDiagrams%2FAPIGateway.validate.svg)

## Алгоритм работы при запросе к публичным ресурсам системы

1. `api-gateway` получает запрос, анализирует URL запроса: ищет совпадение в `application.yml`. Если для найденного маршрута в списке `filters` `JwtValidation` не указан, `JwtValidationGatewayFilterFactory` не задействуется.
2. `api-gateway`  проксирует запрос к целевому сервису.


3. `api-gateway` получает ответ от целевого сервиса.


4. `api-gateway` проксирует ответ клиенту.

![APIGateway.routing.svg](..%2FDiagrams%2FAPIGateway.routing.svg)