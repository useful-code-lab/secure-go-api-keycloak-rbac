# 🔐 Secure Go API Starter — Keycloak OIDC + RBAC + Multipass

Готовая основа для создания **защищённых API на Go** с аутентификацией через Keycloak, проверкой JWT и разграничением доступа по ролям.

Проект показывает не только отдельную настройку Keycloak, а полный рабочий сценарий:

**пользователь → Keycloak → JWT → Go API → проверка роли → доступ к ресурсу**

Инфраструктура разворачивается в изолированной среде **Multipass**, поэтому проект удобно использовать для обучения, экспериментов и создания собственного прототипа защищённого API.

---

## 🎯 Что вы получите

Этот проект может быть полезен, если вам нужно быстро разобраться с авторизацией API и получить рабочую основу для собственного сервиса.

С его помощью можно:

* добавить OIDC-аутентификацию в Go API;
* проверять JWT-токены, выданные Keycloak;
* ограничивать доступ к API с помощью RBAC;
* проверять `audience` токена и принимать только предназначенные для API токены;
* разделить сервер авторизации и API на отдельные узлы;
* получить готовую структуру для дальнейшего расширения;
* протестировать защищённый API в изолированной виртуальной среде;
* использовать проект как основу для собственного backend-сервиса.

Вместо того чтобы собирать такую инфраструктуру с нуля, можно использовать этот проект как **стартовый шаблон** и адаптировать его под свои endpoints, роли и бизнес-логику.

---

## 💡 Для кого проект

Проект подойдёт разработчикам, которые:

* изучают **Go и backend-разработку**;
* хотят разобраться с **Keycloak и OIDC**;
* создают REST API с авторизацией;
* хотят понять работу **JWT и RBAC** на практике;
* экспериментируют с Docker и виртуальными машинами;
* собирают собственный backend или микросервис;
* хотят иметь изолированную среду для тестирования авторизации.

---

## 🧩 Какие задачи помогает решить

### Аутентификация

Keycloak выступает в роли Identity Provider и отвечает за аутентификацию пользователей.

Go API не хранит пароли пользователей и не реализует собственную систему входа. Вместо этого API принимает JWT, выданный Keycloak, и проверяет его подлинность.

### Авторизация

После проверки токена API определяет, какие действия разрешены пользователю.

Например:

```text
USER
 └── JWT
      │
      ▼
┌──────────────┐
│   Go API     │
├──────────────┤
│ JWT проверка │
│ Audience     │
│ RBAC         │
└──────┬───────┘
       │
       ├── user  → чтение / создание
       │
       └── admin → удаление
```

Таким образом, authentication и authorization разделены:

* **Keycloak** отвечает за идентификацию пользователя;
* **JWT** переносит информацию о пользователе и его правах;
* **Go API** проверяет токен;
* **RBAC** определяет доступ к защищённым операциям.

---

# 🏗 Архитектура

Проект состоит из двух виртуальных машин Ubuntu 24.04.

| Нода        | Назначение            |
| ----------- | --------------------- |
| `auth-node` | Keycloak + PostgreSQL |
| `api-node`  | Go API                |

### Схема взаимодействия

```text
                  ┌─────────────────────┐
                  │       User          │
                  └──────────┬──────────┘
                             │
                             │ Authentication
                             ▼
                  ┌─────────────────────┐
                  │      Keycloak       │
                  │       OIDC          │
                  └──────────┬──────────┘
                             │
                             │ JWT
                             ▼
                  ┌─────────────────────┐
                  │       Go API        │
                  │                     │
                  │ JWT validation      │
                  │ Audience validation │
                  │ RBAC                │
                  └──────────┬──────────┘
                             │
                             ▼
                         Resources
```

---

# 🔐 Что реализовано

## OAuth2 / OIDC

Keycloak используется как Identity Provider.

API получает токены через стандартный OpenID Connect / OAuth2 flow.

## JWT

API проверяет JWT, полученный от Keycloak.

Проверяются:

* подпись токена;
* срок действия;
* issuer;
* audience;
* роли пользователя.

## Audience Validation

API принимает только токены, предназначенные для конкретного клиента.

В токене ожидается:

```json
{
  "aud": "notes-api"
}
```

Это позволяет дополнительно ограничить использование токена конкретным API.

## RBAC

Права пользователя определяются на основе ролей Keycloak.

В демонстрационном проекте используется роль:

```text
admin
```

Например:

```text
GET    /api/v1/notes       → JWT
POST   /api/v1/notes       → JWT
DELETE /api/v1/notes/{id}  → admin
```

---

# 🚀 Быстрый старт

## 1. Требования

Перед запуском установите:

* Go;
* Docker;
* Docker Compose;
* Multipass.

---

## 2. Создание виртуальных машин

```bash
multipass launch --name auth-node --cpus 2 --memory 2G
multipass launch --name api-node --cpus 1 --memory 1G
```

Проверить созданные машины:

```bash
multipass list
```

---

## 3. Подключение проекта к API-ноду

```bash
multipass mount . api-node:/home/ubuntu/app
```

---

# 🔑 Запуск Keycloak

Подключите deployment-конфигурацию:

```bash
multipass mount deploy/keycloak auth-node:/home/ubuntu/keycloak-deploy
```

Откройте shell виртуальной машины:

```bash
multipass shell auth-node
```

Запустите Keycloak:

```bash
cd ~/keycloak-deploy
sudo docker-compose up -d
```

После запуска Keycloak будет доступен на порту `8080`.

> Значения IP-адресов в примерах являются частью демонстрационной конфигурации. Для собственного окружения используйте адрес, назначенный вашей Multipass-сетью.

---

# ⚙️ Настройка Keycloak

## Realm

Создайте Realm:

```text
my-project
```

## Client

Создайте клиента:

```text
notes-api
```

Рекомендуемые параметры демонстрационной конфигурации:

```text
Client authentication: ON
Standard flow: ON
Direct access grants: ON
```

Сохраните `Client Secret`.

---

## Роль

Создайте роль:

```text
admin
```

---

## Пользователь

Создайте тестового пользователя:

```text
testuser
```

Назначьте ему роль:

```text
admin
```

Для реального окружения используйте собственные безопасные учётные данные.

---

# 🎯 Настройка Audience

Создайте Client Scope:

```text
notes-api-scope
```

Тип:

```text
OpenID Connect
```

Добавьте Audience Mapper:

```text
Type: Audience
Included Client Audience: notes-api
```

После этого привяжите Client Scope к клиенту:

```text
Clients
 → notes-api
 → Client Scopes
 → Add
 → Default
```

После получения JWT проверьте payload.

Ожидаемое значение:

```json
"aud": "notes-api"
```

---

# 🔑 Получение JWT

Для демонстрационного окружения можно получить токен через token endpoint Keycloak:

```bash
curl -X POST "http://KEYCLOAK_HOST:8080/realms/my-project/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=notes-api" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "username=testuser" \
  -d "password=YOUR_PASSWORD" \
  -d "grant_type=password"
```

Полученный access token сохраните:

```bash
TOKEN="your_jwt_token"
```

> Для production-системы рекомендуется выбирать OAuth2/OIDC flow в соответствии с типом клиента и требованиями безопасности. Resource Owner Password Credentials не следует использовать как универсальный production-подход.

---

# 🚀 Запуск Go API

Подключитесь к API-ноду:

```bash
multipass shell api-node
```

При необходимости установите Go:

```bash
sudo snap install go --classic
```

Перейдите в проект:

```bash
cd app
```

Установите зависимости:

```bash
go mod tidy
```

Запустите API:

```bash
go run cmd/api/main.go
```

---

# ⚙️ Переменные окружения

Создайте `.env` на основе `.env.example`.

Пример:

```env
KEYCLOAK_URL=http://KEYCLOAK_HOST:8080
KEYCLOAK_REALM=my-project
KEYCLOAK_CLIENT_ID=notes-api
PORT=8000
```

Не добавляйте реальные секреты и пароли в Git.

---

# 🛠 API

| Метод    | Endpoint             | Доступ | Назначение                     |
| -------- | -------------------- | ------ | ------------------------------ |
| `GET`    | `/health`            | Public | Проверка работоспособности API |
| `GET`    | `/api/v1/notes`      | JWT    | Получение заметок              |
| `POST`   | `/api/v1/notes`      | JWT    | Создание заметки               |
| `DELETE` | `/api/v1/notes/{id}` | Admin  | Удаление заметки               |

Такой набор endpoints позволяет на практике проверить сразу несколько сценариев авторизации:

```text
Public
  ↓
JWT authentication
  ↓
RBAC authorization
```

---

# 🧪 Проверка API

## Health check

```bash
curl http://API_HOST:8000/health
```

---

## Получение заметок

```bash
curl \
  -H "Authorization: Bearer $TOKEN" \
  http://API_HOST:8000/api/v1/notes/
```

---

## Создание заметки

```bash
curl -X POST http://API_HOST:8000/api/v1/notes/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"New Note","content":"Content of the note"}'
```

---

## Удаление заметки

```bash
curl -X DELETE http://API_HOST:8000/api/v1/notes/1 \
  -H "Authorization: Bearer $TOKEN"
```

Для выполнения операции пользователь должен иметь соответствующую роль.

---

# 🤖 Автоматическая проверка

В проекте предусмотрен shell-скрипт для проверки API.

```bash
cd tests
chmod +x check_notes_api.sh
./check_notes_api.sh
```

Скрипт позволяет проверить:

* получение JWT;
* публичный endpoint;
* защищённые endpoints;
* авторизацию;
* RBAC;
* доступ к административной операции.

Это удобно после изменения конфигурации Keycloak или кода API: вместо ручной проверки можно быстро прогнать основной сценарий.

---

# 🔒 Модель безопасности

В проекте демонстрируется несколько уровней проверки.

### 1. OIDC / OAuth2

Keycloak выполняет аутентификацию пользователя.

### 2. JWT validation

Go API проверяет полученный access token.

### 3. Audience validation

API проверяет, что токен предназначен именно для этого сервиса.

### 4. RBAC

API использует роли из:

```text
realm_access.roles
```

### 5. Bearer Token

Защищённые endpoints требуют:

```http
Authorization: Bearer <JWT>
```

---

# 📦 Структура проекта

```text
.
├── cmd/
│   └── api/
├── deploy/
│   └── keycloak/
├── internal/
├── tests/
├── .env.example
├── docker-compose.yml
├── go.mod
├── go.sum
└── README.md
```

---

# 🧰 Технологии

* **Go** — backend API
* **Keycloak** — Identity Provider
* **OpenID Connect** — аутентификация
* **OAuth2** — протокол авторизации
* **JWT** — передача и проверка claims
* **RBAC** — разграничение доступа
* **PostgreSQL** — хранилище Keycloak
* **Docker / Docker Compose** — запуск инфраструктуры
* **Multipass** — изолированная виртуальная среда
* **Shell** — автоматизированная проверка API

---

# 🔄 Как использовать проект для своего API

Проект можно рассматривать как стартовую точку.

Например, вместо:

```text
/api/v1/notes
```

можно добавить собственные ресурсы:

```text
/api/v1/users
/api/v1/orders
/api/v1/products
/api/v1/documents
```

А роли:

```text
admin
```

расширить:

```text
admin
manager
user
```

Таким образом, основную инфраструктуру аутентификации и авторизации не обязательно реализовывать с нуля — можно адаптировать существующую структуру под собственный сервис.

---

# 🎓 Что можно изучить на этом проекте

После запуска проекта можно на практике разобраться:

* как Go API взаимодействует с Keycloak;
* как устроен OIDC;
* откуда берётся JWT;
* какие claims находятся внутри токена;
* зачем нужен `aud`;
* как API проверяет подпись JWT;
* как роли Keycloak попадают в токен;
* как реализуется RBAC;
* как разделить Identity Provider и API;
* как запускать подобную инфраструктуру в виртуальной среде.

---

# ⚠️ Важно для production

Проект предназначен прежде всего как **готовая учебная и стартовая основа**.

Перед использованием в production необходимо отдельно проверить:

* хранение секретов;
* TLS/HTTPS;
* сетевые правила;
* настройки Keycloak;
* политики паролей;
* срок жизни токенов;
* OAuth2/OIDC flow;
* управление пользователями;
* аудит и логирование;
* резервное копирование PostgreSQL;
* ограничения доступа к административным интерфейсам.

Демонстрационные IP-адреса, пароли и настройки не следует переносить в production без изменений.

---

# 🚀 Идея проекта

Главная цель проекта — показать практический путь от **аутентификации пользователя до защищённого API endpoint**.

Вместо самостоятельной реализации системы авторизации с нуля разработчик получает рабочую архитектуру, которую можно изучить, запустить и использовать как основу для собственного Go-сервиса.

```text
Keycloak
   │
   │ OIDC / JWT
   ▼
Go API
   │
   ├── Authentication
   ├── Audience validation
   └── RBAC
        │
        ▼
   Protected resources
```

## 📌 Коротко

**Secure Go API Starter** — это практический шаблон для разработчиков, которым нужно добавить в Go API аутентификацию через Keycloak, проверку JWT и ролевой доступ без построения всей инфраструктуры с нуля.
