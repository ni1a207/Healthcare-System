# Endpoint `PUT /patients/{id}`

### Общая информация
**Назначение:** Метод предназначен для изменении данных у существующего пациента в системе.

**Метод:** `PUT`

**Внутренний URL:** `http://patient-service:4000/patients/{id}`

**Внешний URL:** `http://{gateway-host}/api/patients{id}`

---

### Входные и выходные параметры
<details>
<summary><b><font color="#2196F3">Входные параметры</font></b></summary>

Path:

| Параметр | Тип данных | Обязательность | Описание                          | Значение/пример  | Маппинг с patient_db |
|:---------|:-----------|:---------------|:----------------------------------|:-----------------|:---------------------|
| id       | UUID       |да| Уникальный идентификатор пациента | 123e4567-e89b-12d3-a456-426614174000 | patient.id           |


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
curl -X PUT "/patients/{id}" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
-d '{
  "name": "John Doe",
  "email": "john.doe@example.com",
  "address": "123 Main St, Springfield",
  "dateOfBirth": "1985-06-15",
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

1. `patient-service` через `PatientController` получает входящий запрос `PUT  /patients/{id}` с объектом `PatientRequestDTO`, валидирует входные параметры:
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


2. `PatientService` парсит поле `dateOfBirth` в объект `LocalDate`, если формат даты в строке не соответствует `ISO_LOCAL_DATE (YYYY-MM-DD)` - сервис возвращет HTTP статус код `500 INTERNAL SERVER ERROR`.


3. `PatientService` через `PatientRepository` обновляет данные в базе данных `patient_db` в таблице `patient`. Если запись в базе данных не найдена по `id` - сервис возвращает HTTP статус-код `404 NOT FOUND`.


4. `Patient Service` возвращает ответ `200 OK + JSON` через `API Gateway` на клиент. 

