# Endpoint `POST /login`

## Общая информация
**Назначение:** Аутентификация пользователя и генерация JWT-токена при успешном входе.

**Метод:** POST

**Внутренний URL:** `http://auth-service:4005/login`

**Внешний URL:** `http://{gateway-host}/auth/login`

---

## Входные и выходные параметры
<details>
<summary><span style="color: #2E86C1;"><b>Входные параметры</b></span></summary>

Body (JSON):

| Параметр | Тип данных | Обязательность | Описание | Значение/пример | Маппинг с auth_db |
|:------|:-----------|:---------------|:---------|:-----------------|:------------------|
| email | string | да | Логин пользователя | user@example.com | users.email |
| password | string | да | Пароль пользователя | password123 | users.password |


</details>

<details>
<summary><span style="color: #2E86C1;"><b>Выходные параметры</b></span></summary>

Body (JSON):

| Параметр | Тип данных | Обязательность | Описание | Значение/пример | Маппинг с auth_db |
|:------|:-----------|:---------------|:---------|:-----------------|:------------------|
| token | string | да | Сгенерированный JWT-токен | eyJhbGciOiJIUzI1NiJ9... | - |

</details>

---

## Примеры запроса и ответа

Запрос:
~~~json
POST /login
Content-Type: application/json

{
"email": "user@example.com",
"password": "securePassword123"
}
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
      <td rowspan="4">400</td>
      <td rowspan="4">BAD REQUEST</td>
      <td>"Email is required"</td>
      <td>Поле email пустое или отсутствует. 
    </tr>
    <tr>
      <td>"Email should be a valid email address"</td>
      <td>Некорректный формат email. 
    </tr>
    <tr>
      <td>"Password is required"</td>
      <td>Поле password пустое или отсутствует. 
    </tr>
    <tr>
      <td>"Password must be at least 8 characters long"</td>
      <td>Длина пароля менее 8 символов. 
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

---

## Алгоритм работы

1. `auth-service` через `AuthController` получает запрос `POST /login`. Входящий JSON-пакет десериализуется в Java-объект `LoginRequestDTO` и валидируются входные параметры:

<details>
<summary><span style="color: #2E86C1;">Валидация входных параметров </span></summary>

<table>
  <thead>
    <tr>
      <th>Параметр</th>
      <th>Результат (в случае ошибки)</th>
      <th>HTTP Код</th>
      <th>Сообщение (Body)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="2">email</td>
      <td>Строка является пустой, состоит только из пробелов или поле полностью отсутствует в JSON-запросе.</td>
      <td rowspan="4">400</td>
      <td>"Email is required"</td>
    </tr>
    <tr>
      <td>Текстовое значение не соответствует базовому формату адреса (отсутствует символ @, либо он расположен в начале/конце строки).</td>
      <td>"Email should be a valid email address"</td>
    </tr>
    <tr>
      <td rowspan="2">password</td>
      <td>Строка является пустой, состоит только из пробелов или поле полностью отсутствует в JSON-запросе.</td>
      <td>"Password is required"</td>
    </tr>
    <tr>
      <td>Количество символов в переданной строке меньше минимально допустимого порога (8 знаков).</td>
      <td>"Password must be at least 8 characters long"</td>
    </tr>
  </tbody>
</table>
</details>


2. `AuthController` вызывает метод `authenticate(LoginRequestDTO)` в `AuthService` для выполнения бизнес-логики аутентификации.


3. `AuthService` выполняет поиск пользователя в базе данных `auth_db`. Через интерфейс `UserService` вызывается метод `findByEmail`. С помощью Hibernate формируется и выполняется запрос к таблице `users` для извлечения записи по уникальному идентификатору `email`. Если запись в базе данных не найдена — `AuthService` возвращает пустое значение `Optional.empty()`, `AuthController` возвращает статус-код `401 UNAUTHORIZED`.


4. `AuthService` верифицирует пароль пользователя. Система извлекает зашифрованный хеш из найденного объекта пользователя в базе данных (поле `password`) и передает его вместе с «сырым» паролем из `LoginRequestDTO` в метод `matches()` компонента `BCryptPasswordEncoder`.

   4.1. Алгоритм `BCrypt` извлекает соль из сохраненного хеша, применяет её к паролю из запроса и выполняет побитовое сравнение. Если результаты не совпадают, метод `matches()` возвращает значение `false`, `AuthService` возвращает `Optional.empty()`, `AuthController` возвращает статус-код `401 UNAUTHORIZED`.

   Если в объекте `LoginRequestDTO` поле `password` имеет значение `null`, метод `BCryptPasswordEncoder.matches()` выбрасывает исключение `IllegalArgumentException` — `AuthController` возвращает статус-код `500 INTERNAL SERVER ERROR`.


5. `AuthService` генерирует `JWT`. После успешной аутентификации вызывается метод `jwtUtil.generateToken()`, который использует библиотеку `JJWT` для сборки токена:

   5.1. В `Payload` токена записывается: `email` пользователя в качестве `sub` и его роль в качестве `role` как пользовательский `Claim`.

   5.2. Устанавливается время выпуска `iat` и время истечения срока действия токена `exp` — через 10 часов от текущего момента.

   5.3. Токен подписывается методом `signWith`. Используется алгоритм шифрования `SignatureAlgorithm.HS256` и секретный ключ `Secret Key`, определенный в конфигурации сервиса.


6. `AuthController` упаковывает сгенерированную строку `JWT` в объект `LoginResponseDTO`. Перед отправкой HTTP-ответа выполняется сериализация объекта `LoginResponseDTO` в формат JSON. 


7. `auth-service` через `AuthController`  возвращет ответ клиенту `200 OK + JSON Body`.

Если в процессе выполнения метода возникает критический сбой\необработанные исключения, не связанный с данными токена (например, ошибка конфигурации секретного ключа или нехватка памяти сервера) — `auth-service` через `AuthController` возвращает статус-код `500 INTERNAL SERVER ERROR`.

---
![POST.login.svg](..%2FDiagrams%2FPOST.login.svg)