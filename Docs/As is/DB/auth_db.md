![auth_db.png](auth_db.png)



#### Таблица: users

| Название | Обяз. | Тип | Ограничения | Описание | Пример |
|:---|:---|:---|:---|:---|:---|
| **id** | Да | UUID | Primary Key | Идентификатор пользователя | 223e4567-e89b... |
| **email** | Да | VARCHAR(255) | Unique, Not Null | Логин пользователя (Email) | testuser@test.com |
| **password** | Да | VARCHAR(255) | Not Null | Хэш пароля (BCrypt) | $2b$12$7hoR... |
| **role** | Да | VARCHAR(50) | Not Null | Роль пользователя | ADMIN |
