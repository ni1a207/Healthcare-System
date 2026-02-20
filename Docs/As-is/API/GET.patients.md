# Endpoint `GET /patients`

### Общая информация
**Назначение:** Метод предназначен для получения списка пациентов зарегистрированных в системе.

**Метод:** `GET`

**Внутренний URL:** `http://patient-service:4000/patients`

**Внешний URL:** `http://{gateway-host}/api/patients`

---

### Входные и выходные параметры
<details>
<summary><b><font color="#2196F3">Входные параметры</font></b></summary>

Header:

| Параметр  | Тип данных | Обязательность | Описание                                        | Значение/пример  | Маппинг с patient_db |
|:------|:-----------|:---------------|:------------------------------------------------|:-----------------|:---------------------|
| Authorization | string | да | Токен доступа (на уровне сервиса нет валидации) | Bearer <token> | - |
</details> 

<details>

<summary><b><font color="#2196F3">Выходные параметры</font></b></summary>

<table>
  <thead>
    <tr>
      <th colspan="2">Параметр</th>
      <th>Тип данных</th>
      <th>Описание</th>
      <th>Значение/пример</th>
      <th>Маппинг с patient_db</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td colspan="2"><b>[ ]</b></td>
      <td>Array</td>
      <td>Массив объектов пациентов</td>
      <td>—</td>
      <td>—</td>
    </tr>
    <tr>
      <td style="width: 20px;"></td>
      <td>id</td>
      <td>string (UUID)</td>
      <td>Уникальный идентификатор записи в БД</td>
      <td>123e4567-e89b-12d3-a456-426614174000</td>
      <td>patient.id</td>
    </tr>
    <tr>
      <td></td>
      <td>name</td>
      <td>string</td>
      <td>ФИО пациента</td>
      <td>John Doe</td>
      <td>patient.name</td>
    </tr>
    <tr>
      <td></td>
      <td>email</td>
      <td>string</td>
      <td>Электронная почта</td>
      <td>john.doe@example.com</td>
      <td>patient.email</td>
    </tr>
    <tr>
      <td></td>
      <td>address</td>
      <td>string</td>
      <td>Адрес проживания</td>
      <td>123 Main St, Springfield</td>
      <td>patient.address</td>
    </tr>
    <tr>
      <td></td>
      <td>dateOfBirth</td>
      <td>string (date)</td>
      <td>Дата рождения пациента</td>
      <td>1985-06-15</td>
      <td>patient.date_of_birth</td>
    </tr>
     <tr>
      <td></td>
      <td>registeredDate</td>
      <td>string (date)</td>
      <td>Дата регистрации</td>
      <td>2025-12-20</td>
      <td>patient.registered_date</td>
    </tr>		
  </tbody>
</table></details> 

---

## Примеры запроса и ответа

Запрос:
~~~ 
curl -X GET "/patients" \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \

~~~

Ответ:
~~~
Response code : 200 OK
Response body (json):
[
  {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "address": "123 Main St, Springfield",
    "dateOfBirth": "1985-06-15",
    "registeredDate": "2025-12-10"
  },
  {
    "id": "6ff9700a-040a-4119-a7a9-f6662122e423",
    "name": "Иван Иванов",
    "email": "ivan@mail.com",
    "address": "г. Москва, ул. Ленина",
    "dateOfBirth": "1990-05-15",
    "registeredDate": "2025-12-10"
  }
]
~~~


---
## Алгоритм работы

1. **`patient-service`** через **`DispatcherServlet`** получает входящий HTTP-запрос `GET /patients`. Запрос мапится на метод **`getPatients()`** в **`PatientController`**.


2. **`PatientController`** вызывает метод **`getPatients()`** в **`PatientService`**.


3. **`PatientService`** вызывает метод **`findAll()`** в **`PatientRepository`**.


4. **`PatientRepository`** передает управление реализации **`SimpleJpaRepository`** (Proxy), которая обеспечивает выполнение стандартной операции JPA.


5. `Hibernate (JPA Provider)` генерирует SQL-запрос `SELECT * FROM patient`. Для его выполнения он запрашивает активное соединение у пула соединений `HikariCP`, который передает запрос через `JDBC Driver` в базу данных `patient_db`. При невозможности получить соединение от `HikariCP` или отказе базы данных, выполнение прерывается, и управление передается в `GlobalExceptionHandler`. Данный компонент перехватывает исключение, формирует структуру ошибки и возвращает клиенту `500 INTERNAL SERVER ERROR`.


6. **`patient_db`** выполняет запрос, возвращает результат в виде `ResultSet`.


7. `Hibernate  (JPA Provider)` получает ответ от базы данных, выполняется процесс **`Hydration`** (маппинг строк `ResultSet` в поля Java-объектов сущности `Patient`).


8. **`PatientRepository`** возвращает полученный список сущностей **`List<Patient>`** в **`PatientService`**.


9. **`PatientService`** обрабатывает список через **`Stream API`**, вызывая метод **`PatientMapper.toDTO(patient)`**.


10. Внутри `PatientMapper` данные копируются в `PatientResponseDTO`. При возникновении исключения (например, RuntimeException из-за несоответствия типов данных) выполнение прерывается, и управление передается в `GlobalExceptionHandler`. Данный компонент перехватывает исключение, формирует структуру ошибки и возвращает клиенту `500 INTERNAL SERVER ERROR`.


11. После успешной трансформации всех элементов **`PatientService`** возвращает итоговый **`List<PatientResponseDTO>`** в **`PatientController`**.


12. **`PatientController`** формирует и возвращает объект **`ResponseEntity`** обратно в **`DispatcherServlet`**.


13. **`DispatcherServlet`** передает список объектов в библиотеку **`Jackson`** для выполнения сериализации.


14. **`Jackson`** возвращает готовую JSON-строку.


15. **`DispatcherServlet`** записывает в тело ответа полученную JSON-строку и отправляет клиенту со статусом `200 OK`.

![GET.patients.svg](..%2FDiagrams%2FGET.patients.svg)