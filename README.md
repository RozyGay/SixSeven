## 1. Что где лежит

| Путь | Что это |
|---|---|
| `RpToolkit/Modules/AdminLink/` | исходники модуля |
| `RpToolkit/bin/Release/RpToolkit.dll` | собранный плагин (Release) |
| `configs/RPToolkit-AdminLink/keystore.json` | ключи администраторов + ключ овнера |
| `configs/RPToolkit-AdminLink/allowed_ips.txt` | белый список IP (необязательно) |

**В конфиге плагина (`RpToolkit.yml` / конфиг EXILED) появилась секция:**

```yaml
admin_link:
  enabled: true
  port: 3091
  bind_address: 0.0.0.0
  owner_public_hash: ""
  max_requests_per_minute: 120
  allow_arbitrary_commands: true
  blocked_commands:
    - adminlink
  expose_player_ip: true
  verbose_log: false
```

---

## 2. ...

- PYSTO

---

## 3. Ключи и hash-коды

### Как это устроено

- У каждого администратора есть **raw-ключ** — строка из 64 hex-символов.
- По сети передаётся **не сам ключ**, а `hash = sha256(raw_key)` — тоже 64 hex-символа.
- На сервере хранится **только hash**. Raw-ключ не сохраняется нигде, поэтому
  показывается один раз — при создании.
- Регистр не важен: `hash` можно передавать и в нижнем, и в верхнем регистре.

### Резервный ключ овнера

Чтобы плагин был работоспособен сразу после установки, в него вшит резервный
ключ:

```
raw:  RPTK-OWNER-f8c9726aa3b284db0f94d22bb0c87647894faa5e
hash: b4e58e531b345bbc1e93ad1d853fa486a852ffa4d97922e064ba5e7e247db243
```

> ⚠️ **Этот ключ нужен только для первого входа. Обязательно замени его своим
> через `key_rotate_owner` (§7) и пропиши новый hash в конфиг.** Пока в конфиге
> `owner_public_hash` пуст, при каждом старте в логе будет предупреждение.

### Как считать hash (для приложения)

```
hash = lowercase( hex( sha256( raw_key_as_utf8_bytes ) ) )
```

Примеры:

```python
import hashlib
hashlib.sha256(b"RPTK-OWNER-f8c9726aa3b284db0f94d22bb0c87647894faa5e").hexdigest()
# -> b4e58e531b345bbc1e93ad1d853fa486a852ffa4d97922e064ba5e7e247db243
```

```kotlin
MessageDigest.getInstance("SHA-256")
    .digest(rawKey.toByteArray(Charsets.UTF_8))
    .joinToString("") { "%02x".format(it) }
```

```javascript
// Node
crypto.createHash('sha256').update(rawKey, 'utf8').digest('hex')
```

### Роли

| Роль | Что может |
|---|---|
| `moderator` | `ping`, `help`, `server_status`, `player_list`, `player_info` |
| `admin` | всё от moderator + `kick`, `ban`, `command`, `round_restart` |
| `owner` | всё от admin + `server_shutdown`, `server_restart` и всё управление ключами |

Овнер — один, его ключ нельзя отозвать или понизить. Администраторов можно
создать до 256 штук.

---

## 4. Формат сообщений (транспорт)

Одно TCP-соединение = один запрос = один ответ. После ответа сервер закрывает
соединение.

**Кадр запроса и ответа:**

```
[4 байта: длина JSON, big-endian][UTF-8 JSON]
```

Максимальный размер кадра — **65536 байт**. Превышение даёт ошибку
`bad_request`, а не обрыв соединения.

Таймауты на сервере: 8 секунд на чтение и 8 на запись. Если игровое действие
не уложилось в 3 секунды — ответ `timeout`.

### Запрос

```json
{
  "v": 1,
  "hash": "b4e58e531b345bbc1e93ad1d853fa486a852ffa4d97922e064ba5e7e247db243",
  "action": "player_list",
  "params": { },
  "rid": "req-001"
}
```

| Поле | Тип | Обяз. | Что это |
|---|---|---|---|
| `v` | int | да | Версия протокола. Всегда `1` |
| `hash` | string | да | SHA-256 от ключа, 64 hex-символа |
| `action` | string | да | Действие, см. §6 |
| `params` | object | нет | Параметры действия |
| `rid` | string | нет | Твой идентификатор запроса — вернётся в ответе как есть |

### Ответ (успех)

```json
{
  "ok": true,
  "v": 1,
  "ts": 1758290400,
  "action": "player_list",
  "rid": "req-001",
  "data": { "count": 2, "players": [ ... ] }
}
```

### Ответ (ошибка)

```json
{
  "ok": false,
  "v": 1,
  "ts": 1758290400,
  "action": "ban",
  "rid": "req-001",
  "error": {
    "code": "bad_params",
    "msg": "Параметр 'minutes' должен быть положительным числом.",
    "field": "minutes"
  }
}
```

Поле `field` есть не всегда — только когда ошибка относится к конкретному
параметру.

---

## 5. Примеры подключения

### Python

```python
import socket, struct, json, hashlib

HOST, PORT = "1.2.3.4", 3091

def call(raw_key, action, params=None, rid=None):
    h = hashlib.sha256(raw_key.encode()).hexdigest()
    req = {"v": 1, "hash": h, "action": action,
           "params": params or {}, "rid": rid}
    body = json.dumps(req, ensure_ascii=False).encode("utf-8")

    with socket.create_connection((HOST, PORT), timeout=10) as s:
        s.sendall(struct.pack(">I", len(body)) + body)
        header = s.recv(4)
        (n,) = struct.unpack(">I", header)
        data = b""
        while len(data) < n:
            chunk = s.recv(n - len(data))
            if not chunk:
                break
            data += chunk

    return json.loads(data.decode("utf-8"))

print(call("RPTK-OWNER-f8c9726aa3b284db0f94d22bb0c87647894faa5e", "ping"))
```

### Kotlin / Android

```kotlin
fun call(rawKey: String, action: String, params: JSONObject = JSONObject()): JSONObject {
    val hash = MessageDigest.getInstance("SHA-256")
        .digest(rawKey.toByteArray(Charsets.UTF_8))
        .joinToString("") { "%02x".format(it) }

    val req = JSONObject()
        .put("v", 1)
        .put("hash", hash)
        .put("action", action)
        .put("params", params)

    val body = req.toString().toByteArray(Charsets.UTF_8)

    Socket().use { sock ->
        sock.connect(InetSocketAddress(HOST, PORT), 10_000)
        sock.soTimeout = 10_000
        val out = sock.getOutputStream()
        out.write(ByteBuffer.allocate(4).putInt(body.size).array()) // big-endian
        out.write(body)
        out.flush()

        val input = DataInputStream(sock.getInputStream())
        val len = input.readInt()
        val buf = ByteArray(len)
        input.readFully(buf)
        return JSONObject(String(buf, Charsets.UTF_8))
    }
}
```

> Порт `3091` должен быть открыт в фаерволе сервера и, если сервер за NAT,
> проброшен наружу.

---

## 6. Действия

Легенда прав: **M** = moderator, **A** = admin, **O** = owner.

### `ping` — M

Проверить ключ, связь и свои права.

Параметров нет.

```json
{ "pong": true, "role": "owner", "label": "owner", "id": "owner",
  "server_time": 1758290400, "plugin": "RPToolkit", "protocol": 1 }
```

### `help` — M

Список действий, их параметров и кодов ошибок. Удобно для отладки.

Параметров нет. Отдаёт `version`, `actions` (описание всех действий, их
параметров и минимальной роли) и `errors` (расшифровка кодов).

### `server_status` — M

Состояние сервера.

```json
{
  "name": "MIRAGE PROJECT", "port": 7777, "version": "14.0.0",
  "players": 12, "max_players": 25, "tps": 58.4, "ip": "1.2.3.4",
  "round": { "in_progress": true, "is_lobby": false, "is_started": true,
             "is_ended": false, "is_locked": false, "elapsed_seconds": 342 },
  "plugin_version": "2.0.1"
}
```

`version` — версия игры, `plugin_version` — версия плагина.

### `player_list` — M

Список игроков онлайн.

```json
{
  "count": 2,
  "players": [
    {
      "id": 3, "nickname": "Игрок", "display_nickname": "Игрок",
      "user_id": "76561198000000000@steam", "raw_user_id": "76561198000000000",
      "role": "ClassD", "role_team": "ClassD",
      "group": "admin", "group_badge": "⚡", "rank_name": "admin", "rank_color": "orange",
      "remote_admin": true, "kick_power": 5, "is_staff": true,
      "is_northwood": false, "is_global_moderator": false,
      "ping": 42, "ip": "5.6.7.8",
      "is_alive": true, "is_scp": false, "is_muted": false,
      "is_overwatch": false, "is_god_mode": false, "is_noclip": false,
      "zone": "LightContainment"
    }
  ]
}
```

`ip` пустая строка, если в конфиге `expose_player_ip: false`.

### `player_info` — M

Карточка одного игрока.

| Параметр | Тип | Обяз. |
|---|---|---|
| `id` | int | да |

Отдаёт те же поля, что `player_list`, плюс `room`, `position` (`x`, `y`, `z`),
`health`, `max_health`, `armor` (тип брони или `"None"`), `inventory`
(массив `{"type": ..., "serial": ...}`).

Нет такого игрока → ошибка `player_not_found`.

### `kick` — A

Кикнуть с причиной.

| Параметр | Тип | Обяз. |
|---|---|---|
| `id` | int | да |
| `reason` | string | да |

```json
{ "kicked": true, "id": 3, "user_id": "76561198...@steam", "reason": "Правила" }
```

Нет такого игрока → ошибка `player_not_found`.

### `ban` — A

Забанить на N минут с причиной.

| Параметр | Тип | Обяз. |
|---|---|---|
| `id` | int | да |
| `minutes` | int (> 0) | да |
| `reason` | string | да |

```json
{ "banned": true, "id": 3, "user_id": "76561198...@steam",
  "minutes": 60, "seconds": 3600, "reason": "Читы" }
```

Нет такого игрока → ошибка `player_not_found`.

### `command` — A

Любая консольная команда сервера. Слеш в начале не обязателен — он будет
добавлен сам.

| Параметр | Тип | Обяз. |
|---|---|---|
| `cmd` | string | да |

```json
{ "executed": true, "cmd": "/roundrestart", "result": "" }
```

> Это полный доступ к консоли сервера. Если не нужно — выстави
> `allow_arbitrary_commands: false`. Отдельные команды можно запретить списком
> `blocked_commands` (сравнение по первому слову, без слеша).

### `round_restart` — A

Перезапустить раунд.

| Параметр | Тип | Обяз. | По умолчанию |
|---|---|---|---|
| `fast` | bool | нет | `true` |

`fast: true` — быстрый рестарт, игроки остаются на сервере.
`fast: false` — с переподключением.

```json
{ "restarted": true, "fast": true }
```

### `server_shutdown` — O

Остановить сервер.

| Параметр | По умолчанию |
|---|---|
| `quit` (bool) | `false` |

`quit: true` завершает процесс приложения сервера.

```json
{ "shutdown": true, "quit": false }
```

> После этого сервер **не поднимется сам** - его нужно запускать вручную или
> через систему перезапуска.

### `server_restart` — O

Полный перезапуск сервера с переподключением игроков. Параметр `fast`
(по умолчанию `false`).

```json
{ "restarting": true, "fast": false }
```

### `key_create` — O

Создать администратора и получить его ключ.

| Параметр | По умолчанию |
|---|---|
| `label` (string) | `admin-<случайное>` |
| `role` (`moderator` \| `admin`) | `moderator` |

```json
{
  "id": "a1b2c3d4e5f6",
  "label": "Вася",
  "role": "admin",
  "hash": "3f9a...",
  "raw_key": "c41d...",
  "warning": "Ключ показан ОДИН раз и не хранится на сервере. Сохраните его немедленно."
}
```

> `raw_key` возвращается **только здесь** — это единственный шанс его увидеть.
> Отдай его администратору, в приложении храни `hash` (или сам raw, если
> приложение считает hash локально).

### `key_list` — O

Список администраторов. Сами ключи не возвращаются — только `id`, `label`,
`role`, `revoked`, `created_unix`, `last_used_unix`, `created_by`.

```json
{ "count": 2, "keys": [ { "id": "a1b2c3d4e5f6", "label": "Вася", "role": "admin",
    "revoked": false, "created_unix": 1758290000, "last_used_unix": 1758290400,
    "created_by": "owner" } ], "hint": "9c1f3a22" }
```

`hint` — первые символы hash овнера, чтобы понимать, какой ключ сейчас активен.

### `key_revoke` — O

Отозвать ключ администратора.

| Параметр | Тип | Обяз. |
|---|---|---|
| `id` | string (из `key_list`) | да |

Ключ овнера отозвать нельзя — вернётся `owner_immutable`.

```json
{ "id": "a1b2c3d4e5f6", "revoked": true }
```

### `key_role` — O

Сменить роль администратора.

| Параметр | Тип | Обяз. |
|---|---|---|
| `id` | string | да |
| `role` | `moderator` \| `admin` | да |

Роль `owner` назначить нельзя — вернётся `bad_params`.

```json
{ "id": "a1b2c3d4e5f6", "role": "admin" }
```

### `key_rotate_owner` — O

Сгенерировать новый ключ овнера. **Старый ключ сразу перестаёт работать.**
Ключи остальных администраторов сохраняются.

```json
{
  "hash": "9c1f...",
  "raw_key": "77ba...",
  "hint": "9c1f3a22",
  "warning": "Старый ключ овнера больше не работает. Запишите OwnerPublicHash в конфиг..."
}
```

> Обязательно перенеси `hash` в конфиг `owner_public_hash`, иначе при следующем
> старте плагина конфиг вернёт старое значение и новый ключ перестанет работать.

### `audit` — O

Журнал последних действий (в памяти, до 500 записей, при перезапуске
очищается).

| Параметр | По умолчанию |
|---|---|
| `limit` (int) | `50` |

```json
{ "entries": [ { "unix": 1758290400, "identity_id": "owner", "label": "owner",
    "role": "owner", "action": "ban", "target": "76561198...@steam",
    "ok": true, "code": "", "ip": "1.2.3.4" } ] }
```

---

## 7. Как поменять ключ овнера

### Способ 1 — через приложение (рекомендуемый)

1. Зайди под текущим ключом овнера.
2. Вызови `key_rotate_owner`.
3. Сохрани `raw_key` (он показан один раз).
4. В конфиге плагина пропиши:
   ```yaml
   admin_link:
     owner_public_hash: "<hash из ответа>"
   ```
5. Перезапусти плагин и проверь `ping` новым ключом.

### Способ 2 — руками, если доступ потерян

1. Посчитай hash от нового ключа:
   ```
   hash = sha256_hex("МОЙ-НОВЫЙ-КЛЮЧ")
   ```
   Например в PowerShell:
   ```powershell
   $b = [Text.Encoding]::UTF8.GetBytes("МОЙ-НОВЫЙ-КЛЮЧ")
   -join ([Security.Cryptography.SHA256]::Create().ComputeHash($b) | % { $_.ToString("x2") })
   ```
2. Впиши его в `owner_public_hash` конфига.
3. Перезапусти сервер.

Конфиг всегда главнее файла `keystore.json` — это и есть механизм восстановления
доступа.

### Способ 3 — удалить keystore

Останови сервер, удали `configs/RPToolkit-AdminLink/keystore.json` и задай свой
`owner_public_hash` в конфиге. Все созданные администраторы при этом сотрутся.

---

## 8. Все ошибки и что с ними делать

| Код | Значение | Как исправить |
|---|---|---|
| `disabled` | Модуль выключен | `admin_link.enabled: true` в конфиге |
| `bad_json` | Тело не JSON | Проверь сериализацию и кодировку (UTF-8) |
| `bad_request` | Нет `hash` или `action`; кадр > 65536 байт | Заполни обязательные поля |
| `bad_version` | `v` не равен 1 | Пришли `"v": 1` |
| `bad_hash_format` | `hash` не 64 hex-символа | Пришли `sha256(ключа)` в hex, 64 символа |
| `unauthorized` | Ключ не найден или отозван | Проверь ключ; запроси новый у овнера |
| `forbidden` | Не хватает прав | Нужна роль выше (см. §3) |
| `unknown_action` | Нет такого действия | `action="help"` — список действий |
| `bad_params` | Параметр отсутствует, пустой или не того типа | Сверься с §6; `minutes` должен быть > 0 |
| `player_not_found` | Игрока с таким `id` нет | `id` актуален только пока игрок онлайн — обнови `player_list` |
| `rate_limited` | Слишком много запросов с IP | Подожди минуту; увеличь `max_requests_per_minute` |
| `ip_denied` | IP не в белом списке | Добавь IP в `allowed_ips.txt` |
| `timeout` | Сервер не ответил за 3 с | Повтори запрос |
| `internal_error` | Ошибка плагина | Смотри логи сервера |
| `identity_not_found` | Нет администратора с таким id | `key_list` → актуальный id |
| `owner_immutable` | Попытка изменить/отозвать овнера | Овнера менять нельзя, только ротировать |
| `already_revoked` | Ключ уже отозван | — |
| `identity_limit_reached` | Больше 256 администраторов | Отзови неиспользуемые ключи |
| `persist_failed` | Keystore не записан на диск | Проверь права на папку конфигов |
| `hash_collision` | Крайне редкая коллизия | Повтори запрос |

---

## 9. Частые проблемы

**Приложение не подключается**
Порт `3091` закрыт фаерволом или не проброшен. Проверь:
`Test-NetConnection -ComputerName <ip> -Port 3091`.
В логе сервера при старте должна быть строка
`[AdminLink] Модуль запущен. Порт: 3091`.

**Всегда `unauthorized`**
Скорее всего передаётся сам ключ, а не `sha256(ключа)`, или строка не из
64 символов. Проверь: `sha256` от `RPTK-OWNER-f8c9726aa3b284db0f94d22bb0c87647894faa5e`
должен дать `b4e58e531b345bbc1e93ad1d853fa486a852ffa4d97922e064ba5e7e247db243`.
Если совпадает — значит формат верный, проблема в ключе.

**Ответ приходит, но `error.code = bad_params` на `ban`**
`minutes` должен быть целым числом больше нуля. Строка `"60"` тоже сработает —
модуль умеет её распознать.

**После перезапуска сервера новый ключ овнера не работает**
`owner_public_hash` в конфиге пуст или содержит старое значение — конфиг
перекрывает файл. См. §7.

**Игрок был в `player_list`, а `kick` вернул `player_not_found`**
Игрок вышел между запросами. Обнови список и возьми свежий `id`.

**Не читается ответ**
Не забудь 4 байта длины **перед** JSON, big-endian, и читай ровно `length`
байт (TCP может отдать тело частями — читай в цикле).

---

## 10. Безопасность

**Что сделано:**

- По сети передаётся только `sha256(ключа)`, raw-ключ — никогда.
- Raw-ключи не хранятся на сервере: только hash.
- Сравнение hash — постоянного времени (защита от подбора по времени ответа).
- Ключи генерируются криптографическим ГПСЧ (32 байта = 256 бит).
- Лимит запросов в минуту на IP, после превышения — отклонение.
- Белый список IP (`allowed_ips.txt`, по одному адресу в строке, `#` — комментарий).
- Лимит кадра 64 КБ — защита от переполнения памяти.
- Иерархия прав: опасные действия (`server_shutdown`, `server_restart`,
  управление ключами) доступны только овнеру.
- Произвольные команды отключаются (`allow_arbitrary_commands`) и дополняются
  списком запрещённых (`blocked_commands`).
- Все действия пишутся в журнал (`audit`) с ролью и IP.

---

## 11. Для разработчика

Структура модуля:

| Файл | Ответственность |
|---|---|
| `AdminLinkModule.cs` | Модуль EXILED (`ModuleBase`): жизненный цикл, приём запроса, белый список IP |
| `AdminLinkSettings.cs` | Секция `AdminLink` конфига плагина |
| `Transport.cs` | TCP-сервер: кадрирование, таймауты, лимит запросов |
| `Protocol.cs` | Разбор/сборка JSON-конвертов, коды ошибок |
| `Dispatcher.cs` | Действия и права; игровой мир трогается **только** из главного потока Unity |
| `KeyStore.cs` | Ключи, роли, отзыв, ротация овнера, журнал |
| `Crypto.cs` | SHA-256, сравнение постоянного времени, ГПСЧ |
| `Models.cs` | Роли и типы данных |
