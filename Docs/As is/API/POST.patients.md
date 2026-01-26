# Endpoint `POST {host}/api/patients`

### Общая информация
**Назначение:** Метод предназначен для добавления нового пациента в систему: сохранение данных о пациенте в системе и открытие счета для оплаты медицинских услуг.

**Метод:** POST

**URL:** {host}/api/patients

---

### Входные и выходные параметры
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

| Параметр | Тип данных | Описание | Значение/пример | Маппинг с patient_db |
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
~~~ 
curl -X POST "{host}/api/v1/patients" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
-d '{
  "name": "John Doe",
  "email": "john.doe@example.com",
  "address": "123 Main St, Springfield",
  "dateOfBirth": "1985-06-15",
  "registeredDate": "2024-01-10"
}
~~~

Ответ:
~~~
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


---
## Алгоритм работы

1. `patient-service` через `DispatcherServlet` принимает входящий HTTP-запрос `POST /patients`.


2. `DispatcherServlet` передает входящий `JSON` в библиотеку `Jackson` для выполнения десериализации.


3. `Jackson` десериализует тело запроса из `JSON` в объект `PatientRequestDTO` и возвращает в `DispatcherServlet`. Если формат полей типа `LocalDate` (`dateOfBirth`, `registeredDate`) не соответствует `ISO_LOCAL_DATE (YYYY-MM-DD)`, `Jackson` выбрасывает `InvalidFormatException`. `GlobalExceptionHandler` перехватывает ошибку и возвращает HTTP статус-код `400 BAD REQUEST`.


4. `DispatcherServlet` валидирует параметры объекта `PatientRequestDTO`:
    <details style="font-weight: normal;">
      <summary style="cursor: pointer; color: #2196F3; font-weight: normal;">
        Валидация параметров
      </summary>
      <div style="font-weight: normal;">

   | Параметр | Результат |
          |:---|:---|
   | name | Ошибка валидации: параметр пустой или передан как null.<br>`400 BAD REQUEST message: "Name is required"`<br><br>Ошибка валидации: длина имени превышает 100 символов.<br>`400 BAD REQUEST message: "Name cannot exceed 100 characters"` |
   | email | Ошибка валидации: поле пустует или передано как null.<br>`400 BAD REQUEST message: "Email is required"`<br><br>Ошибка валидации: строка не соответствует формату адреса электронной почты.<br>`400 BAD REQUEST message: "Email should be valid"` |
   | address | Ошибка валидации: поле пустует или передано как null.<br>`400 BAD REQUEST message: "Address is required"` |
   | dateOfBirth | Ошибка валидации: поле пустует или передано как null.<br>`400 BAD REQUEST message: "Date of birth is required"` |
   | registeredDate | Ошибка валидации: поле отсутствует или передано пустым при создании записи.<br>`400 BAD REQUEST message: "Registered date is required"` |

      </div>
    </details>

   В случае ошибки валидации `GlobalExceptionHandler` перехватывает ошибку и отправляет ответ клиенту.


5. `DispatcherServlet` при успешной валидации вызывает метод `createPatient(patientRequestDTO)` в `PatientController`, передавая объект в качестве аргумента.


6. `PatientController` вызывает метод `createPatient(patientRequestDTO patientRequestDTO)` в `PatientService`.


7. `PatientService` выполняет проверку уникальности `email`, вызывая `patientRepository.existsByEmail(patientRequestDTO.getEmail())`. Это инициирует SQL-запрос в базу данных `patient_db`, в таблицу `patient` : `SELECT count(*) FROM patient WHERE email = ?`. Если такой `email` уже существует в БД, выбрасывается исключение `EmailAlreadyExistsException`, которое перехватывает `GlobalExceptionHandler`, клиенту возвращается HTTP статус-код `400 BAD REQUEST, message: "Email address already exists"`.


8. `PatientMapper` мапит данные из `PatientRequestDTO` в сущность `Patient`. Генерируется уникальный идентификатор записи `UUID`.


9. `PatientRepository` делает запись в базу данных `patient_db` в таблицу `patient`. `Hibernate` запрашивает соединение у пула `HikariCP` и выполняет SQL-запрос `INSERT INTO patient (id, name, email, address, date_of_birth, registered_date) VALUES (?, ?, ?, ?, ?, ?);`. 


10. `PatientService` формирует gRPC-сообщение `BillingRequest` и вызывает `BillingServiceGrpcClient`.


11. `BillingServiceGrpcClient` выполняет блокирующий вызов метода `createBillingAccount` в `Billing Service`, передаются параметры `patientId` (UUID), `name` и `email`. 
 
 
12. `Billing Service` обрабатывает запрос, возвращает объект с данными открытого счета (в коде данные захардкожены, всегда будет возвращаться объект с параметрами: {
    "patientId": "12333",
    "name" : "John Doe",
    "email" : "john.doe@example.com"
    })
Если `Billing Service` недоступен или вернул ошибку выбрасывается исключение `StatusRuntimeException`,  `GlobalExceptionHandler` перехватывает ошибку и возвращает клиенту HTTP статус-код `500 INTERNAL SERVER ERROR`. В методе `createPatient` нет блока `try-catch` и аннотации `@Transactional`, выполнение запроса прерывается.

**Важно!** Запись в базе данных `patient_db` на шаге 9 уже зафиксирована (COMMIT). В системе возникает рассинхрон: пациент в БД создан, но финансовый счет в `BillingSrvice` отсутствует.

12. При успешном ответе от `Billing Service` (статус OK), `PatientService` вызывает `KafkaProducer.sendEvent(newPatient)`.


13. `KafkaProducer` сериализует Protobuf-сообщение `PatientEvent` (тип PATIENT_CREATED) и отправляет его в топик `patient`. Если `Kafka` недоступен, выбрасывается исключение, `GlobalExceptionHandler` возвращает клиенту HTTP статус-код `500 INTERNAL SERVER ERROR`.


14. `PatientMapper` мапит сохраненную сущность в объект `PatientResponseDTO`.


15. `PatientController` оборачивает `DTO` в `ResponseEntity.ok()` и возвращает его в `DispatcherServlet`.


16. `Jackson` выполняет сериализацию объекта в `JSON`.


17. `DispatcherServlet` записывает `JSON` в тело ответа и отправляет его клиенту со статусом `200 OK`.

![POST.patients.svg](..%2FDiagrams%2FPOST.patients.svg)
