# WPS Integration for SuiteCRM 7

Jump to: **[English](#english)** · **[Русский](#русский)**

## English

A SuiteCRM 7 plugin integrating with **Workflow Processing Service (WPS)** — a business process automation service. The package is self-contained: it contains only the CRM code, installers and documentation.

- [1. Purpose and Scope](#1-purpose-and-scope)
- [2. Compatibility](#2-compatibility)
- [3. Requirements](#3-requirements)
- [4. Installation](#4-installation)
- [5. Web Server Configuration](#5-web-server-configuration)
- [6. WPS_* Parameters](#6-wps_-parameters)
- [7. Post-Installation Verification](#7-post-installation-verification)
- [8. Connecting WPS to This CRM](#8-connecting-wps-to-this-crm)
- [9. Upgrading](#9-upgrading)
- [10. Uninstalling](#10-uninstalling)
- [11. Troubleshooting](#11-troubleshooting)
- [12. Event and API Contract](#12-event-and-api-contract)

### 1. Purpose and Scope

The plugin connects the CRM and WPS in both directions:

| Direction | How it is implemented |
|---|---|
| CRM → WPS | The application-level `after_save` logic hook (`WpsEventHook`) publishes the saved record to the WPS Redis Stream: `XADD crm_events MAXLEN ~ 100000 * data '{json}'`. The Redis client is written in pure PHP (`WpsRedisClient`), the phpredis extension is not required. |
| WPS → CRM | The custom entry point `wpsApi` (`WpsApi`) exposes an API with the WPS contract: `GET/POST/PUT /api/v9/{module}[/{id}]`, search `?filter[field]=value&size=1`, dictionary `/api/v9/modules`. The `/api/v9/...` path is routed to the entry point by a web server rewrite rule. |
| Marker | The `Leads.wps_processed_c` field ("Processed by WPS"), set by WPS processes. |

The SuiteCRM core is not modified: files are installed into `custom/` via the Extension framework, the package appears in Admin → Module Loader and is removed from there.

```
suiteCRM7/
├── README.md                 this file
├── CHANGELOG.md
├── VERSION                   package version (= manifest.php → version)
├── package/                  contents of the zip for Module Loader (manifest.php at the root)
│   ├── manifest.php
│   ├── files/custom/include/WPS/   WpsApi, WpsEventHook, WpsRedisClient, WpsBeanSerializer, WpsConfig, entry point
│   ├── files/extension/            logic hook and entry point registration, field labels (en_us, ru_RU)
│   └── scripts/                    creation / removal of Leads.wps_processed_c (idempotent)
├── bin/
│   ├── install.sh            automatic installation and reinstallation
│   ├── uninstall.sh          removal
│   ├── build.sh              builds dist/wps_integration-<version>-suitecrm7.zip
│   └── wps_setup.php         CLI: config / install / uninstall / status (called from install.sh and uninstall.sh)
├── config/
│   ├── wps.env.example       all WPS_* parameters with descriptions
│   ├── apache-wps.conf       /api/v9 rewrite for a VirtualHost
│   ├── htaccess-wps.conf     /api/v9 rewrite for .htaccess (if the vhost is unavailable)
│   └── nginx-wps.conf        /api/v9 rewrite for nginx
└── dist/                     output of build.sh
```

### 2. Compatibility

| | Version |
|---|---|
| SuiteCRM | 7.14.x (tested on 7.14.9). `manifest.php` accepts `^7\.14\.` |
| PHP | 8.1, 8.2 (same as SuiteCRM 7.14; tested on 8.2) + the `zip` extension |
| Database | MariaDB 10.4–10.11 / MySQL 5.7–8.0 (same as SuiteCRM 7.14; tested on MariaDB 10.11) |
| Web server | Apache 2.4 with `mod_rewrite` (tested); nginx — the snippet is provided, not tested |
| WPS | CRM API contract `/api/v9`, Redis Stream `crm_events` |

For SuiteCRM 8 there is a separate `SuiteCRM8` package — this package will not be installed by the SuiteCRM 8 installer. To install this package in another 7.x at your own risk, use `bin/install.sh --force` (Module Loader will refuse).

### 3. Requirements

- SuiteCRM is installed (`config.php` contains `'installer_locked' => true`).
- The WPS Redis Stream is reachable from the CRM (default `redis-stream:6379`), and the CRM web server is reachable from WPS.
- For `bin/install.sh`: bash, a PHP CLI of the same version as the CRM, root privileges, the web server user (`www-data`)
  or `sudo`; for the Docker mode — `docker` on the host.
- The web server user can write to `custom/`, `cache/`, `upload/` and `config_override.php`.

### 4. Installation

First prepare the parameters (secrets end up neither in the package nor on the command line):

```bash
cp config/wps.env.example config/wps.env
# set WPS_API_STATIC_KEY (= CRM_API_STATIC_KEY in WPS) and the WPS Redis address
```

Parameters already set in the environment take precedence over the file. A different file is passed with the `--env-file` flag.

#### 4.1. Script: SuiteCRM on This Machine

1. Run from the package directory (as root, as `www-data`, or via `sudo`):

   ```bash
   bin/install.sh --crm-path /var/www/html --apache-conf /etc/apache2/sites-available/000-default.conf
   ```

2. The script, step by step:
   1. verifies that this is an installed SuiteCRM 7, that the PHP `zip` extension exists and that write permissions are in place;
   2. writes `$sugar_config['wps']` to `config_override.php`;
   3. installs the package via `ModuleInstaller` and registers it in Module Loader (`upgrade_history`), creating the
      `Leads.wps_processed_c` field if it does not exist yet;
   4. adds the `/api/v9` rewrite to the specified VirtualHost between the `# BEGIN/END WPS integration` markers, checks
      the configuration (`apachectl -t`) and reloads Apache. Without `--apache-conf` the fragment is only printed
      (see [Section 5](#5-web-server-configuration)), and with `--htaccess` it is installed into `.htaccess`;
   5. prints the status and calls `GET /api/v9/modules` with a correct and an incorrect key: `200` and `401` are
      expected. The URL used for the check is `--site-url` (default `http://localhost`).
3. Exit code `0` — everything is installed and the API responds; `3` — the package is installed but the self-check failed.

All options: `bin/install.sh --help` (`--web-user`, `--skip-config`, `--skip-check`, `--force`).

#### 4.2. Script: SuiteCRM in a Docker Container

1. Run on the host where `docker` is available:

   ```bash
   bin/install.sh --docker <crm-container> --env-file config/wps.env \
       --apache-conf /etc/apache2/sites-available/000-default.conf
   ```

   `--crm-path` is the path inside the container (default `/var/www/html`), `--apache-conf` is also a path inside the container.
2. The script copies the package to the container's `/tmp` (`docker cp`), performs the same steps as `www-data`
   (`docker exec -u www-data`), and runs the self-check with curl inside the container. Temporary files are removed.
3. The plugin files live in the CRM directory and survive `docker compose restart`. The Apache configuration usually
   lives in the image: after recreating the container (`up --force-recreate`, a new image) the rule from `--apache-conf`
   will be lost. Therefore it is better to put the rewrite into the image configuration (as in the `.tehnical/suiteCRM7`
   stand of the WPS repository) or to install it into `.htaccess` (`--htaccess`, the file lives in the CRM volume).
   Recreating the CRM **volume** also deletes the plugin — in that case the installation must be repeated.

#### 4.3. Manual Installation via Admin → Module Loader

1. Build the zip: `bin/build.sh` → `dist/wps_integration-1.1.0-suitecrm7.zip` (`manifest.php` at the archive root;
   requires `zip` or a PHP CLI with `ext-zip`).
2. Log in to the CRM as an administrator and open **Admin → Module Loader**.
3. **Choose File** → select the zip → **Upload**.
4. In the "WPS Integration for SuiteCRM 7" row click **Install**, then **Commit**. Wait for "Module Installed
   Successfully".
5. Add the parameters to `config_override.php` (Module Loader does not write them). The easiest way is this, from the
   CRM root:

   ```bash
   sudo -u www-data env $(grep -v '^#' /path/to/wps.env | xargs) php /path/to/suiteCRM7/bin/wps_setup.php config
   ```

   or manually — the keys are described in `package/files/custom/include/WPS/WpsConfig.php`.
6. Configure the web server ([Section 5](#5-web-server-configuration)).

> **Note.** In Module Loader, navigate through the Admin menu. Opening the direct link
> `index.php?module=Administration&action=UpgradeWizard` without a Referer triggers SuiteCRM's XSRF protection
> ("Possible Cross Site Request Forgery").

### 5. Web Server Configuration

WPS builds URLs as `{CRM_API_BASE_URL}/{module}/{id}?filter...`, so the CRM must respond on the path `/api/v9/...`.
The entry point is invoked as `index.php?entryPoint=wpsApi&wps_path=...`; the rewrite performs the routing.

**Apache, VirtualHost (recommended)** — `config/apache-wps.conf`, inside `<VirtualHost>`:

```apache
    # BEGIN WPS integration: /api/v9/{module}[/{id}] -> wpsApi entry point (query string is kept)
    RewriteEngine On
    RewriteRule ^/api/v9/(.*)$ /index.php?entryPoint=wpsApi&wps_path=$1 [QSA,PT,L]
    # END WPS integration
```

`mod_rewrite` is required (`a2enmod rewrite`), then `apachectl -t && apachectl -k graceful`.

**Apache, `.htaccess` (if the vhost is unavailable)** — `config/htaccess-wps.conf` or `bin/install.sh --htaccess`.

The block is placed **at the beginning** of `.htaccess`, before `# BEGIN SUITECRM RESTRICTIONS`, because:
- in the generated part of `.htaccess` in SuiteCRM 7 there is a deprecated rule
  `RewriteRule ^api/(.*?)$ lib/API/public/index.php/$1 [L]`, which would otherwise intercept `/api/v9/...`
  (responding with 401 from the OAuth API);
- Admin → Repair → **Rebuild .htaccess File** overwrites everything starting from `# BEGIN SUITECRM RESTRICTIONS`,
  while everything above it is preserved. Verified: after rebuilding `.htaccess` the WPS block remains and the API
  responds 200.

`AllowOverride All` (or at least `FileInfo Options`) is required for the CRM directory.

**nginx** — `config/nginx-wps.conf`, in the `server { }` block before `location`:

```nginx
    rewrite ^/api/v9/(.*)$ /index.php?entryPoint=wpsApi&wps_path=$1 last;
```

This variant was not tested by the package author — verify it with the command from [Section 7](#7-post-installation-verification).

If the CRM is opened under a subpath (for example `/crm`), add the prefix to both parts of the rule.

### 6. WPS_* Parameters

Written to `$sugar_config['wps']` (`config_override.php`) by the `wps_setup.php config` command (which `install.sh` calls).

| Variable | `$sugar_config['wps']` key | Default | Description |
|---|---|---|---|
| `WPS_ENABLED` | `enabled` | `true` | `false` — events are not published (the API keeps working) |
| `WPS_REDIS_HOST` | `redis_host` | `redis-stream` | host of the WPS Redis Stream |
| `WPS_REDIS_PORT` | `redis_port` | `6379` | port |
| `WPS_REDIS_PASSWORD` | `redis_password` | empty | password (`AUTH`) |
| `WPS_REDIS_TIMEOUT` | `redis_timeout` | `1.0` | connection and read timeout in seconds: saving a record does not wait any longer when Redis is unavailable |
| `WPS_REDIS_STREAM` | `stream` | `crm_events` | stream name |
| `WPS_REDIS_STREAM_MAXLEN` | `stream_maxlen` | `100000` | `XADD MAXLEN ~` (0 — unlimited) |
| `WPS_EVENT_MODULES` | `event_modules` | `Leads,Contacts,Accounts,Opportunities,Cases,Calls,Tasks,Meetings` | modules whose saves are published (empty — all) |
| `WPS_API_STATIC_KEY` | `api_static_key` | empty | API key: `X-Auth-Key = key . gmdate("Y.m.d H")`, the previous hour's key is also accepted. An empty key — the API responds 401 to everything |
| `WPS_API_USER_ID` | `api_user_id` | `1` | the CRM user the API acts on behalf of (`created_by`, `modified_user_id`) |
| — | `api_readonly_modules` | `Users, Employees` | modules that cannot be written to (405) |

To change the parameters without reinstalling: `bin/install.sh … --skip-check` or `wps_setup.php config` from the CRM root.

### 7. Post-Installation Verification

```bash
# 1. status: package, hook, entry point, field, parameters, Redis availability (from the CRM root, as www-data)
sudo -u www-data php /path/to/suiteCRM7/bin/wps_setup.php status

# 2. API with the key (the key changes every hour, the time is UTC)
KEY="${WPS_API_STATIC_KEY}$(date -u '+%Y.%m.%d %H')"
curl -s -H "X-Auth-Key: $KEY" http://<crm>/api/v9/modules | head -c 200            # 200 and {"data":[{"type":"modules",...
curl -s -o /dev/null -w '%{http_code}\n' -H 'X-Auth-Key: wrong' http://<crm>/api/v9/modules   # 401
curl -s -g -H "X-Auth-Key: $KEY" "http://<crm>/api/v9/Leads?filter[status]=New&size=1"   # field search (-g: brackets are not a template)

# 3. event: save any lead in CRM and look at the tail of the stream in WPS Redis
redis-cli -h <redis-wps> XREVRANGE crm_events + - COUNT 1
```

### 8. Connecting WPS to This CRM

In the WPS (backend) `.env`:

```bash
CRM_API_BASE_URL=http://<crm>/api/v9      # the path must end with /api/v9
CRM_API_STATIC_KEY=<the same WPS_API_STATIC_KEY>
MOCK_SERVICE_URL=                         # empty: the module dictionary is taken from the CRM
```

Restart the WPS backend and refresh the module dictionary (`POST /api/dictionaries/modules/refresh`).

### 9. Upgrading

Running `bin/install.sh` again with a new package version is an upgrade: the script removes the files of the
installed version (as Module Loader does on upgrade), installs the new one and updates the Module Loader record. The
`wps_processed_c` field with its data and the parameters are preserved. The same run with the same version is a
reinstallation (idempotent).

Via Module Loader: first **Uninstall** the old version (select "Do not remove tables" to keep the field),
then install the new one as in [4.3](#43-manual-installation-via-admin--module-loader).

### 10. Uninstalling

```bash
bin/uninstall.sh --crm-path /var/www/html [--apache-conf <vhost>] [--htaccess] [--keep-config] [--remove-fields]
bin/uninstall.sh --docker <container> [the same options]
```

| What | By default | Option |
|---|---|---|
| `custom/include/WPS` files, logic hook, entry point, labels | removed | — |
| record and zip in Module Loader | removed | — |
| `$sugar_config['wps']` block in `config_override.php` | removed | `--keep-config` — keep |
| `Leads.wps_processed_c` field **with data** | kept | `--remove-fields` — remove |
| rewrite in vhost / `.htaccess` | kept | `--apache-conf <file>` / `--htaccess` — remove the block between the markers |

Via Module Loader: **Uninstall** in the package row; "Remove tables" also removes the field with its data.

### 11. Troubleshooting

| Symptom | Cause | Solution |
|---|---|---|
| `install.sh`: "is not the legacy root of SuiteCRM" | wrong `--crm-path` | specify the CRM root (it contains `include/entryPoint.php`) |
| "looks like SuiteCRM 8: use the SuiteCRM 8 package" | this is SuiteCRM 8 | use the `SuiteCRM8` package |
| "does not support SuiteCRM x.y" | version outside `^7\.14\.` | upgrade the CRM or use `--force` at your own risk |
| self-check: `404` for both keys | rewrite missing or the entry point is not registered | [Section 5](#5-web-server-configuration); `wps_setup.php status` |
| self-check: `401` for both keys | without the rewrite the request went to the deprecated SuiteCRM API (`^api/`), or the key is empty/different | check the rewrite (the block must come before the SuiteCRM rules); `WPS_API_STATIC_KEY`; the server clock (UTC) |
| `POST/PUT` → "Bad data passed in" | body without `Content-Type: application/json` | WPS sends the header; add it in your own requests |
| saving a record is slow, `[WPS] publish failed` in `suitecrm.log` | WPS Redis is unreachable | network / `WPS_REDIS_HOST`; the record is saved but the event is lost (no outbox); timeout — `WPS_REDIS_TIMEOUT` |
| Module Loader: "Possible Cross Site Request Forgery" | a direct URL was opened without a Referer | go through the Admin → Module Loader menu |
| Module Loader: "Duplicate column name 'wps_processed_c'" | installing version 1.0.0 (the field was in the manifest's `custom_fields`) over an already saved field | use 1.1.0+: the field is created by the script only if it does not exist |
| after removal, `custom/include/WPS` files remain | installations by the old installer (1.0.0) were reinstalled without removal, and ModuleInstaller restored the "restore point" | `bin/install.sh` will remove the leftovers ("Removed leftover…"), then run `bin/uninstall.sh` |

Logs: `suitecrm.log` in the CRM root (lines `[WPS]`), the Apache error log.

### 12. Event and API Contract

**Event** (the `data` field of the stream message) — a JSON object: `id`, `module`, `object_type` and all stored
fields of the record, including `_cstm`, re-read from the database (dates — DB format, UTC). Relations, non-db fields
and secrets (`user_hash` etc.) are not included. Service fields: `wps_event` (`after_save`), `wps_is_update` (`1`/`0`),
`wps_user_id`, `wps_published_at`.

**API** (`X-Auth-Key`, JSON:API):

| Request | Response |
|---|---|
| `GET /api/v9/modules` | `{data: [{type: "modules", id: "Leads", attributes: {name, label}}]}` |
| `GET /api/v9/{module}/{id}` | `{data: {type, id, attributes}}`, `404` if not found |
| `GET /api/v9/{module}?filter[f]=v&size=1` | `{data: [...], meta}`; filters are combined with AND, `size` ≤ 100, order `date_entered ASC`; unknown field → `422` |
| `POST /api/v9/{module}` | body `{data: {type, attributes}}` → `201 {data}` |
| `PUT /api/v9/{module}/{id}` | → `200 {data}`, `404` if not found |

Errors: `{errors: [{status, title, detail?}]}` — `400` (malformed JSON), `401` (key), `404`, `405` (method or a
record in `Users`/`Employees`), `422`, `500`. Unknown and system attributes are not written; their list is in
`meta.ignored_attributes`. Dates are accepted as `Y-m-d H:i:s` (UTC) or ISO 8601.

---

## Русский

Плагин интеграции SuiteCRM 7 с **Workflow Processing Service (WPS)** — сервисом автоматизации бизнес-процессов. Пакет самодостаточен: в нём только код для CRM, установщики и документация.

- [1. Назначение и состав](#1-назначение-и-состав)
- [2. Совместимость](#2-совместимость)
- [3. Требования](#3-требования)
- [4. Установка](#4-установка)
- [5. Настройка веб-сервера](#5-настройка-веб-сервера)
- [6. Параметры WPS_*](#6-параметры-wps_)
- [7. Проверка после установки](#7-проверка-после-установки)
- [8. Подключение WPS к этой CRM](#8-подключение-wps-к-этой-crm)
- [9. Обновление](#9-обновление)
- [10. Удаление](#10-удаление)
- [11. Диагностика](#11-диагностика)
- [12. Контракт события и API](#12-контракт-события-и-api)

### 1. Назначение и состав

Плагин связывает CRM и WPS в обе стороны:

| Направление | Как сделано |
|---|---|
| CRM → WPS | Logic hook `after_save` уровня приложения (`WpsEventHook`) публикует сохранённую запись в Redis Stream WPS: `XADD crm_events MAXLEN ~ 100000 * data '{json}'`. Клиент Redis написан на чистом PHP (`WpsRedisClient`), расширение phpredis не нужно. |
| WPS → CRM | Custom entry point `wpsApi` (`WpsApi`) отдаёт API с контрактом WPS: `GET/POST/PUT /api/v9/{module}[/{id}]`, поиск `?filter[field]=value&size=1`, справочник `/api/v9/modules`. Путь `/api/v9/...` переводится на entry point правилом rewrite веб-сервера. |
| Маркер | Поле `Leads.wps_processed_c` («Обработан WPS»), его ставят процессы WPS. |

Ядро SuiteCRM не изменяется: файлы ставятся в `custom/` через Extension-фреймворк, пакет виден в
Admin → Module Loader и удаляется оттуда.

```
suiteCRM7/
├── README.md                 этот файл
├── CHANGELOG.md
├── VERSION                   версия пакета (= manifest.php → version)
├── package/                  содержимое zip для Module Loader (manifest.php в корне)
│   ├── manifest.php
│   ├── files/custom/include/WPS/   WpsApi, WpsEventHook, WpsRedisClient, WpsBeanSerializer, WpsConfig, entry point
│   ├── files/extension/            регистрация logic hook и entry point, подписи поля (en_us, ru_RU)
│   └── scripts/                    создание / удаление поля Leads.wps_processed_c (идемпотентно)
├── bin/
│   ├── install.sh            автоматическая установка и переустановка
│   ├── uninstall.sh          удаление
│   ├── build.sh              сборка dist/wps_integration-<версия>-suitecrm7.zip
│   └── wps_setup.php         CLI: config / install / uninstall / status (вызывается из install.sh и uninstall.sh)
├── config/
│   ├── wps.env.example       все параметры WPS_* с описанием
│   ├── apache-wps.conf       rewrite /api/v9 для VirtualHost
│   ├── htaccess-wps.conf     rewrite /api/v9 для .htaccess (если vhost недоступен)
│   └── nginx-wps.conf        rewrite /api/v9 для nginx
└── dist/                     результат build.sh
```

### 2. Совместимость

| | Версия |
|---|---|
| SuiteCRM | 7.14.x (проверено на 7.14.9). `manifest.php` принимает `^7\.14\.` |
| PHP | 8.1, 8.2 (как у SuiteCRM 7.14; проверено на 8.2) + расширение `zip` |
| БД | MariaDB 10.4–10.11 / MySQL 5.7–8.0 (как у SuiteCRM 7.14; проверено на MariaDB 10.11) |
| Веб-сервер | Apache 2.4 с `mod_rewrite` (проверено); nginx — фрагмент приложен, не проверялся |
| WPS | контракт CRM API `/api/v9`, Redis Stream `crm_events` |

Для SuiteCRM 8 есть отдельный пакет `SuiteCRM8` — этот пакет установщик в SuiteCRM 8 не поставит.
Чтобы поставить пакет в другую 7.x на свой риск, используйте `bin/install.sh --force` (Module Loader откажет).

### 3. Требования

- SuiteCRM установлена (в `config.php` есть `'installer_locked' => true`).
- Из CRM доступен Redis Stream WPS (по умолчанию `redis-stream:6379`), а из WPS — веб-сервер CRM.
- Для `bin/install.sh`: bash, PHP CLI той же версии, что у CRM, права root, пользователя веб-сервера (`www-data`)
  или `sudo`; для режима Docker — `docker` на хосте.
- Пользователь веб-сервера может писать в `custom/`, `cache/`, `upload/` и `config_override.php`.

### 4. Установка

Сначала подготовьте параметры (секреты не попадают ни в пакет, ни в командную строку):

```bash
cp config/wps.env.example config/wps.env
# задайте WPS_API_STATIC_KEY (= CRM_API_STATIC_KEY в WPS) и адрес Redis WPS
```

Параметры, уже заданные в окружении, важнее файла. Другой файл передаётся флагом `--env-file`.

#### 4.1. Скрипт: SuiteCRM на этой машине

1. Выполните из каталога пакета (от root, от `www-data` или через `sudo`):

   ```bash
   bin/install.sh --crm-path /var/www/html --apache-conf /etc/apache2/sites-available/000-default.conf
   ```

2. Скрипт по шагам:
   1. проверяет, что это установленная SuiteCRM 7, есть PHP `zip` и права на запись;
   2. пишет `$sugar_config['wps']` в `config_override.php`;
   3. ставит пакет через `ModuleInstaller` и регистрирует его в Module Loader (`upgrade_history`), создаёт поле
      `Leads.wps_processed_c`, если его ещё нет;
   4. добавляет rewrite `/api/v9` в указанный VirtualHost между маркерами `# BEGIN/END WPS integration`, проверяет
      конфигурацию (`apachectl -t`) и перезагружает Apache. Без `--apache-conf` фрагмент только печатается
      (см. [раздел 5](#5-настройка-веб-сервера)), а с `--htaccess` ставится в `.htaccess`;
   5. печатает статус и вызывает `GET /api/v9/modules` с правильным и неправильным ключом: ожидается `200` и `401`.
      Адрес для проверки — `--site-url` (по умолчанию `http://localhost`).
3. Код выхода `0` — всё установлено и API отвечает; `3` — пакет установлен, но самопроверка не прошла.

Все опции: `bin/install.sh --help` (`--web-user`, `--skip-config`, `--skip-check`, `--force`).

#### 4.2. Скрипт: SuiteCRM в Docker-контейнере

1. Выполните на хосте, где есть `docker`:

   ```bash
   bin/install.sh --docker <контейнер-с-crm> --env-file config/wps.env \
       --apache-conf /etc/apache2/sites-available/000-default.conf
   ```

   `--crm-path` — путь внутри контейнера (по умолчанию `/var/www/html`), `--apache-conf` — тоже путь внутри контейнера.
2. Скрипт копирует пакет в `/tmp` контейнера (`docker cp`), выполняет те же шаги от `www-data`
   (`docker exec -u www-data`), а самопроверку — curl внутри контейнера. Временные файлы удаляются.
3. Файлы плагина лежат в каталоге CRM и переживают `docker compose restart`. Конфигурация Apache обычно живёт в образе:
   после пересоздания контейнера (`up --force-recreate`, новый образ) правило из `--apache-conf` пропадёт. Поэтому
   rewrite лучше положить в конфигурацию образа (как в стенде `.tehnical/suiteCRM7` репозитория WPS) или поставить его
   в `.htaccess` (`--htaccess`, файл лежит в томе с CRM). Пересоздание **тома** CRM удаляет и плагин — тогда установку
   нужно повторить.

#### 4.3. Установка вручную через Admin → Module Loader

1. Соберите zip: `bin/build.sh` → `dist/wps_integration-1.1.0-suitecrm7.zip` (`manifest.php` в корне архива;
   нужен `zip` или PHP CLI с `ext-zip`).
2. Войдите в CRM администратором, откройте **Admin → Module Loader**.
3. **Choose File** → выберите zip → **Upload**.
4. В строке «WPS Integration for SuiteCRM 7» нажмите **Install**, затем **Commit**. Дождитесь «Module Installed
   Successfully».
5. Добавьте параметры в `config_override.php` (Module Loader их не пишет). Удобнее всего так, из корня CRM:

   ```bash
   sudo -u www-data env $(grep -v '^#' /path/to/wps.env | xargs) php /path/to/suiteCRM7/bin/wps_setup.php config
   ```

   или вручную — ключи описаны в `package/files/custom/include/WPS/WpsConfig.php`.
6. Настройте веб-сервер ([раздел 5](#5-настройка-веб-сервера)).

> **Внимание.** В Module Loader переходите через меню Admin. Прямая ссылка на `index.php?module=Administration&action=UpgradeWizard`
> без Referer вызывает защиту SuiteCRM от XSRF («Possible Cross Site Request Forgery»).

### 5. Настройка веб-сервера

WPS собирает URL как `{CRM_API_BASE_URL}/{module}/{id}?filter...`, поэтому CRM должна отвечать на путь `/api/v9/...`.
Entry point вызывается как `index.php?entryPoint=wpsApi&wps_path=...`; перевод делает rewrite.

**Apache, VirtualHost (рекомендуется)** — `config/apache-wps.conf`, внутрь `<VirtualHost>`:

```apache
    # BEGIN WPS integration: /api/v9/{module}[/{id}] -> wpsApi entry point (query string is kept)
    RewriteEngine On
    RewriteRule ^/api/v9/(.*)$ /index.php?entryPoint=wpsApi&wps_path=$1 [QSA,PT,L]
    # END WPS integration
```

Нужен `mod_rewrite` (`a2enmod rewrite`), затем `apachectl -t && apachectl -k graceful`.

**Apache, `.htaccess` (если vhost недоступен)** — `config/htaccess-wps.conf` или `bin/install.sh --htaccess`.

Блок ставится **в начало** `.htaccess`, перед `# BEGIN SUITECRM RESTRICTIONS`, потому что:
- в генерируемой части `.htaccess` SuiteCRM 7 есть устаревшее правило `RewriteRule ^api/(.*?)$ lib/API/public/index.php/$1 [L]`,
  которое иначе перехватит `/api/v9/...` (ответит 401 от OAuth API);
- Admin → Repair → **Rebuild .htaccess File** перезаписывает всё начиная с `# BEGIN SUITECRM RESTRICTIONS`,
  а то, что выше, сохраняет. Проверено: после пересборки `.htaccess` блок WPS остаётся и API отвечает 200.

Нужен `AllowOverride All` (или как минимум `FileInfo Options`) для каталога CRM.

**nginx** — `config/nginx-wps.conf`, в блок `server { }` перед `location`:

```nginx
    rewrite ^/api/v9/(.*)$ /index.php?entryPoint=wpsApi&wps_path=$1 last;
```

Этот вариант не проверялся автором пакета — проверьте командой из [раздела 7](#7-проверка-после-установки).

Если CRM открыта по подпути (например, `/crm`), добавьте префикс в обе части правила.

### 6. Параметры WPS_*

Записываются в `$sugar_config['wps']` (`config_override.php`) командой `wps_setup.php config` (её вызывает `install.sh`).

| Переменная | Ключ `$sugar_config['wps']` | По умолчанию | Описание |
|---|---|---|---|
| `WPS_ENABLED` | `enabled` | `true` | `false` — события не публикуются (API продолжает работать) |
| `WPS_REDIS_HOST` | `redis_host` | `redis-stream` | хост Redis Stream WPS |
| `WPS_REDIS_PORT` | `redis_port` | `6379` | порт |
| `WPS_REDIS_PASSWORD` | `redis_password` | пусто | пароль (`AUTH`) |
| `WPS_REDIS_TIMEOUT` | `redis_timeout` | `1.0` | таймаут подключения и чтения, секунды: дольше сохранение записи при недоступном Redis не ждёт |
| `WPS_REDIS_STREAM` | `stream` | `crm_events` | имя стрима |
| `WPS_REDIS_STREAM_MAXLEN` | `stream_maxlen` | `100000` | `XADD MAXLEN ~` (0 — без ограничения) |
| `WPS_EVENT_MODULES` | `event_modules` | `Leads,Contacts,Accounts,Opportunities,Cases,Calls,Tasks,Meetings` | модули, чьи сохранения публикуются (пусто — все) |
| `WPS_API_STATIC_KEY` | `api_static_key` | пусто | ключ API: `X-Auth-Key = ключ . gmdate("Y.m.d H")`, принимается и ключ прошлого часа. Пустой ключ — API отвечает 401 на всё |
| `WPS_API_USER_ID` | `api_user_id` | `1` | пользователь CRM, от имени которого работает API (`created_by`, `modified_user_id`) |
| — | `api_readonly_modules` | `Users, Employees` | модули, запись в которые запрещена (405) |

Изменить параметры без переустановки: `bin/install.sh … --skip-check` или `wps_setup.php config` из корня CRM.

### 7. Проверка после установки

```bash
# 1. статус: пакет, хук, entry point, поле, параметры, доступность Redis (из корня CRM, от www-data)
sudo -u www-data php /path/to/suiteCRM7/bin/wps_setup.php status

# 2. API с ключом (ключ меняется каждый час, время — UTC)
KEY="${WPS_API_STATIC_KEY}$(date -u '+%Y.%m.%d %H')"
curl -s -H "X-Auth-Key: $KEY" http://<crm>/api/v9/modules | head -c 200            # 200 и {"data":[{"type":"modules",...
curl -s -o /dev/null -w '%{http_code}\n' -H 'X-Auth-Key: wrong' http://<crm>/api/v9/modules   # 401
curl -s -g -H "X-Auth-Key: $KEY" "http://<crm>/api/v9/Leads?filter[status]=New&size=1"   # поиск по полю (-g: скобки не шаблон)

# 3. событие: сохраните любой лид в CRM и посмотрите хвост стрима в Redis WPS
redis-cli -h <redis-wps> XREVRANGE crm_events + - COUNT 1
```

### 8. Подключение WPS к этой CRM

В `.env` WPS (backend):

```bash
CRM_API_BASE_URL=http://<crm>/api/v9      # путь должен заканчиваться на /api/v9
CRM_API_STATIC_KEY=<тот же WPS_API_STATIC_KEY>
MOCK_SERVICE_URL=                         # пусто: справочник модулей берётся из CRM
```

Перезапустите backend WPS и обновите справочник модулей (`POST /api/dictionaries/modules/refresh`).

### 9. Обновление

Повторный запуск `bin/install.sh` с новой версией пакета — это обновление: скрипт удаляет файлы установленной версии
(как Module Loader при обновлении), ставит новую и обновляет запись в Module Loader. Поле `wps_processed_c` с данными и
параметры сохраняются. Тот же запуск с той же версией — переустановка (идемпотентно).

Через Module Loader: сначала **Uninstall** старой версии (выберите «Do not remove tables», чтобы сохранить поле),
затем установка новой, как в [4.3](#43-установка-вручную-через-admin--module-loader).

### 10. Удаление

```bash
bin/uninstall.sh --crm-path /var/www/html [--apache-conf <vhost>] [--htaccess] [--keep-config] [--remove-fields]
bin/uninstall.sh --docker <container> [те же опции]
```

| Что | По умолчанию | Опция |
|---|---|---|
| файлы `custom/include/WPS`, logic hook, entry point, подписи | удаляются | — |
| запись и zip в Module Loader | удаляются | — |
| блок `$sugar_config['wps']` в `config_override.php` | удаляется | `--keep-config` — оставить |
| поле `Leads.wps_processed_c` **с данными** | остаётся | `--remove-fields` — удалить |
| rewrite в vhost / `.htaccess` | остаётся | `--apache-conf <файл>` / `--htaccess` — удалить блок между маркерами |

Через Module Loader: **Uninstall** в строке пакета; «Remove tables» удаляет и поле с данными.

### 11. Диагностика

| Симптом | Причина | Решение |
|---|---|---|
| `install.sh`: «is not the legacy root of SuiteCRM» | неверный `--crm-path` | укажите корень CRM (там `include/entryPoint.php`) |
| «looks like SuiteCRM 8: use the SuiteCRM 8 package» | это SuiteCRM 8 | используйте пакет `SuiteCRM8` |
| «does not support SuiteCRM x.y» | версия вне `^7\.14\.` | обновите CRM или `--force` на свой риск |
| самопроверка: `404` на оба ключа | нет rewrite или entry point не зарегистрирован | [раздел 5](#5-настройка-веб-сервера); `wps_setup.php status` |
| самопроверка: `401` на оба ключа | без rewrite запрос ушёл в устаревший API SuiteCRM (`^api/`), либо пустой/другой ключ | проверьте rewrite (блок должен стоять до правил SuiteCRM); `WPS_API_STATIC_KEY`; часы сервера (UTC) |
| `POST/PUT` → «Bad data passed in» | тело без `Content-Type: application/json` | WPS отправляет заголовок; в своих запросах добавьте его |
| сохранение записи медленное, в `suitecrm.log` `[WPS] publish failed` | Redis WPS недоступен | сеть / `WPS_REDIS_HOST`; запись сохраняется, событие теряется (outbox нет); таймаут — `WPS_REDIS_TIMEOUT` |
| Module Loader: «Possible Cross Site Request Forgery» | открыт прямой URL без Referer | заходите через меню Admin → Module Loader |
| Module Loader: «Duplicate column name 'wps_processed_c'» | установка версии 1.0.0 (поле было в `custom_fields` manifest) поверх сохранённого поля | используйте 1.1.0+: поле создаётся скриптом только если его нет |
| после удаления остались файлы `custom/include/WPS` | установки старым установщиком (1.0.0) переустанавливались без удаления, и ModuleInstaller восстановил «restore point» | `bin/install.sh` удалит остатки («Removed leftover…»), затем `bin/uninstall.sh` |

Логи: `suitecrm.log` в корне CRM (строки `[WPS]`), лог ошибок Apache.

### 12. Контракт события и API

**Событие** (поле `data` сообщения стрима) — JSON-объект: `id`, `module`, `object_type` и все хранимые поля записи,
включая `_cstm`, перечитанные из БД (даты — формат БД, UTC). Связи, non-db поля и секреты (`user_hash` и т. п.)
не попадают. Служебные поля: `wps_event` (`after_save`), `wps_is_update` (`1`/`0`), `wps_user_id`, `wps_published_at`.

**API** (`X-Auth-Key`, JSON:API):

| Запрос | Ответ |
|---|---|
| `GET /api/v9/modules` | `{data: [{type: "modules", id: "Leads", attributes: {name, label}}]}` |
| `GET /api/v9/{module}/{id}` | `{data: {type, id, attributes}}`, `404` если нет |
| `GET /api/v9/{module}?filter[f]=v&size=1` | `{data: [...], meta}`; фильтры по И, `size` ≤ 100, порядок `date_entered ASC`; неизвестное поле → `422` |
| `POST /api/v9/{module}` | тело `{data: {type, attributes}}` → `201 {data}` |
| `PUT /api/v9/{module}/{id}` | → `200 {data}`, `404` если нет |

Ошибки: `{errors: [{status, title, detail?}]}` — `400` (битый JSON), `401` (ключ), `404`, `405` (метод или запись в
`Users`/`Employees`), `422`, `500`. Неизвестные и системные атрибуты не записываются, их список — в
`meta.ignored_attributes`. Даты принимаются как `Y-m-d H:i:s` (UTC) или ISO 8601.
