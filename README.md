# Блокировки платежей (API + БД)

В продукте реализован сервис по отправке платежей юридическим лицам. Но периодически возникают ситуации, когда платежи надо приостановить, потому что клиент ненадежен (есть подозрение на мошенничество и нужно время разобраться), либо клиент предоставил не те реквизиты для перечисления и платежи отбиваются банком клиента (нужно время на выяснение верных реквизитов).

### Запрос бизнеса следующий:

- нужна возможность заблокировать платежи конкретного клиента;
- нужна возможность разблокировать платежи клиента;
- нужна возможность проверить, заблокирован ли клиент;
- нужна возможность отличать блокировки мошенников от блокировок добропорядочных клиентов.

### Решение по реализации этого запроса бизнеса:

- Подготовлена спецификация endpoint’ов в формате OpenAPI для реализации указынных выше задач;
- Разработана структура хранения нужной информации в БД.

---

## Построение концептуально-инфологической модели

### 1. Формирование сущностей

| Сущность            | Описание                          |
|---------------------|---------------------------------|
| `client`           | Хранение информации о клиентах  |
| `client_block`     | Хранение текущих блокировок     |
| `client_block_history` | Хранение истории блокировок и разблокировок |

### 2. Определение атрибутов и требований к ним для каждой сущности

#### `client`
| Название атрибута | Тип  | Описание                     |
|-------------------|------|-----------------------------|
| `id` (PK)        | UUID | Идентификатор клиента       |

В качестве первичного ключа выбран атрибут `id`, поскольку он однозначно идентифицирует клиента банка.

#### `client_block`
| Название атрибута | Тип         | Описание                                        |
|-------------------|------------|------------------------------------------------|
| `id` (PK)        | UUID       | Идентификатор блокировки                        |
| `client_id` (FK) | UUID       | Идентификатор клиента                          |
| `reason`        | ENUM       | Причина блокировки (`scam`, `incorrect_details`) |
| `comment`       | TEXT       | Комментарий                                    |
| `blocked_at`    | TIMESTAMP  | Дата и время блокировки                        |

Атрибут `client_id` является внешним ключом сущности `client_block`.

#### `client_block_history`
| Название атрибута | Тип         | Описание                                        |
|-------------------|------------|------------------------------------------------|
| `id` (PK)        | UUID       | Идентификатор записи                          |
| `client_id` (FK) | UUID       | Идентификатор клиента                         |
| `action`        | ENUM       | Действие (`block`, `unblock`)                  |
| `reason`        | ENUM       | Причина блокировки (`scam`, `incorrect_details`) |
| `comment`       | TEXT       | Комментарий                                   |
| `created_at`    | TIMESTAMP  | Дата и время блокировки                       |

### 3. Выявление связей между сущностями
1. `client` - `client_block`: один к одному  
   - Один клиент может иметь ноль или одну запись о блокировке.  
   - Каждая запись о блокировке относится только к одному клиенту.  
2. `client` - `client_block_history`: один ко многим  
   - Один клиент может иметь несколько записей в истории блокировок.  
   - Каждая запись в истории относится только к одному клиенту.  

---
### 4. ER-диаграмма
![diagram](images/ER.jpg) 

---

### 5. Endpoint’ы

Файл спецификации OpenAPI в формате YAML находится в [репозитории этого проекта](openapi.yaml). 

Интерактивная Swagger-документация (OpenAPI 3.0) расположена на [GitHub Pages данного репозитория](https://medentseva-elizaveta.github.io/Blocking_payments/).


### 5.1. Блокировка клиента `POST /client/{id}/block`
**Логика работы:**
1. Проверка существования клиента с `id`.
2. Проверка отсутствия активной блокировки в `client_block`.
3. Создание записи в `client_block`:
   - `id` - идентификатор записи
   - `client_id` - идентификатор клиента
   - `reason` - причина (`scam`, `incorrect_details`)
   - `comment` - комментарий сотрудника
   - `blocked_at` - дата и время создания записи
4. Создание записи в `client_block_history`:
   - `id` - идентификатор записи
   - `client_id` - идентификатор клиента
   - `action` - действие (`block`)
   - `reason` - причина (`scam`, `incorrect_details`)
   - `comment` - комментарий сотрудника
   - `created_at` - дата и время создания записи 
5. Возврат статуса **`201 (Created)`** при успешном выполнении запроса.

![diagram](images/Block.png) 

---

### 5.2. Разблокировка клиента `POST /client/{id}/unblock`
**Логика работы:**
1. Проверка существования клиента с `id`.
2. Проверка наличия активной блокировки в `client_block`.
3. Удаление записи из `client_block` с `client_id`=`id`.
4. Создание записи в `client_block_history` (`action`=`unblock`).
5. Возврат статуса **`200 (OK)`** при успешном выполнении запроса.

![diagram](images/Unblock.png) 

---

### 5.3. Проверка статуса `GET /client/{id}/status`
**Логика работы:**
1. Проверка существования клиента с `id`.
2. Поиск активной блокировки в `client_block`.
3. Если запись найдена, возвращается **`200 (OK)`** с телом ответа:
   ```json
   {
     "blocked": true,
     "reason": "scam"
   }

![diagram](images/Status.png) 

---

### 5.4. История блокировок `GET /client/{id}/block/history`
**Логика работы:**
1. Проверка существования клиента с идентификатором `id`.
2. Получение записей из таблицы `client_block_history` с `client_id`=`id`, `reason`= `scam/incorrect_details`, в зависимости от переданного query-параметра и пагинацией
3. Возврат статуса **`200 (OK)`** с телом ответа, в котором содержится список записей о блокировках.

![diagram](images/History1.png) 
![diagram](images/History2.png) 

---
