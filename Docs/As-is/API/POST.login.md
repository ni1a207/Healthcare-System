# Endpoint `POST /login`

## Общая информация
**Назначение:** Аутентификация пользователя и генерация JWT-токена при успешном входе.

**Метод:** `POST`

**Внутренний URL:** `http://auth-service:4005/login`

**Внешний URL:** `http://{gateway-host}/auth/login`

---

## Входные и выходные параметры
<details>
<summary><span style="color: #2E86C1;"><b>Входные параметры</b></span></summary>

Body (JSON):

| Параметр | Тип данных | Обязательность | Описание | Значение/пример | Маппинг с auth-service-db |
|:------|:-----------|:---------------|:---------|:-----------------|:--------------------------|
| email | string | да | Логин пользователя | user@example.com | users.email               |
| password | string | да | Пароль пользователя | password123 | users.password            |


</details>

<details>
<summary><span style="color: #2E86C1;"><b>Выходные параметры</b></span></summary>

Body (JSON):

| Параметр | Тип данных | Описание | Значение/пример | Маппинг с auth-service-db |
|:------|:---------------|:---------|:-----------------|:--------------------------|
| token | string |  Сгенерированный JWT-токен | eyJhbGciOiJIUzI1NiJ9... | -                         |

</details>

---

## Примеры запроса и ответа

Запрос:
~~~json
curl -X POST "http://{gateway-host}/auth/login" \
-H "Content-Type: application/json" \
-d '{
"email": "user@example.com",
"password": "securePassword123"
}'
~~~

Ответ (Успех):
~~~json
Response code : 200 OK

{
"token": "eyJhbGciOiJIUzI1NiJ9..."
}
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
      <td>JSON с токеном</td>
      <td>Успешная проверка email, password.</td>
    </tr>
    <tr>
      <td>401</td>
      <td>UNAUTHORIZED</td>
      <td>- </td>
      <td>Пользователь не найден или пароль не совпал.</td>
    </tr>
    <tr>
      <td>500</td>
      <td>INTERNAL SERVER ERR</td>
      <td>-</td>
      <td>Системная ошибка (БД недоступна, ошибка ключа).</td>
    </tr>
  </tbody>
</table>
</details>

> **Важно:** Валидация входных параметров не реализована — в AuthController отсутствует аннотация @Valid. Поля email и password принимаются без проверки формата и обязательности. HTTP статус-код 400 BAD REQUEST недостижим.
---

## Алгоритм работы


1. `auth-service` через `DispatcherServlet` принимает входящий HTTP-запрос `POST /login`.


2. `DispatcherServlet` передаёт входящий JSON в библиотеку `Jackson` для десериализации.


3. `Jackson` десериализует тело запроса из JSON в объект `LoginRequestDTO` и возвращает в `DispatcherServlet`.


> **Важно:** Аннотация `@Valid` в `AuthController` отсутствует — валидация полей `LoginRequestDTO` не выполняется. Некорректные или пустые значения передаются в `AuthService` без проверки.


4. `DispatcherServlet` вызывает метод `login(loginRequestDTO)` в `AuthController`.


5. `AuthController` вызывает метод `authenticate(loginRequestDTO)` в `AuthService`.


6. `AuthService` через `UserService` вызывает метод `findByEmail(email)`, который обращается к `UserRepository`. `Hibernate` формирует и выполняет SQL-запрос к таблице `users` базы данных `auth-service-db`: `SELECT * FROM users WHERE email = ?`. Если запись не найдена — `UserRepository` возвращает `Optional.empty()`, `AuthService` возвращает `Optional.empty()`, `AuthController` возвращает HTTP статус-код `401 UNAUTHORIZED`.


7. `AuthService` верифицирует пароль. Из найденного объекта `User` извлекается хеш пароля (поле password) и передаётся вместе с сырым паролем из `LoginRequestDTO` в метод `passwordEncoder.matches(rawPassword, hashedPassword) `компонента `BCryptPasswordEncoder`. Алгоритм `BCrypt` извлекает соль из хеша, применяет её к сырому паролю и выполняет побитовое сравнение. Если результаты не совпадают — `matches()` возвращает `false`, `AuthService` возвращает `Optional.empty()`, `AuthController` возвращает HTTP статус-код `401 UNAUTHORIZED`. Если поле `password` в `LoginRequestDTO` имеет значение `null` — `BCryptPasswordEncoder.matches()` выбрасывает `IllegalArgumentException`, `AuthController` возвращает HTTP статус-код `500 INTERNAL SERVER ERROR`.


8. `AuthService` вызывает метод `jwtUtil.generateToken(email, role)`. `JwtUtil` с помощью библиотеки `JJWT` собирает токен: в `Payload` записывается `email` пользователя как `sub` и его роль как пользовательский Claim `role`. Устанавливается время выпуска `iat` и время истечения `exp` — через 10 часов от текущего момента. Токен подписывается методом `signWith(secretKey)` — алгоритм подписи `HS256` определяется автоматически библиотекой `JJWT` на основе типа ключа. Возвращается готовая JWT-строка.


9. `AuthService` возвращает `Optional<String>` с JWT-токеном в `AuthController`.


10. `AuthController` оборачивает токен в объект `LoginResponseDTO` и передаёт в `DispatcherServlet`.


11. `Jackson` выполняет сериализацию `LoginResponseDTO` в JSON.


12. `DispatcherServlet` записывает JSON в тело ответа и отправляет клиенту со статусом `200 OK`.

Если в процессе выполнения алгоритма возникает критический сбой — необработанное исключение, не связанное с данными пользователя (например, недоступность БД или ошибка конфигурации секретного ключа) — `auth-service` возвращает HTTP статус-код `500 INTERNAL SERVER ERROR`.

![POST.login.svg](..%2FDiagrams%2FPOST.login.svg)

---

## Логирование

| Шаг в алгоритме       | Уровень | Класс | Сообщение                                   |
|:----------------------|:--------|:----|:--------------------------------------------|
|-|-|-| Логирование в данном методе не предусмотрено|