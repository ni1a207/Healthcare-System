# Endpoint `POST /patients`

## Общая информация
**Назначение:** Метод предназначен для добавления нового пациента в систему: сохранение данных о пациенте в системе и открытие счета для оплаты медицинских услуг.

**Метод:** `POST`

**Внутренний URL:** `http://patient-service:4000/patients`

**Внешний URL:** `http://{gateway-host}/api/patients`

---

## Входные и выходные параметры
<details>
<summary><b><font color="#2196F3">Входные параметры</font></b></summary>

Header:

| Параметр  | Тип данных | Обязательность | Описание                                        | Значение/пример  | Маппинг с patient_db |
|:------|:-----------|:---------------|:------------------------------------------------|:-----------------|:---------------------|
|Content-Type|string|да| Тип передаваемого контента                      | application/json | -                    |
| Authorization | string | да | Токен доступа (на уровне сервиса нет валидации) | Bearer <token> | - |
Body (JSON):

| Параметр | Тип данных | Обязательность | Описание                                             | Значение/пример   | Маппинг с patient_db    |
|:---------|:-----------|:---------------|:-----------------------------------------------------|:------------------|:------------------------|
| name | string | да | Имя пациента (max 100 символов) | John Doe | patient.name            |
| email | string | да | Электронная почта (должна быть уникальной) | john.doe@example.com | patient.email           |
| address | string | да | Адрес проживания | 123 Main St, Springfield | patient.address         |
| dateOfBirth | string | да | Дата рождения (формат ISO LocalDate) | 1985-06-15 | patient.date_of_birth   |
| registeredDate | string | да | Дата регистрации (обязательно при создании) | 2024-01-10 | patient.registered_date |
</details> 

<details>
<summary><b><font color="#2196F3">Выходные параметры</font></b></summary>
Body (JSON):

| Параметр | Тип данных | Описание                            | Значение/пример | Маппинг с patient_db |
|:---|:---|:------------------------------------|:---|:---|
| id | string | Уникальный идентификатор пациента, UUID генерируется сервером | 123e4567-e89b-12d3-a456-426614174000 | patient.id |
| name | string | Имя пациента                        | John Doe | patient.name |
| email | string | Электронная почта                   | john.doe@example.com | patient.email |
| address | string | Адрес проживания                    | 123 Main St, Springfield | patient.address |
| dateOfBirth | string | Дата рождения (YYYY-MM-DD)          | 1985-06-15 | patient.date_of_birth |
</details> 

---

## Примеры запроса и ответа

Запрос:
~~~ json
curl -X POST "{host}/api/patients" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
-d '
{
  "name": "John Doe",
  "email": "john.doe@example.com",
  "address": "123 Main St, Springfield",
  "dateOfBirth": "1985-06-15",
  "registeredDate": "2024-01-10"
}'
~~~

Ответ:
~~~json
Response code : 200 OK
Response body (json):
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "John Doe",
  "email": "john.doe@example.com",
  "address": "123 Main St, Springfield",
  "dateOfBirth": "1985-06-15"
}
~~~
<details>
<summary><span style="color: #2196F3;"><b>Варианты возвращаемых HTTP статус кодов</b></span></summary>


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
      <td>JSON с данными пациента</td>
      <td>Пациент успешно создан.</td>
    </tr>
    <tr>
      <td rowspan="6">400</td>
      <td rowspan="6">BAD REQUEST</td>
      <td>"Name is required"</td>
      <td>Поле name пустое или отсутствует.</td>
    </tr>
    <tr>
      <td>"Name cannot exceed 100 characters"</td>
      <td>Длина имени превышает 100 символов.</td>
    </tr>
    <tr>
      <td>"Email is required"</td>
      <td>Поле email пустое или отсутствует.</td>
    </tr>
    <tr>
      <td>"Email should be valid"</td>
      <td>Некорректный формат email.</td>
    </tr>
    <tr>
      <td>"Address is required"</td>
      <td>Поле address пустое или отсутствует.</td>
    </tr>
    <tr>
      <td>"Date of birth is required"</td>
      <td>Поле dateOfBirth пустое или отсутствует.</td>
    </tr>
    <tr>
      <td rowspan="2">400</td>
      <td rowspan="2">BAD REQUEST</td>
      <td>"Registered date is required"</td>
      <td>Поле registeredDate пустое или отсутствует.</td>
    </tr>
    <tr>
      <td>"Email address already exists"</td>
      <td>Пациент с таким email уже зарегистрирован.</td>
    </tr>
    <tr>
      <td>500</td>
      <td>INTERNAL SERVER ERROR</td>
      <td>-</td>
      <td>Неверный формат даты, Billing Service недоступен или вернул ошибку.</td>
    </tr>
  </tbody>
</table>
</details>


---
## Алгоритм работы

1. `patient-service` через `DispatcherServlet` принимает входящий HTTP-запрос `POST /patients`.


2. `DispatcherServlet` передает входящий `JSON` в библиотеку `Jackson` для выполнения десериализации.


3. `Jackson` десериализует тело запроса из `JSON` в объект `PatientRequestDTO` и возвращает в `DispatcherServlet`. Параметры `dateOfBirth` и `registeredDate` десериализуются просто как строки, без проверки формата (проверка формата на шаге 8).


4. `DispatcherServlet` валидирует параметры объекта `PatientRequestDTO`:
<details style="font-weight: normal;">
  <summary style="cursor: pointer; color: #2196F3; font-weight: normal;">
    Валидация параметров
  </summary>
  <div style="font-weight: normal;">
  <table>
    <thead>
      <tr>
        <th>Параметр</th>
        <th>Условие</th>
        <th>Сообщение</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td rowspan="2"><b>name</b></td>
        <td>Пустое или null</td>
        <td><code>"Name is required"</code></td>
      </tr>
      <tr>
        <td>Длина > 100 символов</td>
        <td><code>"Name cannot exceed 100 characters"</code></td>
      </tr>
      <tr>
        <td rowspan="2"><b>email</b></td>
        <td>Пустое или null</td>
        <td><code>"Email is required"</code></td>
      </tr>
      <tr>
        <td>Некорректный формат</td>
        <td><code>"Email should be valid"</code></td>
      </tr>
      <tr>
        <td><b>address</b></td>
        <td>Пустое или null</td>
        <td><code>"Address is required"</code></td>
      </tr>
      <tr>
        <td><b>dateOfBirth</b></td>
        <td>Пустое или null</td>
        <td><code>"Date of birth is required"</code></td>
      </tr>
      <tr>
        <td><b>registeredDate</b></td>
        <td>Пустое или null (только при создании)</td>
        <td><code>"Registered date is required"</code></td>
      </tr>
    </tbody>
  </table>
  </div>
</details>

   В случае ошибки валидации `GlobalExceptionHandler` перехватывает ошибку и отправляет ответ клиенту.


5. `DispatcherServlet` при успешной валидации вызывает метод `createPatient(patientRequestDTO)` в `PatientController`, передавая объект в качестве аргумента.


6. `PatientController` вызывает метод `createPatient(patientRequestDTO patientRequestDTO)` в `PatientService`.


7. `PatientService` выполняет проверку уникальности `email`, вызывая `patientRepository.existsByEmail(patientRequestDTO.getEmail())`. Это инициирует SQL-запрос в базу данных `patient_db`, в таблицу `patient` : `SELECT count(*) FROM patient WHERE email = ?`. Если такой `email` уже существует в БД, выбрасывается исключение `EmailAlreadyExistsException`, которое перехватывает `GlobalExceptionHandler`, клиенту возвращается HTTP статус-код `400 BAD REQUEST, message: "Email address already exists"`.


8. `PatientMapper` мапит данные из `PatientRequestDTO` в сущность `Patient`, парсит параметры `dateOfBirth` и `registeredDate` через `LocalDate.parse()`. Если строка не соответствует формату `ISO_LOCAL_DATE (YYYY-MM-DD)`, выбрасывается `DateTimeParseException`, `GlobalExceptionHandler` не перехватывает это исключение — клиенту возвращается HTTP статус-код `500 INTERNAL SERVER ERROR`.


9. `PatientRepository` делает запись в базу данных `patient_db` в таблицу `patient`. `Hibernate` запрашивает соединение у пула `HikariCP`, генерирует уникальный идентификатор записи `UUID` и выполняет SQL-запрос `INSERT INTO patient (id, name, email, address, date_of_birth, registered_date) VALUES (?, ?, ?, ?, ?, ?);`. 


10. `PatientService` формирует gRPC-сообщение `BillingRequest` и вызывает `BillingServiceGrpcClient`.


11. `BillingServiceGrpcClient` выполняет блокирующий вызов метода `createBillingAccount` в `Billing Service`, передаются параметры `patientId` (UUID), `name` и `email`. 
 
 
12. `Billing Service` обрабатывает запрос, возвращает объект с данными открытого счета (в коде данные захардкожены, всегда будет возвращаться объект с заданными параметрами). Если `Billing Service` недоступен или вернул ошибку выбрасывается исключение `StatusRuntimeException`,  `GlobalExceptionHandler` перехватывает ошибку и возвращает клиенту HTTP статус-код `500 INTERNAL SERVER ERROR`. В методе `createPatient` нет блока `try-catch` и аннотации `@Transactional`, выполнение запроса прерывается.

~~~json
{
    "accountId": "12345",
    "status": "ACTIVE"
}
~~~    

> **Важно**: Запись в базе данных `patient_db` на шаге 9 уже зафиксирована (COMMIT). В случае ошибки, в системе возникает рассинхрон: пациент в БД создан, но финансовый счет в `BillingService` отсутствует.

13. При успешном ответе от `Billing Service` (статус OK), `PatientService` вызывает `KafkaProducer.sendEvent(newPatient)`.


14. `KafkaProducer` сериализует Protobuf-сообщение `PatientEvent` (тип PATIENT_CREATED) и отправляет его в топик `patient`. Если Kafka недоступна или возник таймаут, исключение перехватывается внутри `KafkaProducer` — ошибка записывается в лог, выполнение продолжается. Клиент получает `200 OK`, запись в БД сохранена, gRPC-вызов выполнен. Сообщение в Kafka считается потерянным (Data Inconsistency).


15. `PatientMapper` мапит сохраненную сущность в объект `PatientResponseDTO`.


16. `PatientController` оборачивает `DTO` в `ResponseEntity.ok()` и возвращает его в `DispatcherServlet`.


17. `Jackson` выполняет сериализацию объекта в `JSON`.


18. `DispatcherServlet` записывает `JSON` в тело ответа и отправляет его клиенту со статусом `200 OK`.


![POST.patients.svg](..%2FDiagrams%2FPOST.patients.svg)

---

## Логирование

| Шаг в алгоритме       | Уровень | Класс | Сообщение |
|:----------------------|:--------|:----|:--------|
| 11                    | INFO    |BillingServiceGrpcClient|Received response from billing service via GRPC: {response}|
| 14                    | ERROR   |KafkaProducer| Error sending PatientCreated event: {event}|
| При старте приложения | INFO    |BillingServiceGrpcClient |Connecting to Billing Service GRPC service at {address}:{port}|