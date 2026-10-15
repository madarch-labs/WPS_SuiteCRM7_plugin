# WPS Integration for SuiteCRM 7

**EN:** A SuiteCRM 7 plugin integrating with **Workflow Processing Service (WPS)** — a business process automation service. The package is self-contained: it contains only the CRM code, installers and documentation.

**RU:** Плагин интеграции SuiteCRM 7 с **Workflow Processing Service (WPS)** — сервисом автоматизации бизнес-процессов. Пакет самодостаточен: в нём только код для CRM, установщики и документация.

- [1. Purpose and Scope / Назначение и состав](#1-purpose-and-scope--назначение-и-состав)
- [2. Compatibility / Совместимость](#2-compatibility--совместимость)
- [3. Requirements / Требования](#3-requirements--требования)
- [4. Installation / Установка](#4-installation--установка)
- [5. Web Server Configuration / Настройка веб-сервера](#5-web-server-configuration--настройка-веб-сервера)
- [6. WPS_* Parameters / Параметры WPS_*](#6-wps_-parameters--параметры-wps_)
- [7. Post-Installation Verification / Проверка после установки](#7-post-installation-verification--проверка-после-установки)
- [8. Connecting WPS to This CRM / Подключение WPS к этой CRM](#8-connecting-wps-to-this-crm--подключение-wps-к-этой-crm)
- [9. Upgrading / Обновление](#9-upgrading--обновление)
- [10. Uninstalling / Удаление](#10-uninstalling--удаление)
- [11. Troubleshooting / Диагностика](#11-troubleshooting--диагностика)
- [12. Event and API Contract / Контракт события и API](#12-event-and-api-contract--контракт-события-и-api)

## 1. Purpose and Scope / Назначение и состав

**EN:** The plugin connects the CRM and WPS in both directions:

| Direction / Направление | How it is implemented / Как сделано |
|---|---|
| CRM → WPS | EN: application-level `after_save` logic hook (`WpsEventHook`) publishes the saved record to the WPS Redis Stream: `XADD crm_events MAXLEN ~ 100000 * data '{json}'`. The Redis client is written in pure PHP (`WpsRedisClient`), the phpredis extension is not required.<br>RU: logic hook `after_save` уровня приложения (`WpsEventHook`) публикует сохранённую запись в Redis Stream WPS: `XADD crm_events MAXLEN ~ 100000 * data '{json}'`. Клиент Redis написан на чистом PHP (`WpsRedisClient`), расширение phpredis не нужно |
| WPS → CRM | EN: custom entry point `wpsApi` (`WpsApi`) exposes an API with the WPS contract: `GET/POST/PUT /api/v9/{module}[/{id}]`, search `?filter[field]=value&size=1`, dictionary `/api/v9/modules`. The `/api/v9/...` path is routed to the entry point by a web server rewrite rule.<br>RU: custom entry point `wpsApi` (`WpsApi`) отдаёт API с контрактом WPS: `GET/POST/PUT /api/v9/{module}[/{id}]`, поиск `?filter[field]=value&size=1`, справочник `/api/v9/modules`. Путь `/api/v9/...` переводится на entry point правилом rewrite веб-сервера |
| Marker / Маркер | EN: the `Leads.wps_processed_c` field ("Processed by WPS"), set by WPS processes.<br>RU: поле `Leads.wps_processed_c` («Обработан WPS»), его ставят процессы WPS |

**EN:** The SuiteCRM core is not modified: files are installed into `custom/` via the Extension framework, the package appears in Admin → Module Loader and is removed from there.

**RU:** Ядро SuiteCRM не изменяется: файлы ставятся в `custom/` через Extension-фреймворк, пакет виден в
Admin → Module Loader и удаляется оттуда.

```
suiteCRM7/
├── README.md                 this file / этот файл
├── CHANGELOG.md
├── VERSION                   package version (= manifest.php → version) / версия пакета (= manifest.php → version)
├── package/                  contents of the zip for Module Loader (manifest.php at the root) / содержимое zip для Module Loader (manifest.php в корне)
│   ├── manifest.php
│   ├── files/custom/include/WPS/   WpsApi, WpsEventHook, WpsRedisClient, WpsBeanSerializer, WpsConfig, entry point
│   ├── files/extension/            logic hook and entry point registration, field labels (en_us, ru_RU) / регистрация logic hook и entry point, подписи поля (en_us, ru_RU)
│   └── scripts/                    creation / removal of Leads.wps_processed_c (idempotent) / создание / удаление поля Leads.wps_processed_c (идемпотентно)
├── bin/
│   ├── install.sh            automatic installation and reinstallation / автоматическая установка и переустановка
│   ├── uninstall.sh          removal / удаление
│   ├── build.sh              builds dist/wps_integration-<version>-suitecrm7.zip / сборка dist/wps_integration-<версия>-suitecrm7.zip
│   └── wps_setup.php         CLI: config / install / uninstall / status (called from install.sh and uninstall.sh) / вызывается из install.sh и uninstall.sh
├── config/
│   ├── wps.env.example       all WPS_* parameters with descriptions / все параметры WPS_* с описанием
│   ├── apache-wps.conf       /api/v9 rewrite for a VirtualHost / rewrite /api/v9 для VirtualHost
│   ├── htaccess-wps.conf     /api/v9 rewrite for .htaccess (if the vhost is unavailable) / rewrite /api/v9 для .htaccess (если vhost недоступен)
│   └── nginx-wps.conf        /api/v9 rewrite for nginx / rewrite /api/v9 для nginx
└── dist/                     output of build.sh / результат build.sh
```

## 2. Compatibility / Совместимость

| | Version / Версия |
|---|---|
| SuiteCRM | EN: 7.14.x (tested on 7.14.9). `manifest.php` accepts `^7\.14\.`<br>RU: 7.14.x (проверено на 7.14.9). `manifest.php` принимает `^7\.14\.` |
| PHP | EN: 8.1, 8.2 (same as SuiteCRM 7.14; tested on 8.2) + the `zip` extension<br>RU: 8.1, 8.2 (как у SuiteCRM 7.14; проверено на 8.2) + расширение `zip` |
| Database / БД | EN: MariaDB 10.4–10.11 / MySQL 5.7–8.0 (same as SuiteCRM 7.14; tested on MariaDB 10.11)<br>RU: MariaDB 10.4–10.11 / MySQL 5.7–8.0 (как у SuiteCRM 7.14; проверено на MariaDB 10.11) |
| Web server / Веб-сервер | EN: Apache 2.4 with `mod_rewrite` (tested); nginx — the snippet is provided, not tested<br>RU: Apache 2.4 с `mod_rewrite` (проверено); nginx — фрагмент приложен, не проверялся |
| WPS | EN: CRM API contract `/api/v9`, Redis Stream `crm_events`<br>RU: контракт CRM API `/api/v9`, Redis Stream `crm_events` |

**EN:** For SuiteCRM 8 there is a separate `SuiteCRM8` package — this package will not be installed by the SuiteCRM 8 installer. To install this package in another 7.x at your own risk, use `bin/install.sh --force` (Module Loader will refuse).

**RU:** Для SuiteCRM 8 есть отдельный пакет `SuiteCRM8` — этот пакет установщик в SuiteCRM 8 не поставит.
Чтобы поставить пакет в другую 7.x на свой риск, используйте `bin/install.sh --force` (Module Loader откажет).

## 3. Requirements / Требования

**EN:**
- SuiteCRM is installed (`config.php` contains `'installer_locked' => true`).
- The WPS Redis Stream is reachable from the CRM (default `redis-stream:6379`), and the CRM web server is reachable from WPS.
- For `bin/install.sh`: bash, a PHP CLI of the same version as the CRM, root privileges, the web server user (`www-data`)
  or `sudo`; for the Docker mode — `docker` on the host.
- The web server user can write to `custom/`, `cache/`, `upload/` and `config_override.php`.

**RU:**
- SuiteCRM установлена (в `config.php` есть `'installer_locked' => true`).
- Из CRM доступен Redis Stream WPS (по умолчанию `redis-stream:6379`), а из WPS — веб-сервер CRM.
- Для `bin/install.sh`: bash, PHP CLI той же версии, что у CRM, права root, пользователя веб-сервера (`www-data`)
  или `sudo`; для режима Docker — `docker` на хосте.
- Пользователь веб-сервера может писать в `custom/`, `cache/`, `upload/` и `config_override.php`.

## 4. Installation / Установка

**EN:** First prepare the parameters (secrets end up neither in the package nor on the command line):

**RU:** Сначала подготовьте параметры (секреты не попадают ни в пакет, ни в командную строку):

```bash
cp config/wps.env.example config/wps.env
# set WPS_API_STATIC_KEY (= CRM_API_STATIC_KEY in WPS) and the WPS Redis address
# задайте WPS_API_STATIC_KEY (= CRM_API_STATIC_KEY в WPS) и адрес Redis WPS
```

**EN:** Parameters already set in the environment take precedence over the file. A different file is passed with the `--env-file` flag.

**RU:** Параметры, уже заданные в окружении, важнее файла. Другой файл передаётся флагом `--env-file`.

### 4.1. Script: SuiteCRM on This Machine / Скриптом: SuiteCRM на этой машине

**EN:**
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
      (see [Section 5](#5-web-server-configuration--настройка-веб-сервера)), and with `--htaccess` it is installed into `.htaccess`;
   5. prints the status and calls `GET /api/v9/modules` with a correct and an incorrect key: `200` and `401` are
      expected. The URL used for the check is `--site-url` (default `http://localhost`).
3. Exit code `0` — everything is installed and the API responds; `3` — the package is installed but the self-check failed.

**RU:**
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
      (см. [раздел 5](#5-web-server-configuration--настройка-веб-сервера)), а с `--htaccess` ставится в `.htaccess`;
   5. печатает статус и вызывает `GET /api/v9/modules` с правильным и неправильным ключом: ожидается `200` и `401`.
      Адрес для проверки — `--site-url` (по умолчанию `http://localhost`).
3. Код выхода `0` — всё установлено и API отвечает; `3` — пакет установлен, но самопроверка не прошла.

**EN:** All options: `bin/install.sh --help` (`--web-user`, `--skip-config`, `--skip-check`, `--force`).

**RU:** Все опции: `bin/install.sh --help` (`--web-user`, `--skip-config`, `--skip-check`, `--force`).

### 4.2. Script: SuiteCRM in a Docker Container / Скриптом: SuiteCRM в Docker-контейнере

**EN:**
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

**RU:**
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

### 4.3. Manual Installation via Admin → Module Loader / Вручную через Admin → Module Loader

**EN:**
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
6. Configure the web server ([Section 5](#5-web-server-configuration--настройка-веб-сервера)).

**RU:**
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
6. Настройте веб-сервер ([раздел 5](#5-web-server-configuration--настройка-веб-сервера)).

> **Note / Внимание.** EN: In Module Loader, navigate through the Admin menu. Opening the direct link
> `index.php?module=Administration&action=UpgradeWizard` without a Referer triggers SuiteCRM's XSRF protection
> ("Possible Cross Site Request Forgery").
>
> RU: В Module Loader переходите через меню Admin. Прямая ссылка на `index.php?module=Administration&action=UpgradeWizard`
> без Referer вызывает защиту SuiteCRM от XSRF («Possible Cross Site Request Forgery»).

## 5. Web Server Configuration / Настройка веб-сервера

**EN:** WPS builds URLs as `{CRM_API_BASE_URL}/{module}/{id}?filter...`, so the CRM must respond on the path `/api/v9/...`.
The entry point is invoked as `index.php?entryPoint=wpsApi&wps_path=...`; the rewrite performs the routing.

**RU:** WPS собирает URL как `{CRM_API_BASE_URL}/{module}/{id}?filter...`, поэтому CRM должна отвечать на путь `/api/v9/...`.
Entry point вызывается как `index.php?entryPoint=wpsApi&wps_path=...`; перевод делает rewrite.

**Apache, VirtualHost (recommended / рекомендуется)** — `config/apache-wps.conf`, inside `<VirtualHost>` / внутрь `<VirtualHost>`:

```apache
    # BEGIN WPS integration: /api/v9/{module}[/{id}] -> wpsApi entry point (query string is kept)
    RewriteEngine On
    RewriteRule ^/api/v9/(.*)$ /index.php?entryPoint=wpsApi&wps_path=$1 [QSA,PT,L]
    # END WPS integration
```

**EN:** `mod_rewrite` is required (`a2enmod rewrite`), then `apachectl -t && apachectl -k graceful`.

**RU:** Нужен `mod_rewrite` (`a2enmod rewrite`), затем `apachectl -t && apachectl -k graceful`.

**Apache, `.htaccess` (if the vhost is unavailable / если vhost недоступен)** — `config/htaccess-wps.conf` or `bin/install.sh --htaccess`.
**EN:** The block is placed **at the beginning** of `.htaccess`, before `# BEGIN SUITECRM RESTRICTIONS`, because:
- in the generated part of `.htaccess` in SuiteCRM 7 there is a deprecated rule
  `RewriteRule ^api/(.*?)$ lib/API/public/index.php/$1 [L]`, which would otherwise intercept `/api/v9/...`
  (responding with 401 from the OAuth API);
- Admin → Repair → **Rebuild .htaccess File** overwrites everything starting from `# BEGIN SUITECRM RESTRICTIONS`,
  while everything above it is preserved. Verified: after rebuilding `.htaccess` the WPS block remains and the API
  responds 200.

**RU:** Блок ставится **в начало** `.htaccess`, перед `# BEGIN SUITECRM RESTRICTIONS`, потому что:
- в генерируемой части `.htaccess` SuiteCRM 7 есть устаревшее правило `RewriteRule ^api/(.*?)$ lib/API/public/index.php/$1 [L]`,
  которое иначе перехватит `/api/v9/...` (ответит 401 от OAuth API);
- Admin → Repair → **Rebuild .htaccess File** перезаписывает всё начиная с `# BEGIN SUITECRM RESTRICTIONS`,
  а то, что выше, сохраняет. Проверено: после пересборки `.htaccess` блок WPS остаётся и API отвечает 200.

**EN:** `AllowOverride All` (or at least `FileInfo Options`) is required for the CRM directory.

**RU:** Нужен `AllowOverride All` (или как минимум `FileInfo Options`) для каталога CRM.

**nginx** — `config/nginx-wps.conf`, in the `server { }` block before `location` / в блок `server { }` перед `location`:

```nginx
    rewrite ^/api/v9/(.*)$ /index.php?entryPoint=wpsApi&wps_path=$1 last;
```

**EN:** This variant was not tested by the package author — verify it with the command from [Section 7](#7-post-installation-verification--проверка-после-установки).

**RU:** Этот вариант не проверялся автором пакета — проверьте командой из [раздела 7](#7-post-installation-verification--проверка-после-установки).

**EN:** If the CRM is opened under a subpath (for example `/crm`), add the prefix to both parts of the rule.

**RU:** Если CRM открыта по подпути (например, `/crm`), добавьте префикс в обе части правила.

## 6. WPS_* Parameters / Параметры WPS_*

**EN:** Written to `$sugar_config['wps']` (`config_override.php`) by the `wps_setup.php config` command (which `install.sh` calls).

**RU:** Записываются в `$sugar_config['wps']` (`config_override.php`) командой `wps_setup.php config` (её вызывает `install.sh`).

| Variable / Переменная | `$sugar_config['wps']` key / Ключ | Default / По умолчанию | Description / Описание |
|---|---|---|---|
| `WPS_ENABLED` | `enabled` | `true` | EN: `false` — events are not published (the API keeps working)<br>RU: `false` — события не публикуются (API продолжает работать) |
| `WPS_REDIS_HOST` | `redis_host` | `redis-stream` | EN: host of the WPS Redis Stream<br>RU: хост Redis Stream WPS |
| `WPS_REDIS_PORT` | `redis_port` | `6379` | EN: port<br>RU: порт |
| `WPS_REDIS_PASSWORD` | `redis_password` | empty / пусто | EN: password (`AUTH`)<br>RU: пароль (`AUTH`) |
| `WPS_REDIS_TIMEOUT` | `redis_timeout` | `1.0` | EN: connection and read timeout in seconds: saving a record does not wait any longer when Redis is unavailable<br>RU: таймаут подключения и чтения, секунды: дольше сохранение записи при недоступном Redis не ждёт |
| `WPS_REDIS_STREAM` | `stream` | `crm_events` | EN: stream name<br>RU: имя стрима |
| `WPS_REDIS_STREAM_MAXLEN` | `stream_maxlen` | `100000` | EN: `XADD MAXLEN ~` (0 — unlimited)<br>RU: `XADD MAXLEN ~` (0 — без ограничения) |
| `WPS_EVENT_MODULES` | `event_modules` | `Leads,Contacts,Accounts,Opportunities,Cases,Calls,Tasks,Meetings` | EN: modules whose saves are published (empty — all)<br>RU: модули, чьи сохранения публикуются (пусто — все) |
| `WPS_API_STATIC_KEY` | `api_static_key` | empty / пусто | EN: API key: `X-Auth-Key = key . gmdate("Y.m.d H")`, the previous hour's key is also accepted. An empty key — the API responds 401 to everything<br>RU: ключ API: `X-Auth-Key = ключ . gmdate("Y.m.d H")`, принимается и ключ прошлого часа. Пустой ключ — API отвечает 401 на всё |
| `WPS_API_USER_ID` | `api_user_id` | `1` | EN: the CRM user the API acts on behalf of (`created_by`, `modified_user_id`)<br>RU: пользователь CRM, от имени которого работает API (`created_by`, `modified_user_id`) |
| — | `api_readonly_modules` | `Users, Employees` | EN: modules that cannot be written to (405)<br>RU: модули, запись в которые запрещена (405) |

**EN:** To change the parameters without reinstalling: `bin/install.sh … --skip-check` or `wps_setup.php config` from the CRM root.

**RU:** Изменить параметры без переустановки: `bin/install.sh … --skip-check` или `wps_setup.php config` из корня CRM.

## 7. Post-Installation Verification / Проверка после установки

```bash
# 1. status: package, hook, entry point, field, parameters, Redis availability (from the CRM root, as www-data)
# 1. статус: пакет, хук, entry point, поле, параметры, доступность Redis (из корня CRM, от www-data)
sudo -u www-data php /path/to/suiteCRM7/bin/wps_setup.php status

# 2. API with the key (the key changes every hour, the time is UTC)
# 2. API с ключом (ключ меняется каждый час, время — UTC)
KEY="${WPS_API_STATIC_KEY}$(date -u '+%Y.%m.%d %H')"
curl -s -H "X-Auth-Key: $KEY" http://<crm>/api/v9/modules | head -c 200            # 200 and {"data":[{"type":"modules",... / 200 и {"data":[{"type":"modules",...
curl -s -o /dev/null -w '%{http_code}\n' -H 'X-Auth-Key: wrong' http://<crm>/api/v9/modules   # 401
curl -s -g -H "X-Auth-Key: $KEY" "http://<crm>/api/v9/Leads?filter[status]=New&size=1"   # field search (-g: brackets are not a template) / поиск по полю (-g: скобки не шаблон)

# 3. event: save any lead in CRM and look at the tail of the stream in WPS Redis
# 3. событие: сохраните любой лид в CRM и посмотрите хвост стрима в Redis WPS
redis-cli -h <redis-wps> XREVRANGE crm_events + - COUNT 1
```

## 8. Connecting WPS to This CRM / Подключение WPS к этой CRM

**EN:** In the WPS (backend) `.env`:

**RU:** В `.env` WPS (backend):

```bash
CRM_API_BASE_URL=http://<crm>/api/v9      # the path must end with /api/v9 / путь должен заканчиваться на /api/v9
CRM_API_STATIC_KEY=<the same WPS_API_STATIC_KEY / тот же WPS_API_STATIC_KEY>
MOCK_SERVICE_URL=                         # empty: the module dictionary is taken from the CRM / пусто: справочник модулей берётся из CRM
```

**EN:** Restart the WPS backend and refresh the module dictionary (`POST /api/dictionaries/modules/refresh`).

**RU:** Перезапустите backend WPS и обновите справочник модулей (`POST /api/dictionaries/modules/refresh`).

## 9. Upgrading / Обновление

**EN:** Running `bin/install.sh` again with a new package version is an upgrade: the script removes the files of the
installed version (as Module Loader does on upgrade), installs the new one and updates the Module Loader record. The
`wps_processed_c` field with its data and the parameters are preserved. The same run with the same version is a
reinstallation (idempotent).

**RU:** Повторный запуск `bin/install.sh` с новой версией пакета — это обновление: скрипт удаляет файлы установленной версии
(как Module Loader при обновлении), ставит новую и обновляет запись в Module Loader. Поле `wps_processed_c` с данными и
параметры сохраняются. Тот же запуск с той же версией — переустановка (идемпотентно).

**EN:** Via Module Loader: first **Uninstall** the old version (select "Do not remove tables" to keep the field),
then install the new one as in [4.3](#43-manual-installation-via-admin--module-loader--вручную-через-admin--module-loader).

**RU:** Через Module Loader: сначала **Uninstall** старой версии (выберите «Do not remove tables», чтобы сохранить поле),
затем установка новой, как в [4.3](#43-manual-installation-via-admin--module-loader--вручную-через-admin--module-loader).

## 10. Uninstalling / Удаление

```bash
bin/uninstall.sh --crm-path /var/www/html [--apache-conf <vhost>] [--htaccess] [--keep-config] [--remove-fields]
bin/uninstall.sh --docker <container> [the same options / те же опции]
```

| What / Что | By default / По умолчанию | Option / Опция |
|---|---|---|
| EN: `custom/include/WPS` files, logic hook, entry point, labels<br>RU: файлы `custom/include/WPS`, logic hook, entry point, подписи | EN: removed<br>RU: удаляются | — |
| EN: record and zip in Module Loader<br>RU: запись и zip в Module Loader | EN: removed<br>RU: удаляются | — |
| EN: `$sugar_config['wps']` block in `config_override.php`<br>RU: блок `$sugar_config['wps']` в `config_override.php` | EN: removed<br>RU: удаляется | EN: `--keep-config` — keep / оставить<br>RU: `--keep-config` — оставить |
| EN: `Leads.wps_processed_c` field **with data**<br>RU: поле `Leads.wps_processed_c` **с данными** | EN: kept<br>RU: остаётся | EN: `--remove-fields` — remove / удалить<br>RU: `--remove-fields` — удалить |
| EN: rewrite in vhost / `.htaccess`<br>RU: rewrite в vhost / `.htaccess` | EN: kept<br>RU: остаётся | EN: `--apache-conf <file>` / `--htaccess` — remove the block between the markers<br>RU: `--apache-conf <файл>` / `--htaccess` — удалить блок между маркерами |

**EN:** Via Module Loader: **Uninstall** in the package row; "Remove tables" also removes the field with its data.

**RU:** Через Module Loader: **Uninstall** в строке пакета; «Remove tables» удаляет и поле с данными.

## 11. Troubleshooting / Диагностика

| Symptom / Симптом | Cause / Причина | Solution / Решение |
|---|---|---|
| EN: `install.sh`: "is not the legacy root of SuiteCRM"<br>RU: `install.sh`: «is not the legacy root of SuiteCRM» | EN: wrong `--crm-path`<br>RU: неверный `--crm-path` | EN: specify the CRM root (it contains `include/entryPoint.php`)<br>RU: укажите корень CRM (там `include/entryPoint.php`) |
| EN: "looks like SuiteCRM 8: use the SuiteCRM 8 package"<br>RU: «looks like SuiteCRM 8: use the SuiteCRM 8 package» | EN: this is SuiteCRM 8<br>RU: это SuiteCRM 8 | EN: use the `SuiteCRM8` package<br>RU: используйте пакет `SuiteCRM8` |
| EN: "does not support SuiteCRM x.y"<br>RU: «does not support SuiteCRM x.y» | EN: version outside `^7\.14\.`<br>RU: версия вне `^7\.14\.` | EN: upgrade the CRM or use `--force` at your own risk<br>RU: обновите CRM или `--force` на свой риск |
| EN: self-check: `404` for both keys<br>RU: самопроверка: `404` на оба ключа | EN: rewrite missing or the entry point is not registered<br>RU: нет rewrite или entry point не зарегистрирован | EN: [Section 5](#5-web-server-configuration--настройка-веб-сервера); `wps_setup.php status`<br>RU: [раздел 5](#5-web-server-configuration--настройка-веб-сервера); `wps_setup.php status` |
| EN: self-check: `401` for both keys<br>RU: самопроверка: `401` на оба ключа | EN: without the rewrite the request went to the deprecated SuiteCRM API (`^api/`), or the key is empty/different<br>RU: без rewrite запрос ушёл в устаревший API SuiteCRM (`^api/`), либо пустой/другой ключ | EN: check the rewrite (the block must come before the SuiteCRM rules); `WPS_API_STATIC_KEY`; the server clock (UTC)<br>RU: проверьте rewrite (блок должен стоять до правил SuiteCRM); `WPS_API_STATIC_KEY`; часы сервера (UTC) |
| EN: `POST/PUT` → "Bad data passed in"<br>RU: `POST/PUT` → «Bad data passed in» | EN: body without `Content-Type: application/json`<br>RU: тело без `Content-Type: application/json` | EN: WPS sends the header; add it in your own requests<br>RU: WPS отправляет заголовок; в своих запросах добавьте его |
| EN: saving a record is slow, `[WPS] publish failed` in `suitecrm.log`<br>RU: сохранение записи медленное, в `suitecrm.log` `[WPS] publish failed` | EN: WPS Redis is unreachable<br>RU: Redis WPS недоступен | EN: network / `WPS_REDIS_HOST`; the record is saved but the event is lost (no outbox); timeout — `WPS_REDIS_TIMEOUT`<br>RU: сеть / `WPS_REDIS_HOST`; запись сохраняется, событие теряется (outbox нет); таймаут — `WPS_REDIS_TIMEOUT` |
| EN: Module Loader: "Possible Cross Site Request Forgery"<br>RU: Module Loader: «Possible Cross Site Request Forgery» | EN: a direct URL was opened without a Referer<br>RU: открыт прямой URL без Referer | EN: go through the Admin → Module Loader menu<br>RU: заходите через меню Admin → Module Loader |
| EN: Module Loader: "Duplicate column name 'wps_processed_c'"<br>RU: Module Loader: «Duplicate column name 'wps_processed_c'» | EN: installing version 1.0.0 (the field was in the manifest's `custom_fields`) over an already saved field<br>RU: установка версии 1.0.0 (поле было в `custom_fields` manifest) поверх сохранённого поля | EN: use 1.1.0+: the field is created by the script only if it does not exist<br>RU: используйте 1.1.0+: поле создаётся скриптом только если его нет |
| EN: after removal, `custom/include/WPS` files remain<br>RU: после удаления остались файлы `custom/include/WPS` | EN: installations by the old installer (1.0.0) were reinstalled without removal, and ModuleInstaller restored the "restore point"<br>RU: установки старым установщиком (1.0.0) переустанавливались без удаления, и ModuleInstaller восстановил «restore point» | EN: `bin/install.sh` will remove the leftovers ("Removed leftover…"), then run `bin/uninstall.sh`<br>RU: `bin/install.sh` удалит остатки («Removed leftover…»), затем `bin/uninstall.sh` |

**EN:** Logs: `suitecrm.log` in the CRM root (lines `[WPS]`), the Apache error log.

**RU:** Логи: `suitecrm.log` в корне CRM (строки `[WPS]`), лог ошибок Apache.

## 12. Event and API Contract / Контракт события и API

**EN:** **Event** (the `data` field of the stream message) — a JSON object: `id`, `module`, `object_type` and all stored
fields of the record, including `_cstm`, re-read from the database (dates — DB format, UTC). Relations, non-db fields
and secrets (`user_hash` etc.) are not included. Service fields: `wps_event` (`after_save`), `wps_is_update` (`1`/`0`),
`wps_user_id`, `wps_published_at`.

**RU:** **Событие** (поле `data` сообщения стрима) — JSON-объект: `id`, `module`, `object_type` и все хранимые поля записи,
включая `_cstm`, перечитанные из БД (даты — формат БД, UTC). Связи, non-db поля и секреты (`user_hash` и т. п.)
не попадают. Служебные поля: `wps_event` (`after_save`), `wps_is_update` (`1`/`0`), `wps_user_id`, `wps_published_at`.

**EN:** **API** (`X-Auth-Key`, JSON:API):

**RU:** **API** (`X-Auth-Key`, JSON:API):

| Request / Запрос | Response / Ответ |
|---|---|
| `GET /api/v9/modules` | `{data: [{type: "modules", id: "Leads", attributes: {name, label}}]}` |
| `GET /api/v9/{module}/{id}` | EN: `{data: {type, id, attributes}}`, `404` if not found<br>RU: `{data: {type, id, attributes}}`, `404` если нет |
| `GET /api/v9/{module}?filter[f]=v&size=1` | EN: `{data: [...], meta}`; filters are combined with AND, `size` ≤ 100, order `date_entered ASC`; unknown field → `422`<br>RU: `{data: [...], meta}`; фильтры по И, `size` ≤ 100, порядок `date_entered ASC`; неизвестное поле → `422` |
| `POST /api/v9/{module}` | EN: body `{data: {type, attributes}}` → `201 {data}`<br>RU: тело `{data: {type, attributes}}` → `201 {data}` |
| `PUT /api/v9/{module}/{id}` | EN: → `200 {data}`, `404` if not found<br>RU: → `200 {data}`, `404` если нет |

**EN:** Errors: `{errors: [{status, title, detail?}]}` — `400` (malformed JSON), `401` (key), `404`, `405` (method or a
record in `Users`/`Employees`), `422`, `500`. Unknown and system attributes are not written; their list is in
`meta.ignored_attributes`. Dates are accepted as `Y-m-d H:i:s` (UTC) or ISO 8601.

**RU:** Ошибки: `{errors: [{status, title, detail?}]}` — `400` (битый JSON), `401` (ключ), `404`, `405` (метод или запись в
`Users`/`Employees`), `422`, `500`. Неизвестные и системные атрибуты не записываются, их список — в
`meta.ignored_attributes`. Даты принимаются как `Y-m-d H:i:s` (UTC) или ISO 8601.
