# Система управления медицинскими данными Healthcare System

## Назначение и цель системы
Данное приложение представляет собой медицинскую ERP-систему на основе микросервисной архитектуры для автоматизации процессов в медицинской клинике. Основная цель — управление данными пациентов, и автоматическое выставление счетов за медицинские услуги.

## Архитектура системы (C4 Container)
![HealthcareC4.svg](Diagrams%2FHealthcareC4.svg)

---
## Сквозные сценарии взаимодействия

<details>
<summary><b><font color="#2196F3">Аутентификация</font></b></summary>Процесс проверки личности пользователя и предоставления ему прав доступа к защищенным ресурсам системы. 

#### Подробный алгоритм:
1.  Пользователь вводит логин и пароль в форму для ввода, нажимает кнопку "Войти". `Frontend` отправляет запрос  `POST /api/v1/auth/login`.
2.  Запрос проксируется через `API Gateway` в `Auth Service`.
3.  `Auth Service` валидирует входные параметры, если параметры невалидны - серивис возвращает HTTP статус-код `400 BAD REQUEST`.
4.  `Auth Service` отправляет SQL-запрос к `auth-db`: выборка из таблицы `users`, где `username` совпадает со значением переменной `username`, которую передал пользователь в JSON-объекте. Если в базе данных пользователь не найден - сервис возвращает HTTP статус-код `401 UNAUTHORIZED`.
5.  `Auth Service` сравнивает присланный пароль с хэшем из базы данных с помощью алгоритма `BCrypt`. Если пароль и хэш не совпадают - сервис возвращает HTTP статус-код `401 UNAUTHORIZED`.
6.  `Auth Service` генерирует `JWT`.
7.  `Auth Service` возвращает ответ на запрос в `API Gateway` статус-код `200 OK + JwtResponse`, `API Gateway` прокcирует ответ на `Frontend`.
8.  `Frontend` получает ответ, сохраняет в localStorage `JWT` и перенаправляет на главную страницу системы.
![AuthorizationSQD.svg](Diagrams%2FAuthorizationSQD.svg)
</details>

---
<details>
<summary><b><font color="#2196F3">Авторизация</font></b></summary>

</details>

---

<details>
<summary><b><font color="#2196F3">Регистрация нового пациента</font></b></summary>

Процесс создания новой учетной записи пациента в системе. Выполняется администратором клиники.

#### Подробный алгоритм:
1. Пользователь вводит данные в форму и нажимает кнопку "Зарегистрировать". `Frontend` отправляет запрос `POST /api/v1/patients`. 
2. Запрос поступает в `API Gateway`, который извлекает `JWT` из заголовка, отправляет запрос `GET /api/v1/auth/validate` в `Auth Service` c `JWT` в query параметрах. Если в запросе не было `JWT` - на клиент возвращается HTTP статус-код `401 UNAUTHORIZED`.
   
    2.1. `Auth Service` извлекает `JWT` из query, если параметр отсутствует или пустой - сервис возвращает HTTP статус-код `400 BAD REQUEST`.

    2.2. `Auth Service` валидирует подпись, срок действия, извлекает `username` из токена для поиска записей в auth-db, если подпись невалидна или срок действия истек, или в базе данных не найдена запись - сервис возвращает HTTP статус-код `401  UNAUTHORIZED`.

    2.3. `Auth Service` возвращает HTTP статус-код `200 OK + JSON`.

    2.4. `API Gateway` проксирует запрос в `Patient Service`. Если в реестре сервисов отсутствуют активные инстансы - возвращается HTTP стату-код `503 SERVICE UNAVAILABLE`. Если сервис не ответил в рамках установленного таймаута - возвращается HTTP стату-код `504 GATEWAY TIMEOUT`.
3. `Patient Service` получает запрос, валидирует входные параметры. Если входные параметры невалидны — сервис возвращает HTTP стату-код `400 BAD REQUEST`.

    3.1 `Patient Service` проверяет отсутствие сущности по полю `email` в `patient-db`, если записи не найдены - фиксирует транзакцию в `patient-db` и возвращает `id`(patientId). Если в `patient-db` будет найдена запись с таким же `email` - сервис возвращает HTTP статус-код `400 BAD REQUEST`.

4. `Patient Service` передает `patientId`, `name` и `email` в адаптер `Billing Service Grpc Client`. 

   4.1. `Billing Service Grpc Client` упаковывает данные в `BillingRequest` (Protobuf) и выполняет блокирующий вызов метода `CreateBillingAccoun` в `Billing Service`.

5. `Billing Service` принимает gRPC-запрос, логирует входящие данные (patientId, name, email) и возвращает объект `BillingResponse`(с захардкоженными значениями accountId: "12345" и status: "ACTIVE"). Если `Billing Service` недоступен - сервис возвращает HTTP статус-код `500 INTERNAL SERVER ERROR`.
6. `Patient Service` получает ответ от `Billing Service`, приступает к формированию события для `Kafka`. 

    6.1. `Patient Service` создает объект `PatientEvent` на основе .proto схемы, устанавливает `event_type: "PATIENT_CREATED"`. 

    6.2. `Patient Service` отправляет объект `PatientEvent` в Kafka-топик `patient-events` через выходной канал `patientEventSupplier-out-0`. Если `Kafka` недоступен - сервис возвращает HTTP статус-код `500 INTERNAL SERVER ERROR`.
7. `Patient Service` возвращает ответ на `Frontend` через `API Gateway` - HTTP статус-код `201 CREATED` + JSON с данными созданного профиля.

Примичание: 
1. При недоступности БД в любом шаге алгоритма - сервис возвращает HTTP статус-код `500 INTERNAL SERVER ERROR`.
2. Метод создания пациента защищен аннотацией @Transactional, что гарантирует атомарность всей цепочки действий. В случае возникновения системных ошибок (HTTP 500) на этапах gRPC-запроса к Billing Service или публикации события в Kafka, происходит автоматический откат (rollback) всех изменений в базе данных.

![DataRetrievalSQD.svg](Diagrams%2FDataRetrievalSQD.svg)
</details>

---

<details>
<summary><b><font color="#2196F3">Просмотр зарегистрированных пациентов</font></b></summary>
...
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
## Микросервисы
| Сервис                                            | БД         | API               | Описание                             |
|:--------------------------------------------------|:-----------|:------------------|:-------------------------------------|
| [API Gateway](Services/ApiGateway.md)             | -          | REST              | Точка входа, маршрутизация запросов  |
| [Auth Service](Services/AuthService.md)           | PostgreSQL | REST              | Управление пользователями и сессиями |
| [Patient Service](Services/PatientService.md)     | PostgreSQL | REST, gRPC, Kafka | Мастер-система для данных пациентов  |
| [Billing Service](Services/BillingService.md)     | -          | gRPC, REST        | Финансовый модуль           |
| [Analytics Service](Services/AnalyticsService.md) | -          | Kafka             | Сбор статистики                      |

---
## Матрица трассировки требований

| №     | Бизнес-функция                             | Релиз | Связанные сервисы                                                                                | Базы данных | API                                                      |
|:------|:-------------------------------------------| :--- |:-------------------------------------------------------------------------------------------------| :--- |:---------------------------------------------------------|
| **1** | Авторизация пользователя                   | **MVP** | [API Gateway](Services/ApiGateway.md)<br/>[Auth Service](Services/AuthService.md)                     | [auth-db]() | [POST /api/auth/login]() <br/>[GET /api/auth/validate]() |
| **2** | Регистрация нового пациента                | **MVP** | [Patient Service](Services/PatientService.md)<br/> [Billing Service](Services/BillingService.md) | [patient-db]() | [POST /api/patients]()<br/> [gRPC: createBilling]()      |
| **3** | Просмотр всех зарегистрированных пациентов | **MVP** | [Patient Service](Services/PatientService.md)                                                    | [patient-db]() | [GET /api/patients/]()                                   |
| **4** | Изменение данных пациента                  | **MVP** | [Patient Service](Services/PatientService.md)                                                    | [patient-db]() | [PUT /api/patients/{id}]()                               |
| **5** | Удаление пациента                   | **MVP** | [Patient Service](Services/PatientService.md)                                                    | [patient-db]() | [DELETE /api/patients/{id}]()                            |

---
