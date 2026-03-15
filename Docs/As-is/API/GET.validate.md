# Endpoint `GET /validate`

## Общая информация
**Назначение:** Валидация токена для запросов к защищенным ресурсам системы.

**Метод:** GET

**Внутренний URL:** `http://auth-service:4005/validate`

**Внешний URL:** `http://{gateway-host}/auth/validate`

---

## Входные и выходные параметры
<details>
<summary><span style="color: #2E86C1;"><b>Входные параметры</b></span></summary>
Header:

| Параметр | Тип данных | Обязательность | Описание | Значение/пример | Маппинг с auth-service-db |
|:------|:-----------|:---------------|:---------|:-----------------|:--------------------------|
|Authorization|string|да|Bearer <token>| Bearer eyJhbGciOiJIUzI1NiJ9... | -                         |

</details>

<details>
<summary><span style="color: #2E86C1;"><b>Выходные параметры</b></span></summary>

Тело ответа отсутствует — `ResponseEntity<Void>`. Результат валидации передаётся через HTTP статус-код.

</details>

---

## Примеры запроса и ответа

Запрос:
~~~json
curl -X GET "http://{gateway-host}/auth/validate" \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..."
~~~

Ответ:
~~~json
Response code : 200 OK
~~~

<details>
<summary><span style="color: #2E86C1;"><b>Варианты возвращаемых HTTP статус кодов</b></span></summary>

<table>
  <thead>
    <tr>
      <th>Код</th>
      <th>Статус</th>
      <th>Сообщение (Body)</th>
      <th>Описание</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>200</td>
      <td>OK</td>
      <td>-</td>
      <td>Токен валиден. Успешно пройдены проверки структуры, подписи и срока действия.</td>
    </tr>
    <tr>
      <td>400</td>
      <td>BAD REQUEST</td>
      <td>-</td>
      <td>Заголовок Authorization отсутствует в запросе — Spring автоматически возвращает 400 до выполнения кода контроллера.</td>
    </tr>
    <tr>
      <td rowspan="2">401</td>
      <td rowspan="2">UNAUTHORIZED</td>
      <td>-</td>
      <td>Заголовок Authorization не содержит префикс "Bearer ".</td>
    </tr>
    <tr>
      <td>-</td>
      <td>JJWT выбросила JwtException — неверная подпись, истёкший срок действия или повреждённая структура токена.</td>
    </tr>
    <tr>
      <td>500</td>
      <td>INTERNAL SERVER ERROR</td>
      <td>-</td>
      <td>Критический сбой — ошибка конфигурации секретного ключа или непредвиденная системная ошибка.</td>
    </tr>
  </tbody>
</table>
</details>


---

## Алгоритм работы 

1. `auth-service` через `DispatcherServlet` принимает входящий HTTP-запрос `GET /validate`.


2. `DispatcherServlet` вызывает метод `validateToken(authHeader)` в `AuthController`. Spring извлекает значение заголовка `Authorization` через аннотацию `@RequestHeader("Authorization")` и передаёт его в параметр `authHeader`. Если заголовок `Authorization` отсутствует в запросе — Spring автоматически возвращает HTTP статус-код `400 BAD REQUEST` до выполнения кода контроллера.


3. `AuthController` проверяет что значение `authHeader` начинается с префикса `Bearer `. Если префикс отсутствует — `AuthController` возвращает HTTP статус-код `401 UNAUTHORIZED`.


4. `AuthController` отсекает префикс `Bearer ` (первые 7 символов) через `authHeader.substring(7)` и вызывает метод `authService.validateToken(token)`.


5. `AuthService` вызывает метод `jwtUtil.validateToken(token)` внутри блока try-catch. `JwtUtil` с помощью библиотеки `JJWT` выполняет последовательные проверки токена:
5.1. Проверка структуры — токен должен состоять из трёх частей (`Header.Payload.Signature`), разделённых точками.
5.2. Проверка подписи — `secretKey` используется для вычисления хеша и сравнения с `Signature` в токене. При несовпадении — `JJWT` выбрасывает `SignatureException`, который перехватывается и оборачивается в `JwtException("Invalid JWT signature")`.
5.3. Проверка срока действия — из `Payload` извлекается клейм `exp` и сравнивается с текущим временем сервера. Если токен истёк — `JJWT` выбрасывает `JwtException`.


6. Если `jwtUtil.validateToken(token)` выбросил любое исключение наследуемое от `JwtException` — `AuthService` перехватывает его в блоке catch и возвращает false. `AuthController` получает false и возвращает HTTP статус-код `401 UNAUTHORIZED`.


7. Если все проверки пройдены успешно — `jwtUtil.validateToken(token)` завершается без исключения, `AuthService` возвращает true. `AuthController` получает true и возвращает `DispatcherServlet` ответ `ResponseEntity.ok().build()`.


8. `DispatcherServlet` отправляет клиенту `200 OK` без тела ответа.

Если в процессе выполнения возникает критический сбой — необработанное исключение не связанное с валидацией токена (например, ошибка конфигурации секретного ключа) — `auth-service` возвращает HTTP статус-код `500 INTERNAL SERVER ERROR`.

![GET.validate.svg](..%2FDiagrams%2FGET.validate.svg)

---
## Логирование

| Шаг в алгоритме       | Уровень | Класс | Сообщение                                   |
|:----------------------|:--------|:----|:--------------------------------------------|
|-|-|-| Логирование в данном методе не предусмотрено|