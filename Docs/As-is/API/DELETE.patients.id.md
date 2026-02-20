# Endpoint `DELETE /patients/{id}`

### Общая информация
**Назначение:** Метод предназначен для удалении данных существующего пациента в системе.

**Метод:** `DELETE`

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
| Authorization | string | да | Токен доступа (на уровне сервиса нет валидации) | Bearer <token> | - |
</details> 

---

## Примеры запроса и ответа

Запрос:
~~~ 
curl -X DELETE "/patients/{id}" \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
~~~

Ответ:
~~~
Response code : 200 OK
Response body (json):
{
}
~~~


---
## Алгоритм работы

1. `patient-service` через `PatientController` получает входящий запрос `DELETE  /patients/{id}` 


2. `PatientService` через `PatientRepository` удаляет данные в базе данных `patient_db` из таблицы `patient`. Если запись в базе данных не найдена по `id` - сервис возвращает HTTP статус-код `404 NOT FOUND`.


3. `Patient Service` возвращает ответ `200 OK` на клиент. 

![DELETE.patients.id.svg](..%2FDiagrams%2FDELETE.patients.id.svg)