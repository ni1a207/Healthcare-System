# Система управления медицинскими данными Healthcare System

## Назначение и цель системы
Данное приложение представляет собой медицинскую ERP-систему на основе микросервисной архитектуры для автоматизации процессов в медицинской клинике. Основная цель — управление данными пациентов, и автоматическое выставление счетов за медицинские услуги.

## Архитектура системы (C4 Container)
![HealthcareC4.svg](Diagrams%2FHealthcareC4.svg)

---
## Сквозные сценарии взаимодействия с системой

<details>
<summary><b><font color="#2196F3">Аутентификация</font></b></summary>
Процесс проверки личности пользователя и предоставления ему прав доступа к защищенным ресурсам системы. 

1.  Пользователь вводит логин и пароль в форму для ввода, нажимает кнопку "Войти". `frontend` отправляет запрос [POST {host}/auth/login](API%2FPOST.login.md) .
2.  Запрос проксируется через `api-gateway` в `auth-service`.
3.  `auth-service` валидирует входные параметры.
4.  `auth-service` отправляет SQL-запрос к `auth-db`: выборка из таблицы `users`, где `username` совпадает со значением переменной `username`, которую передал пользователь в JSON-объекте.
5.  `auth-db` в таблице `patient` находит запись и возвращает данные.
6. `auth-service` сравнивает присланный пароль с хэшем из базы данных с помощью алгоритма `BCrypt`. 
7.  `auth-service` генерирует `JWT`.
8.  `auth-service` возвращает ответ на запрос в `api-gateway` `200 OK + JWT`.
9. `api-gateway` прокcирует ответ на `frontend`.
10. `frontend` получает ответ, сохраняет в localStorage `JWT` и перенаправляет на главную страницу системы.

![Auth.svg](Diagrams%2FAuth.svg)
</details>

---
<details>

<summary><b><font color="#2196F3">Авторизация и маршрутизация</font></b></summary>
Проверка прав доступа к запрашиваемому ресурсу.

1. Пользователь делает запрос к системе. `frontend` отправляет все запросы на сервер через `api-gateway`.
2. `api-gateway` анализирует URL запроса и ищет совпадение в `application.yml`. Если для найденного маршрута в списке `filters` указан - `JwtValidation`, активируется проверка `JWT`.
3. `api-gateway` проверяет заголовок `Authorization`.
4. `api-gateway` отправляет запрос [GET {host}/validate](API%2FGET.validate.md) с заголовком `Authorization`  в `auth-service`.
5. `auth-service` валидирует `JWT`.
6. `auth-service` при успешной валидации возвращает ответ `200 OK`.
7. `api-gateway` получает ответ `200 OK`, прокcирует запрос с заголовком `Authorization` в `target-service` (целевой сервис).
8.  `target-service` выполняет бизнес логику запроса. 
9. `target-service` возвращает ответ.
10. `api-gateway` получает ответ от `target-service` и проксирует ответ на `frontend`.
11. `frontend` обрабатывает данные и отображает ответ пользователю.
![Auth&routing.svg](Diagrams%2FAuth%26routing.svg)

</details> 

---

<details>
<summary><b><font color="#2196F3">Регистрация пациента и создание счета</font></b></summary>

Процесс создания новой учетной записи пациента в системе. Выполняется администратором клиники.

1. Пользователь вводит данные в форму и нажимает кнопку "Зарегистрировать". `frontend` отправляет запрос [POST {host}/patients](API%2FPOST.patients.md).
2. Запрос поступает в `api-gateway`, который инициирует валидацию `JWT`, отправляет запрос [GET {host}/validate](API%2FGET.validate.md) в `auth-service`.
3. `api-gateway` проксирует запрос в `patient-service`.
4. `patient-service` получает запрос, валидирует входные параметры. 
5. `patient-service` ищет записи с совпадением по полю `email` в `patient-db`, если таких записей в базе данных нет — генерирует UUID и делает новую запись в базу данных.
6. `patient-service` передает `patientId`, `name` и `email` в адаптер `BillingServiceGrpcClient`.
7. `BillingServiceGrpcClient` упаковывает данные в `BillingRequest` (Protobuf) и выполняет блокирующий вызов метода `CreateBillingAccount` в `billing-service`.
8. `billing-service` принимает gRPC-запрос, логирует входящие данные, возвращает объект `BillingResponse` (с захардкоженными значениями accountId: "12345" и status: "ACTIVE").
9. `patient-service` создает объект `PatientEvent`, `event_type: "PATIENT_CREATED"`.
10. `patient-service` отправляет объект `PatientEvent` в Kafka-топик `patient`.
11. `patient-service` возвращает ответ на `frontend` через `api-gateway` статус-код `200 OK + JSON` с данными зарегистрированного пациента.
![Registration.svg](Diagrams%2FRegistration.svg)
</details>

---

<details>
<summary><b><font color="#2196F3">Просмотр зарегистрированных пациентов</font></b></summary>

1. Пользователь нажимает кнопку "Все пациенты". `frontend` отправляет запрос [GET /patients](API%2FGET.patients.md).
2. Запрос поступает в `api-gateway`, который инициирует валидацию `JWT`, отправляет запрос [GET {host}/validate](API%2FGET.validate.md) в `auth-service`.
3. `api-gateway` проксирует запрос в `patient-service`.
4. `patient-service` достает все записи из базы данных `patient-db` таблицы `patient`.
5. `patient-service` возвращает ответ на `frontend` через `api-gateway` статус-код `200 OK + JSON` с массивом данных зарегистрированных пациентов.
![ViewPatients.svg](Diagrams%2FViewPatients.svg)

</details>

---
<details>
<summary><b><font color="#2196F3">Изменение лиичных данных пациента</font></b></summary>
...
</details>

---

<details>
<summary><b><font color="#2196F3">Удаление пациента </font></b></summary>
...
</details>

---
## Реестр сервисов и инфраструктурных компонентов
| Сервис                                               | БД                                        | API                        | Описание                             |
|:-----------------------------------------------------|:------------------------------------------|:---------------------------|:-------------------------------------|
| [api-gateway](Services%2Fapi-gateway.md)             | -                                         | REST                       | Точка входа, маршрутизация запросов  |
| [auth-service](Services%2Fauth-service.md)           | PostgreSQL<br/>[auth_db](DB%2Fauth_db.md) | REST                       | Управление пользователями и сессиями |
| [patient-service](Services%2Fpatient-service.md)     | PostgreSQL <br/>[patient_db](DB%2Fpatient_db.md)                | REST, gRPC, Kafka producer | Мастер-система для данных пациентов  |
| [billing-service](Services%2Fbilling-service.md)     | -                                         | gRPC, REST                 | Финансовый модуль                    |
| [analytics-service](Services%2Fanalytics-service.md) | -                                         | Kafka  consumer            | Сбор статистики                      |
| [Kafka](Services%2FKafka.md)                         | -                                         | Kafka                      | Распределенный лог сообщений         |

---
## Матрица трассировки требований

| №     | Бизнес-функция                             | Релиз | Связанные сервисы                                                                                           | Базы данных                                     | API                                                                                                                  |
|:------|:-------------------------------------------| :--- |:------------------------------------------------------------------------------------------------------------|:------------------------------------------------|:---------------------------------------------------------------------------------------------------------------------|
| **1** | Аутентификация и авторизация пользователя  | **MVP** | [api-gateway](Services%2Fapi-gateway.md)<br/>[auth-service](Services%2Fauth-service.md)                     | [auth_db](DB%2Fauth_db.md)                      | [POST /login](API%2FPOST.login.md) <br/> [GET /validate](API%2FGET.validate.md)                                      |
| **2** | Регистрация нового пациента                | **MVP** | [patient-service](Services%2Fpatient-service.md)  <br/> [billing-service](Services%2Fbilling-service.md)    | [patient_db](DB%2Fpatient_db.md)                | [POST /patients](API%2FPOST.patients.md) <br/> [gRPC createBilling](API%2FgRPC.createBilling.md) <br/> kafkaProducer |
| **3** | Просмотр всех зарегистрированных пациентов | **MVP** | [patient-service](Services%2Fpatient-service.md)                                                            | [patient_db](DB%2Fpatient_db.md)                                  | [GET /patients](API%2FGET.patients.md)                                                                               |
| **4** | Изменение данных пациента                  | **MVP** | [patient-service](Services%2Fpatient-service.md)                                                            |[patient_db](DB%2Fpatient_db.md)                                  | [PUT /patients/{id}](API%2FPUT.patients.id.md)                                                                       |
| **5** | Удаление пациента                          | **MVP** | [patient-service](Services%2Fpatient-service.md)                                                            | [patient_db](DB%2Fpatient_db.md)                                   | [DELETE /patients/{id}](API%2FDELETE.patients.id.md)                                                                 |

---
