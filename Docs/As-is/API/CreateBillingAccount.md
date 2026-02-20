# CreateBillingAccount (gRPC)

## Общая информация

**Назначение**: Метод предназначен для создания нового финансового счета для пациента. Метод вызывается сервисом `patient-service`.

**Сервис:** `billing.BillingService`

**Метод:** `CreateBillingAccount`

**Тип вызова:** Unary

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

## Примеры зампроса и ответа

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
1. `patient-service` инициирует gRPC-вызов к `billing-service`. Если `billing-service` выключен или порт закрыт, клиентская библиотека сразу возвращает статус-код `14 UNAVAILABLE`. Дальнейшие шаги не выполняются.


2. Библиотека `grpc-netty-shaded` принимает входящий TCP-пакет. HTTP/2 фреймы извлекаются и передаются в стек обработки gRPC.


3. Библиотека `protobuf-java` выполняет десериализацию бинарных данных из тела запроса в Java-объект `BillingRequest`. Если структура данных повреждена, возвращается статус-код `13 INTERNAL`.


4. `grpc-spring-boot-starter` находит компонент, помеченный аннотацией `@GrpcService`, и передает управление методу `createBillingAccount` класса `BillingGrpcService`.


5. С помощью `SLF4J` выполняется запись в лог уровня INFO: `createBillingAccount request received {данные_запроса}`. В лог попадают значения полей `patientId`, `name` и `email`, полученные из `billingRequest.toString()`.


6. Вызывается метод `BillingResponse.newBuilder()`, который создает экземпляр класса `Builder` для конструирования ответного сообщения.


7. В объекте `Builder` последовательно вызываются методы `setAccountId("12345")` и `setStatus("ACTIVE")`. После этого вызывается метод `build()`, который возвращает готовый неизменяемый объект `BillingResponse`.
 

8. Метод `responseObserver.onNext(response)` передает объект `BillingResponse` обратно в gRPC-инфраструктуру, где `protobuf-java` выполняет его сериализацию в бинарный формат.


9. Метод `responseObserver.onCompleted()` закрывает стрим.`patient-service`получает статус-код `0 OK`.

> **Важно**: Текущая реализация сервиса является заглушкой (mock). Сохранение данных в базу данных и реальная генерация ID аккаунта не выполняются. 


![CreateBillingAccount.svg](..%2FDiagrams%2FCreateBillingAccount.svg)