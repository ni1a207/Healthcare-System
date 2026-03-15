# Endpoint `PUT /patients/{id}`

## Общая информация
**Назначение:** Метод предназначен для изменения данных у существующего пациента в системе.

**Метод:** `PUT`

**Внутренний URL:** `http://patient-service:4000/patients/{id}`

**Внешний URL:** `http://{gateway-host}/api/patients/{id}`

---

## Входные и выходные параметры
<details>
<summary><b><font color="#2196F3">Входные параметры</font></b></summary>

Path:

| Параметр | Тип данных | Обязательность | Описание                          | Значение/пример  | Маппинг с patient-service-db |
|:---------|:-----------|:---------------|:----------------------------------|:-----------------|:---------------------|
| id       | UUID       |да| Уникальный идентификатор пациента | 123e4567-e89b-12d3-a456-426614174000 | patient.id           |


Header:

| Параметр  | Тип данных | Обязательность | Описание                                        | Значение/пример  | Маппинг с patient-service-db |
|:------|:-----------|:---------------|:------------------------------------------------|:-----------------|:---------------------|
|Content-Type|string|да| Тип передаваемого контента                      | application/json | -                    |
| Authorization | string | да | Токен доступа (на уровне сервиса нет валидации) | Bearer <token> | - |
Body (JSON):

| Параметр | Тип данных | Обязательность | Описание                                             | Значение/пример   | Маппинг с patient-service-db    |
|:---------|:-----------|:---------------|:-----------------------------------------------------|:------------------|:------------------------|
| name | string | да | Имя пациента (max 100 символов) | John Doe | patient.name            |
| email | string | да | Электронная почта (должна быть уникальной) | john.doe@example.com | patient.email           |
| address | string | да | Адрес проживания | 123 Main St, Springfield | patient.address         |
| dateOfBirth | string | да | Дата рождения (формат ISO LocalDate) | 1985-06-15 | patient.date_of_birth   |
</details> 

<details>
<summary><b><font color="#2196F3">Выходные параметры</font></b></summary>
Body (JSON):

| Параметр | Тип данных | Описание | Значение/пример | Маппинг с patient-service-db |
|:---|:---|:---|:---|:---|
| id | string | Уникальный идентификатор пациента (UUID) | 123e4567-e89b-12d3-a456-426614174000 | patient.id |
| name | string | Имя пациента | John Doe | patient.name |
| email | string | Электронная почта | john.doe@example.com | patient.email |
| address | string | Адрес проживания | 123 Main St, Springfield | patient.address |
| dateOfBirth | string | Дата рождения (YYYY-MM-DD) | 1985-06-15 | patient.date_of_birth |
</details> 

---

## Примеры запроса и ответа

Запрос:
~~~ json
curl -X PUT "/patients/{id}" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
-d '{
  "name": "John Doe",
  "email": "john.doe@example.com",
  "address": "123 Main St, Springfield",
  "dateOfBirth": "1985-06-15"
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
<summary><b><font color="#2196F3">Варианты возвращаемых HTTP статус кодов</font></b></summary>

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
      <td>Данные пациента успешно обновлены.</td>
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
      <td>"Patient not found"</td>
      <td>Пациент с указанным id не найден.</td>
    </tr>
    <tr>
      <td>"Email address already exists"</td>
      <td>Пациент с таким email уже зарегистрирован.</td>
    </tr>
    <tr>
      <td>500</td>
      <td>INTERNAL SERVER ERROR</td>
      <td>-</td>
      <td>Неверный формат даты.</td>
    </tr>
  </tbody>
</table>
</details>

---
## Алгоритм работы

1. `patient-service` через `DispatcherServlet` принимает входящий HTTP-запрос `PUT /patients/{id}`.


2. `DispatcherServlet` передаёт входящий JSON в библиотеку `Jackson` для десериализации.


3. `Jackson` десериализует тело запроса из JSON в объект `PatientRequestDTO` и возвращает в `DispatcherServlet`. Параметр `dateOfBirth` десериализуется как строка без проверки формата (проверка формата на шаге 8).


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
    </tbody>
  </table>
  </div>
</details>

> **Важно:** Поле `registeredDate` не валидируется при обновлении — в `PatientController` используется `@Validated({Default.class})` без `CreatePatientValidationGroup`.

В случае ошибки валидации `GlobalExceptionHandler` перехватывает `MethodArgumentNotValidException` и возвращает клиенту HTTP статус-код `400 BAD REQUEST + message`.


5. `DispatcherServlet` при успешной валидации вызывает метод `updatePatient(id, patientRequestDTO)` в `PatientController`.


6. `PatientController` вызывает метод `updatePatient(id, patientRequestDTO)` в `PatientService`.


7. `PatientService` выполняет поиск пациента по `id`, вызывая `patientRepository.findById(id)`. `Hibernate` выполняет SQL-запрос к таблице `patient` базы данных `patient-service-db`: `SELECT * FROM patient WHERE id = ?`. Если запись не найдена — выбрасывается `PatientNotFoundException`, `GlobalExceptionHandler` перехватывает исключение и возвращает клиенту HTTP статус-код `400 BAD REQUEST, message: "Patient not found"`.


8. `PatientService` выполняет проверку уникальности `email`, вызывая `patientRepository.existsByEmailAndIdNot(email, id)`. Это исключает из проверки самого пациента (допускает сохранение того же email). SQL-запрос: `SELECT count(*) FROM patient WHERE email = ? AND id != ?`. Если такой `email` уже существует у другого пациента — выбрасывается `EmailAlreadyExistsException`, `GlobalExceptionHandler` перехватывает исключение и возвращает клиенту HTTP статус-код `400 BAD REQUEST, message: "Email address already exists"`.


9. `PatientService` обновляет поля сущности `Patient`: `name`, `address`, `email`, и парсит `dateOfBirth` через `LocalDate.parse(patientRequestDTO.getDateOfBirth())`. Если строка не соответствует формату `ISO_LOCAL_DATE (YYYY-MM-DD)` — выбрасывается `DateTimeParseException`, `GlobalExceptionHandler` не перехватывает это исключение — клиенту возвращается HTTP статус-код `500 INTERNAL SERVER ERROR`.

> **Важно:** При `DateTimeParseException` изменения не сохраняются в БД — метод `patientRepository.save()` не был вызван.

10. `PatientRepository` сохраняет обновлённую сущность в базу данных `patient_db`. `Hibernate` запрашивает соединение у пула `HikariCP` и выполняет SQL-запрос `UPDATE patient SET name = ?, address = ?, email = ?, date_of_birth = ? WHERE id = ?`.


11. `PatientMapper` мапит обновлённую сущность в объект `PatientResponseDTO`.


12. `PatientController` оборачивает DTO в `ResponseEntity.ok()` и возвращает в `DispatcherServlet`.


13. `Jackson` выполняет сериализацию объекта в JSON.


14. `DispatcherServlet` записывает JSON в тело ответа и отправляет клиенту со статусом `200 OK`.

![PUT.patients.id.svg](..%2FDiagrams%2FPUT.patients.id.svg)
---

## Логирование

| Шаг в алгоритме | Уровень | Класс                      | Сообщение                                   |
|:----------------|:--------|:---------------------------|:--------------------------------------------|
| 7               | WARN    | GlobalExceptionHandler     | Patient not found {message}|
| 8               | WARN    | GlobalExceptionHandler                           | Email address already exist {message}|