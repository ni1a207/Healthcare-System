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
Body:

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
Body:

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

1. `Patient Service`получает через `API  Gateway` входящий запрос `POST  /api/patients`. Сервис валидирует входные параметры:
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


2. `Patient Service` парсит поля `dateOfBirth` и `registeredDate` в объекты `LocalDate`, если формат даты в строке не соответствует `ISO_LOCAL_DATE (YYYY-MM-DD)` - сервис возвращет HTTP статус код `500 INTERNAL SERVER ERROR` .


3. `Patient Service` проверяет, существует ли уже такой `email` в `patient_db`. Если `email` найден - сервис возвращает  HTTP статус код `400 BAD REQUEST massage: "Email address already exists"`.  Если `email` не найден, геннерируется `UUID` и данные, переданные во входных параметрах, записываются в `patient_db` таблицу `patient`.


4. `Patient Service` вызывает gRPC метод в `Billing Service BillingServiceGrpcClient.createBillingAccount`:

   4.1. Формируется `BillingRequest` `{"patientId": "123e4567...", "name": "John Doe", "email": "john.doe@example.com"}`(patientId - сгенерированный UUID на шаге 3)

   4.2. Выполняется блокирующий вызов через [blockingStub.createBillingAccount](gRPC.createBilling.md). Если `Billing Service` недоступен или вернул ошибку gRPC - сервис возвращает HTTP сатус код `500 INTERNAL SERVER ERROR`.


5. `Patient Service` получает ответ `Status: OK (GRPC_STATUS_OK) Response body: {"accountId": "12345", "status": "ACTIVE"}` от `Billing Service`.


6. `Patient Service` отправляет событие в `Kafka`, вызывается [KafkaProducer.sendEvent(patient)](Kafka.PatientEvent%20.producer.md).

   6.1. Создается Protobuf-сообщение `PatientEvent` с типом `PATIENT_CREATED`.

   6.2.Выполняется отправка в топик `patient`.


7. `Patient Service` возвращает ответ `200 OK + JSON` через `API Gateway` на клиент. - никаких ошибок нет? 

![POST.api.patients.svg](..%2FDiagrams%2FPOST.api.patients.svg)