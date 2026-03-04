# CreateBillingAccount (gRPC)

## Общая информация

**Назначение**: Метод предназначен для создания нового финансового счета для пациента. Метод вызывается сервисом `patient-service`.

**Сервис:** `billing.BillingService`

**Метод:** `CreateBillingAccount`

**Тип вызова:** `Unary`

---


## Протокол вызова

```protobuf
rpc CreateBillingAccount (BillingRequest) returns (BillingResponse);

```
---
## Входные и выходные параметры
<details>
<summary><b><font color="#2196F3">Входные параметры</font></b></summary>
Metadata:
Генерируются автоматически

| Параметр     | Обязательность | Описание | Значение (по умолчанию) |
|:-------------| :---: | :--- |:------------------------|
| `content-type` | Да | Определяет протокол взаимодействия | application/grpc        |
| `user-agent  ` | Нет | Идентификатор клиента (библиотеки) | grpc-java-netty/1.69.0  |
| `te`           | Да | Тип передачи (ожидание трейлеров) | trailers                |

Body:

| Параметр    | Тип данных | Обязательность | Описание | Значение/пример| Маппинг с db |
|-------------|------------|----------------| --- | - | - |
| `patientId` | string     | Да             | UUID пациента из системы Patient Service | 550e8400-e29b-41d4-a716-446655440000 | Не мапится (Mock) |
| `name`      | string     | Да             | Полное имя пациента для привязки к счету | Иван Иванов | Не мапится (Mock) |
| `email`     | string     | Да             | Контактный email для финансовых уведомлений | ivanov@example.com | Не мапится (Mock) |
</details>

<details>
<summary><b><font color="#2196F3">Выходные параметры</font></b></summary>
Trailers:

| Параметр       | Тип данных | Описание | Код / Значение |
|:---------------|:----------:| :--- |:---------------|
| `grpc-status `   |  Trailer   | Основной результат операции | 0 OK           |
| `grpc-message `  |  Trailer   | Сообщение об ошибке | пустая строка  |

Body:

| Параметр    | Тип данных | Обязательность                             | Описание                                           | Значение/пример| Маппинг с db |
|-------------|------------|--------------------------------------------|----------------------------------------------------| - | - |
| `accountId` | string | Да                                         |  Уникальный номер созданного счета (ID в биллинге) | 12345 | Не мапится (Mock) |
| `status` | string | Да | Текущий статус аккаунта (напр., `ACTIVE`) | ACTIVE | Не мапится (Mock) |
</details> 

---

## Примеры запроса и ответа

Запрос:

```json
Metadata:
content-type: application/grpc
user-agent: grpc-java-netty/1.69.0
te: trailers
Request Body (Protobuf):
{
  "patientId": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Иван Иванов",
  "email": "ivanov@example.com"
}
```

Ответ:
```json
Response Status: 0 OK 
Response Body (Protobuf):
{
  "accountId": "12345",
  "status": "ACTIVE"
}
Response Trailers:
grpc-status: 0
grpc-message: ""
```
<details>
<summary><b><font color="#2196F3">Варианты возвращаемых gRPC статус кодов</font></b></summary>


| Код    | Статус | Описание |
|--------| --- | --- |
| **0**  | `OK` | Счет успешно создан. |
| **13** | `INTERNAL` | Ошибка десериализации бинарного потока. |
| **14** | `UNAVAILABLE` | Сервис биллинга временно недоступен. |
</details> 

---

## Алгоритм работы

1. `patient-service` через `BillingServiceGrpcClient` инициирует gRPC-вызов. `BillingServiceGrpcClient` формирует объект `BillingRequest` через `BillingRequest.newBuilder().setPatientId(patientId).setName(name).setEmail(email).build()` и вызывает `blockingStub.createBillingAccount(request)`. Соединение установлено по `plaintext` (без TLS) через `ManagedChannelBuilder.usePlaintext()`. Если `billing-service` недоступен — клиентская библиотека выбрасывает `StatusRuntimeException` со статусом `14 UNAVAILABLE`.


2. `grpc-netty-shaded` на стороне `billing-service` принимает входящий TCP-пакет, извлекает HTTP/2 фреймы и передаёт их в стек обработки gRPC.


3. `protobuf-java` выполняет десериализацию бинарных данных из тела запроса в Java-объект `BillingRequest`. Если структура данных повреждена — возвращается статус-код `13 INTERNAL`.


4. `grpc-spring-boot-starter` находит компонент `BillingGrpcService` помеченный аннотацией `@GrpcService`и передаёт управление методу `createBillingAccount(billingRequest, responseObserver)`.


5. `BillingGrpcService` формирует объект `BillingResponse` через `BillingResponse.newBuilder().setAccountId("12345").setStatus("ACTIVE").build()`.


>**Важно:** Текущая реализация является заглушкой (mock). Сохранение данных в БД и реальная генерация accountId не выполняются.


6. `responseObserver.onNext(response)` передаёт объект `BillingResponse` в gRPC-инфраструктуру — `protobuf-java` сериализует его в бинарный формат и отправляет обратно в `patient-service`.


7. `responseObserver.onCompleted()` закрывает стрим. `patient-service` получает статус-код `0 OK` и объект `BillingResponse`.


8. `BillingServiceGrpcClient` получает ответ и возвращает объект `BillingResponse` в `PatientService`.

![CreateBillingAccount.svg](..%2FDiagrams%2FCreateBillingAccount.svg)

---
## Логирование

| Шаг в алгоритме       | Уровень | Класс                      | Сообщение                                   |
|:----------------------|:--------|:---------------------------|:--------------------------------------------|
| 4                     | INFO    | BillingGrpcService     |createBillingAccount request received {billingRequest}|
| 8                     | INFO    | BillingServiceGrpcClient                           | Received response from billing service via GRPC: {response}|
| При старте приложения |INFO|BillingServiceGrpcClient|Connecting to Billing Service GRPC service at {address}:{port}|