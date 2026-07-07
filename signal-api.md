# Документация по Signal API

> API платформы Signal (sgnl.pro) для управления данными BIM-проектов.
> 
> Базовый URL: `https://api.sgnl.pro/public/v1`

---

## 🔐 Авторизация

### Получение токена

**Endpoint**: `POST /auth/token`

**Тело запроса**:
```json
{
  "clientId": "your_client_id",
  "clientSecret": "your_client_secret",
  "scopes": [
    "project:read", "project:list",
    "docs:folder:read", "docs:folder:list",
    "docs:item:read", "docs:item:list",
    "docs:version:read", "docs:version:list",
    "docs:object:read", "docs:object:list"
  ]
}
```

**Ответ**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Использование**: передавайте токен в заголовке `Authorization: Bearer <token>`

---

## 📁 Проекты

### Список проектов

**Endpoint**: `GET /projects`

**Заголовки**:
```
Authorization: Bearer <token>
Accept: application/json
```

**Ответ**:
```json
[
  {
    "id": "uuid-project-1",
    "name": "ЖК МЫ поз.6.2",
    "description": "...",
    "createdAt": "2024-01-15T10:00:00Z"
  }
]
```

---

## 📂 Документы и папки

### Получение корневой папки проекта

**Endpoint**: `GET /docs/projects/{projectId}`

**Ответ**:
```json
{
  "id": "uuid-root-folder",
  "name": "root",
  "rootFolderId": "uuid-root-folder",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

### Список дочерних папок

**Endpoint**: `GET /folders/{folderId}/children`

**Ответ**:
```json
[
  {
    "id": "uuid-folder-1",
    "name": "6. СМР",
    "parentId": "uuid-root-folder",
    "createdAt": "2024-01-15T10:00:00Z"
  }
]
```

### Поиск папки по имени

Рекомендуется получить список дочерних папок и выполнить поиск на клиенте:

```javascript
const sameName = (a, b) => 
  (a ?? '').toString().trim().toLowerCase() === 
  (b ?? '').toString().trim().toLowerCase();

const folder = children.find(f => sameName(f.name, "Журнал бетонных работ"));
```

### Рекурсивный обход пути

Для пути `"6. СМР/Журнал бетонных работ"`:

```javascript
const splitPath = (p) =>
  (p ?? '').toString()
    .replaceAll('\\', '/')
    .split('/')
    .map(x => x.trim())
    .filter(Boolean);

// ["6. СМР", "Журнал бетонных работ"]
```

---

## 📄 Файлы (Items)

### Список файлов в папке

**Endpoint**: `GET /items?folderId={folderId}`

**Ответ**:
```json
[
  {
    "id": "uuid-item-1",
    "name": "12.csv",
    "folderId": "uuid-folder-1",
    "topVersionId": "uuid-version-1",
    "createdAt": "2024-01-15T10:00:00Z",
    "updatedAt": "2024-01-15T10:00:00Z"
  }
]
```

**Важно**: поле `topVersionId` необходимо для получения ссылки на скачивание.

### Получение ссылки на скачивание

**Endpoint**: `GET /items/download?itemId={itemId}&versionId={versionId}`

**Ответ**:
```json
{
  "signedUrl": "https://storage.sgnl.pro/...?token=...",
  "expiresAt": "2024-01-15T12:00:00Z"
}
```

**Примечание**: `signedUrl` — временная подписанная ссылка. Скачивайте файл сразу после получения.

---

## 🔄 Полный workflow загрузки файла

```
1. POST /auth/token → получить JWT
2. GET /projects → найти projectId по имени
3. GET /docs/projects/{projectId} → получить rootFolderId
4. GET /folders/{rootFolderId}/children → найти первую папку
5. GET /folders/{folderId}/children → найти следующую папку (рекурсивно)
6. GET /items?folderId={finalFolderId} → найти itemId и versionId
7. GET /items/download?itemId={itemId}&versionId={versionId} → получить signedUrl
8. GET {signedUrl} → скачать файл
```

---

## ⚠️ Обработка ошибок

### Коды ответов

| Код | Значение | Действие |
|-----|----------|----------|
| 200 | OK | Успешно |
| 401 | Unauthorized | Токен истёк или невалиден — запросите новый |
| 403 | Forbidden | Недостаточно scopes — проверьте права |
| 404 | Not Found | Ресурс не найден — проверьте ID |
| 429 | Too Many Requests | Rate limit — подождите и повторите |
| 500 | Internal Error | Проблема на стороне сервера — обратитесь в поддержку |

### Пример обработки ошибок

```javascript
const httpJsonLenient = async (method, url, { headers = {}, body } = {}) =>
  await this.helpers.httpRequest({
    method,
    url,
    headers: { accept: 'application/json', ...headers },
    body,
    json: true,
    returnFullResponse: true,
    ignoreHttpStatusCode: true,
  });

// Проверка ответа
if (!(res.statusCode >= 200 && res.statusCode < 300 && res.body?.token)) {
  throw new Error(`Auth failed: ${res.statusCode} ${JSON.stringify(res.body || {})}`);
}
```

---

## 🔒 Scopes (разрешения)

| Scope | Описание |
|-------|----------|
| `project:read` | Чтение информации о проекте |
| `project:list` | Список проектов |
| `docs:folder:read` | Чтение папок |
| `docs:folder:list` | Список папок |
| `docs:item:read` | Чтение файлов |
| `docs:item:list` | Список файлов |
| `docs:version:read` | Чтение версий |
| `docs:version:list` | Список версий |
| `docs:object:read` | Чтение объектов |
| `docs:object:list` | Список объектов |

---

## 📎 Полезные ссылки

- [Официальный сайт Signal](https://sgnl.pro)
- [Документация API](https://docs.sgnl.pro) *(если доступна)*
- [n8n Community](https://community.n8n.io) — для вопросов по интеграции
