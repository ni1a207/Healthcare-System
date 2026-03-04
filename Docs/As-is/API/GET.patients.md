# Endpoint `GET /patients`

## Общая информация
**Назначение:** Метод предназначен для получения списка пациентов зарегистрированных в системе.

**Метод:** `GET`

**Внутренний URL:** `http://patient-service:4000/patients`

**Внешний URL:** `http://{gateway-host}/api/patients`

---

## Входные и выходные параметры
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
  </tbody>
</table></details> 


---

## Примеры запроса и ответа

Запрос:
~~~json
curl -X GET "http://{gateway-host}/api/patients"\
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
~~~

Ответ:
~~~json
Response code : 200 OK
Response body (json):
[
  {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "address": "123 Main St, Springfield",
    "dateOfBirth": "1985-06-15"
  },
  {
    "id": "6ff9700a-040a-4119-a7a9-f6662122e423",
    "name": "Иван Иванов",
    "email": "ivan@mail.com",
    "address": "г. Москва, ул. Ленина",
    "dateOfBirth": "1990-05-15"
  }
]
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
      <td>JSON массив пациентов</td>
      <td>Список пациентов успешно получен. Если пациентов нет — возвращается пустой массив [].</td>
    </tr>
    <tr>
      <td>500</td>
      <td>INTERNAL SERVER ERROR</td>
      <td>-</td>
      <td>Критический сбой — недоступность БД, ошибка соединения или непредвиденная системная ошибка.</td>
    </tr>
  </tbody>
</table>
</details>

---
## Алгоритм работы

1. `patient-service` через `DispatcherServlet` принимает входящий HTTP-запрос `GET /patients`.


2. `DispatcherServlet` вызывает метод `getPatients()` в `PatientController`.


3. `PatientController` вызывает метод `getPatients()` в `PatientService`.


4. `PatientService` вызывает метод `findAll()` в `PatientRepository`. `PatientRepository` делегирует вызов реализации `SimpleJpaRepository` (Spring Data JPA Proxy).


5. `Hibernate` формирует и выполняет SQL-запрос к таблице `patient` базы данных `patient_db`: `SELECT * FROM patient`. Для выполнения запроса `Hibernate` запрашивает соединение у пула `HikariCP`, который передаёт запрос через JDBC Driver в базу данных. Если соединение недоступно или база данных недоступна — выбрасывается исключение, которое пробрасывается по цепочке через `PatientRepository` → `PatientService` → `PatientController` — Spring Boot возвращает клиенту HTTP cтатус-код `500 INTERNAL SERVER ERROR`.


6. `patient_db` выполняет запрос и возвращает `ResultSet`.


7. `Hibernate` выполняет `Hydration` — маппинг строк `ResultSet` в поля Java-объектов сущности `Patient`. `PatientRepository` возвращает `List<Patient>` в `PatientService`.


8. `PatientService` обрабатывает список через Stream API, для каждого элемента вызывая `PatientMapper.toDTO(patient)`. Внутри `PatientMapper` поля сущности `Patient` копируются в объект `PatientResponseDTO`. Если в процессе маппинга возникает непредвиденное исключение — оно пробрасывается по цепочке через `PatientService` → `PatientController` — Spring Boot возвращает клиенту HTTP cтатус-код `500 INTERNAL SERVER ERROR`.


9. `PatientService` возвращает итоговый `List<PatientResponseDTO>` в `PatientController`.


10. `PatientController` оборачивает список в `ResponseEntity.ok().body(patients)` и возвращает в `DispatcherServlet`.


11. `Jackson` выполняет сериализацию `List<PatientResponseDTO>` в JSON.


12. `DispatcherServlet` записывает JSON в тело ответа и отправляет клиенту со статусом `200 OK`.

![GET.patients.svg](..%2FDiagrams%2FGET.patients.svg)

---
## Логирование

| Шаг в алгоритме       | Уровень | Класс | Сообщение                                   |
|:----------------------|:--------|:----|:--------------------------------------------|
|-|-|-| Логирование в данном методе не предусмотрено|