# Endpoint `DELETE /patients/{id}`

## Общая информация
**Назначение:** Метод предназначен для удалении данных существующего пациента в системе.

**Метод:** `DELETE`

**Внутренний URL:** `http://patient-service:4000/patients/{id}`

**Внешний URL:** `http://{gateway-host}/api/patients/{id}`

---

## Входные и выходные параметры
<details>
<summary><b><font color="#2196F3">Входные параметры</font></b></summary>

Path:

| Параметр | Тип данных | Обязательность | Описание                          | Значение/пример  | Маппинг с patient-service-db |
|:---------|:-----------|:---------------|:----------------------------------|:-----------------|:-----------------------------|
| id       | UUID       |да| Уникальный идентификатор пациента | 123e4567-e89b-12d3-a456-426614174000 | patient.id                   |


Header:

| Параметр  | Тип данных | Обязательность | Описание                                        | Значение/пример  | patient-service-db |
|:------|:-----------|:---------------|:------------------------------------------------|:-----------------|:---------------------|
| Authorization | string | да | Токен доступа (на уровне сервиса нет валидации) | Bearer <token> | - |
</details> 

<details>
<summary><span style="color: #2E86C1;"><b>Выходные параметры</b></span></summary>

Тело ответа отсутствует — `ResponseEntity<Void>`. Результат операции передаётся через HTTP статус-код.

</details>

---

## Примеры запроса и ответа

Запрос:
~~~json
curl -X DELETE "http://{gateway-host}/api/patients/{id}" \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." 
~~~

Ответ:
~~~json
Response code : 204 NO CONTENT
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
      <td>204</td>
      <td>NO CONTENT</td>
      <td>-</td>
      <td>Пациент успешно удалён. Если пациент с указанным id не найден — также возвращается 204, удаление не выполняется без ошибки.</td>
    </tr>
    <tr>
      <td>500</td>
      <td>INTERNAL SERVER ERROR</td>
      <td>-</td>
      <td>Критический сбой — недоступность БД или непредвиденная системная ошибка.</td>
    </tr>
  </tbody>
</table>
</details>

---
## Алгоритм работы

1. `patient-service` через `DispatcherServlet` принимает входящий HTTP-запрос `DELETE /patients/{id}`.


2. `DispatcherServlet` вызывает метод `deletePatient(id)` в `PatientController`. Spring извлекает значение `id` из пути через аннотацию `@PathVariable` и преобразует строку в объект `UUID`.


3. `PatientController` вызывает метод `deletePatient(id)` в `PatientService`.


4. `PatientService` вызывает метод `deleteById(id)` в `PatientRepository`. `PatientRepository` делегирует вызов реализации `SimpleJpaRepository` (Spring Data JPA Proxy).


5. `Hibernate` формирует и выполняет SQL-запрос к таблице `patient` базы данных `patient-service-db`: `DELETE FROM patient WHERE id = ?`. Для выполнения запроса `Hibernate` запрашивает соединение у пула `HikariCP`, который передаёт запрос через `JDBC Driver` в базу данных. Если соединение недоступно или база данных недоступна — выбрасывается исключение, которое пробрасывается по цепочке через `PatientRepository` → `PatientService` → `PatientController` — Spring Boot возвращает клиенту HTTP статус код `500 INTERNAL SERVER ERROR`.


>**Важно:** `deleteById()` не бросает исключение, если запись с указанным `id` не найдена — SQL-запрос выполняется, затронутых строк 0, метод завершается без ошибки.


6. `patient-service-db` выполняет запрос и возвращает количество затронутых строк.


7. `PatientService` завершает выполнение (метод void). `PatientController` возвращает `DispatcherServlet` ответ `ResponseEntity.noContent().build()`.


8. `DispatcherServlet` отправляет клиенту `204 NO CONTENT` без тела ответа.

![DELETE.patients.id.svg](..%2FDiagrams%2FDELETE.patients.id.svg)

---
## Логирование

| Шаг в алгоритме       | Уровень | Класс | Сообщение                                   |
|:----------------------|:--------|:----|:--------------------------------------------|
|-|-|-| Логирование в данном методе не предусмотрено|