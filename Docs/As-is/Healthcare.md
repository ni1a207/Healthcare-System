# Система управления медицинскими данными Healthcare System

## Назначение и цель системы
Данное приложение представляет собой медицинскую ERP-систему на основе микросервисной архитектуры для автоматизации процессов в медицинской клинике. Основная цель — управление данными пациентов, и автоматическое выставление счетов за медицинские услуги.

## Архитектура системы (C4 Container)
![HealthcareC4.svg](Diagrams%2FHealthcareC4.svg)
---
## Бизнес-процессы

<details>
<summary><b><font color="#2196F3">Аутентификация</font></b></summary>

Процесс проверки личности пользователя и предоставления ему прав доступа к защищенным ресурсам системы. 

1.  Пользователь вводит логин и пароль в форму для ввода, нажимает кнопку "Войти". `frontend` отправляет HTTP-запрос [POST api/auth/login](API%2FPOST.login.md) с телом запроса (JSON).
2.  Запрос проксируется через `api-gateway` в `auth-service`.
3. `auth-service` отправляет SQL-запрос к `auth-db`: выборка из таблицы `users`, где `email` совпадает со значением переменной `email`, которую передал пользователь в JSON-объекте.
4.  `auth-db` в таблице `users` находит запись и возвращает данные.
5.  `auth-service` сравнивает присланный пароль с хэшем из базы данных с помощью алгоритма `BCrypt`. 
6.  `auth-service` генерирует `JWT`.
7.  `auth-service` возвращает ответ на запрос в `api-gateway` `200 OK (JSON)`.
8.  `api-gateway` прокcирует ответ на `frontend`.
9. `frontend` получает ответ, сохраняет в localStorage `JWT` и перенаправляет на главную страницу системы.

![Auth.svg](Diagrams%2FAuth.svg)
</details>

---
<details>

<summary><b><font color="#2196F3">Авторизация и маршрутизация</font></b></summary>

Процесс проверки прав доступа к запрашиваемому ресурсу.

1. Пользователь делает запрос к системе. `frontend` отправляет все запросы на сервер через `api-gateway` с заголовком `Authorization: Bearer <token>`.
2. `api-gateway` анализирует URL запроса и ищет совпадение в `application.yml`. Если для найденного маршрута в списке `filters` указан - `JwtValidation`, активируется проверка `JWT`.
3. `api-gateway` проверяет заголовок `Authorization`.
4. `api-gateway` отправляет HTTP-запрос [GET /validate](API%2FGET.validate.md) с заголовком `Authorization`  в `auth-service`.
5. `auth-service` валидирует `JWT`.
6. `auth-service` при успешной валидации возвращает ответ `200 OK`.
7. `api-gateway` получает ответ `200 OK`, прокcирует запрос с заголовком `Authorization` в `target-service` (целевой сервис).
8.  `target-service` выполняет бизнес логику запроса. 
9. `target-service` возвращает ответ.
10. `api-gateway` получает ответ от `target-service` и проксирует ответ на `frontend`.
![Auth&routing.svg](Diagrams%2FAuth%26routing.svg)

</details> 

---

<details>
<summary><b><font color="#2196F3">Регистрация пациента и создание счета</font></b></summary>

Процесс создания новой учетной записи пациента в системе. Выполняется администратором клиники.

1. Пользователь вводит данные в форму и нажимает кнопку "Зарегистрировать". `frontend` отправляет HTTP-запрос [POST api/patients](API%2FPOST.patients.md) с заголовком `Authorization: Bearer <token>` и телом запроса (JSON).
2. Запрос поступает в `api-gateway`, который инициирует валидацию `JWT`, отправляет HTTP-запрос [GET /validate](API%2FGET.validate.md) в `auth-service`.
3. `api-gateway` проксирует запрос в `patient-service`.
4. `patient-service` получает запрос, валидирует входные параметры. 
5. `patient-service` ищет записи с совпадением по полю `email` в `patient-db`, если таких записей в базе данных нет — делает новую запись в базу данных и генерируется UUID.
6. `patient-service` передает `patientId`, `name` и `email` в адаптер `BillingServiceGrpcClient`.
7. `BillingServiceGrpcClient` упаковывает данные в `BillingRequest` (Protobuf) и выполняет блокирующий gRPC-вызов метода [CreateBillingAccount](API%2FCreateBillingAccount.md) в `billing-service`.
8. `billing-service` принимает gRPC-запрос, логирует входящие данные, возвращает объект `BillingResponse` (с захардкоженными значениями accountId: "12345" и status: "ACTIVE").
9. `patient-service` создает объект `PatientEvent`, `event_type: "PATIENT_CREATED"`.
10. `patient-service` отправляет событие [PatientEvent](API%2FPatientEventProducer.md) в Kafka-топик `patient`.
11. `patient-service` возвращает ответ `200 OK (JSON)` с данными зарегистрированного пациента.
12. `api-gateway` проксирует ответ на `frontend`.

![Registration.svg](Diagrams%2FRegistration.svg)
</details>

---

<details>
<summary><b><font color="#2196F3">Просмотр зарегистрированных пациентов</font></b></summary>

Процесс просмотра учетных записей пациентов в системе. Выполняется администратором клиники.

1. Пользователь нажимает кнопку "Все пациенты". `frontend` отправляет HTTP-запрос [GET api/patients](API%2FGET.patients.md) с заголовком `Authorization: Bearer <token>`.
2. Запрос поступает в `api-gateway`, который инициирует валидацию `JWT`, отправляет HTTP-запрос [GET /validate](API%2FGET.validate.md) в `auth-service`.
3. `api-gateway` проксирует запрос в `patient-service`.
4. `patient-service` достает все записи из базы данных `patient-db` таблицы `patient`.
5. `patient-service` возвращает ответ `200 OK (JSON)` с массивом данных пациентов.
6. `api-gateway` проксирует ответ на `frontend`.

![ViewPatients.svg](Diagrams%2FViewPatients.svg)

</details>

---
<details>
<summary><b><font color="#2196F3">Изменение личных данных пациента</font></b></summary>

Процесс обновления\изменения учетной записи пациента в системе. Выполняется администратором клиники.

1. Пользователь вводит данные в форму, нажимает кнопку "Сохранить". `frontend` отправляет HTTP-запрос [PUT /api/patients/{id}](API%2FPUT.patients.id.md) с заголовком `Authorization: Bearer <token>` и телом запроса (JSON).
2. Запрос поступает в `api-gateway`, который инициирует валидацию `JWT`, отправляет запрос [GET /validate](API%2FGET.validate.md) в `auth-service`.
3. `api-gateway` проксирует запрос в `patient-service`.
4. `patient-service` находит запись пациента в `patient_db` по `id`, проверяет уникальность `email`, обновляет поля и сохраняет изменения (UPDATE).
5. `patient-service` возвращает `200 OK (JSON)` с обновлёнными данными пациента.
6. `api-gateway` проксирует ответ на `frontend`.

![UpdatePatient.svg](Diagrams%2FUpdatePatient.svg)
</details>

---

<details>
<summary><b><font color="#2196F3">Удаление пациента </font></b></summary>

Процесс удаления учетной записи пациента из системы. Выполняется администратором клиники.

1. Пользователь в карточке пациента нажимает на кнопку "Удалить". `frontend` отправляет HTTP-запрос [DELETE /api/patients/{id}](API%2FDELETE.patients.id.md) с заголовком `Authorization: Bearer <token>`.
2. Запрос поступает в `api-gateway`, который инициирует валидацию `JWT`, отправляет HTTP-запрос [GET /validate](API%2FGET.validate.md) в `auth-service`.
3. `api-gateway` проксирует запрос в `patient-service`.
4. `patient-service` выполняет удаление записи из `patient_db`.
5. `patient-service` возвращает `204 NO CONTENT`.
6. `api-gateway` проксирует ответ на `frontend`.

![DeletePatient.svg](Diagrams%2FDeletePatient.svg)
</details>

---
## Реестр сервисов и инфраструктурных компонентов
| Сервис                                               | БД                                               | API                            | Описание                             |
|:-----------------------------------------------------|:-------------------------------------------------|:-------------------------------|:-------------------------------------|
| [api-gateway](Services%2Fapi-gateway.md)             | -                                                | REST                           | Точка входа, маршрутизация запросов  |
| [auth-service](Services%2Fauth-service.md)           | PostgreSQL<br/>[auth-db](DB%2Fauth_db.md)        | REST                           | Управление пользователями и сессиями |
| [patient-service](Services%2Fpatient-service.md)     | PostgreSQL <br/>[patient-db](DB%2Fpatient_db.md) | REST, gRPC, Kafka producer     | Мастер-система для данных пациентов  |
| [billing-service](Services%2Fbilling-service.md)     | -                                                | gRPC                           | Финансовый модуль                    |
| [analytics-service](Services%2Fanalytics-service.md) | -                                                | Kafka  consumer                | Сбор статистики                      |
| [kafka](Services%2Fkafka.md)                         | -                                                | TCP (Protobuf), topic: patient | Распределенный лог сообщений         |

---
## Матрица трассировки требований

| №     | Бизнес-процесс                        | Релиз   | Связанные сервисы                                                                                                                                                                                        | Базы данных                      | API                                                                                                                                                                                                                          |
|:------|:--------------------------------------|:--------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:---------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **1** | Аутентификация                        | **MVP** | [api-gateway](Services%2Fapi-gateway.md)<br/>[auth-service](Services%2Fauth-service.md)                                                                                                                  | [auth-db](DB%2Fauth_db.md)       | [POST /login](API%2FPOST.login.md)  <br/> [Алгоритмы API Gateway](API%2FAPIGateway.md)                                                                                                                                       |
| **2** | Авторизация и маршрутизация           | **MVP** | [api-gateway](Services%2Fapi-gateway.md)<br/>[auth-service](Services%2Fauth-service.md)                                                                                                                  | -                                | [GET /validate](API%2FGET.validate.md) <br/> [Алгоритмы API Gateway](API%2FAPIGateway.md)                                                                                                                                    |
| **3** | Регистрация пациента и создание счета | **MVP** |  [api-gateway](Services%2Fapi-gateway.md)<br/>[patient-service](Services%2Fpatient-service.md)  <br/> [billing-service](Services%2Fbilling-service.md)  <br/> [analytics-service (async consumer)](Services%2Fanalytics-service.md) <br/> [kafka](Services%2Fkafka.md) | [patient-db](DB%2Fpatient_db.md) | [POST /patients](API%2FPOST.patients.md) <br/> [CreateBillingAccount](API%2FCreateBillingAccount.md) <br/> [PatientEventProducer](API%2FPatientEventProducer.md) <br/> [PatientEventConsumer](API%2FPatientEventConsumer.md) |
| **4** | Просмотр зарегистрированных пациентов | **MVP** |  [api-gateway](Services%2Fapi-gateway.md)<br/>[patient-service](Services%2Fpatient-service.md)                                                                                                                                                         | [patient-db](DB%2Fpatient_db.md) | [GET /patients](API%2FGET.patients.md)                                                                                                                                                                                       |
| **5** | Изменение личных данных пациента      | **MVP** | [api-gateway](Services%2Fapi-gateway.md)<br/> [patient-service](Services%2Fpatient-service.md)                                                                                                                                                         | [patient-db](DB%2Fpatient_db.md) | [PUT /patients/{id}](API%2FPUT.patients.id.md)                                                                                                                                                                               |
| **6** | Удаление пациента                     | **MVP** | [api-gateway](Services%2Fapi-gateway.md)<br/> [patient-service](Services%2Fpatient-service.md)                                                                                                                                                         | [patient-db](DB%2Fpatient_db.md) | [DELETE /patients/{id}](API%2FDELETE.patients.id.md)                                                                                                                                                                         |

---
