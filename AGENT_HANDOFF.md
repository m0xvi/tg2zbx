# AGENT_HANDOFF.md — паспорт проекта zabbix-tg-bot

> Файл-передача контекста для ИИ-агента / новой рабочей сессии.
> Прочитай его целиком перед любыми изменениями. Секретов здесь нет —
> они живут только в `.env` на серверах (никогда не коммитить!).

---

## 1. Что это за проект

Telegram-бот для управления Zabbix: просмотр/создание/редактирование/удаление
узлов, проблемы (группировка по узлам, фильтры, подтверждения), обслуживания,
графики метрик (matplotlib PNG), последние данные, push-уведомления
«упало/восстановилось», рестарт Zabbix-сервера и перезагрузка машины (/admin).

```
Telegram ←(long polling, через HTTP-прокси)→ aiogram 3 бот (Python)
                                            ←(JSON-RPC, Bearer)→ Zabbix API
```

**Стек:** Python 3.10+ (на серверах 3.12/3.13), aiogram 3 (`aiogram[proxy]`),
requests, matplotlib, python-dotenv, aiohttp + aiohttp-socks (для check_proxy и
SOCKS-прокси). Long polling — вебхук не нужен.

**Язык интерфейса бота — русский**, parse mode HTML (весь пользовательский
текст оборачивать в `esc()`).

## 2. Репозиторий

- GitHub: **https://github.com/m0xvi/tgbzbx** (публичный), ветка **master**.
- Владелец: m0xvi. Push по HTTPS требует PAT-токен GitHub (пароль аккаунта
  не работает с 2021) — `git config credential.helper store`.
- ⚠️ В репо файл примера конфига называется **`env.example`** (без точки);
  локально в рабочих копиях может лежать `.env.example`. `install.sh`
  понимает оба имени. `.env` в `.gitignore`.
- **Проверка свежести кода в репо** (исторически репо отставал от рабочих
  файлов): в свежей версии `zabbix_api.py` есть `auth_mode` и метод
  `_auth_mode_for_version` (адаптивная аутентификация), а в `bot.py` есть
  `ADMIN_USERS` и `install.sh` в корне. Если это не так — код устарел.
- Эталон последней версии на момент 2026-09-03: архив
  `zabbix-tg-bot-2026-09-03.zip` (у владельца). Состав: bot.py, zabbix_api.py,
  config.py, check_proxy.py, install.sh, requirements.txt, env.example,
  .gitignore, README.md, UPDATE.md, zabbix-tg-bot.service, AGENT_HANDOFF.md.

## 3. Развертывания (два, токен один — работает только ОДИН инстанс!)

| | Старый сервер | Новый сервер |
|---|---|---|
| Хост | `vanessa` | `zbx-tch-srv` |
| Папка | `/home/vanessa/zabbix-tg-bot` | `/opt/tgbzbx` (git clone) |
| Python | системный `python3`, без venv | `.venv` (создан `install.sh`) |
| systemd | юнит с `ExecStart=/usr/bin/python3 bot.py`, User=vanessa | юнит сгенерирован `install.sh`, `ExecStart=/opt/tgbzbx/.venv/bin/python bot.py` |
| Zabbix рядом | 6.0 LTS, фронт `http://10.20.9.50/zabbix` | **7.2+**, фронт `http://10.20.9.15/zabbix` |
| Бот | @Zabtest0000_bot (старый токен) | @zbxtgbtch_bot (новый токен) |

⚠️ **Один Telegram-токен = один работающий процесс.** Два одновременно —
`Conflict: terminated by other getUpdates request`. Перед запуском где-либо
убедиться, что на другой машине бот остановлен. Сейчас у машин токены разные,
но правило помнить.

Telegram-соединение идёт через HTTP-прокси с авторизацией (`TG_PROXY_URL`
в `.env`) — прямой доступ к api.telegram.org из сети заказчика закрыт.
MTProto-прокси не подходят, только http(s)/socks5.

## 4. Zabbix-пользователь и права

Бот ходит под пользователем **telegram-bot** (создаётся в каждом Zabbix свой):

- User type: **Admin** (обычный User не может создавать узлы);
- группа пользователей с правами **Read-write** на группы узлов и **Read** на
  группы шаблонов (иначе `host.get` вернёт 0, а `host.create` упадёт);
- в роли пользователя включён **API access** (при 412 — он выключен);
- авторизация: `user.login` (username/password) → токен сессии → заголовок
  `Authorization: Bearer` (см. п.10).

Для `/admin` на машине бота настроен sudoers
`/etc/sudoers.d/zabbix-tg-bot` — ровно две команды без пароля:
`systemctl restart zabbix-server` (оба пути systemctl) и `reboot`
(оба пути). Бот запускает их через `sudo -n`.

## 5. Файлы проекта

| Файл | Что внутри |
|---|---|
| `bot.py` | Весь бот (~1500 строк): роутер aiogram, все команды, FSM-диалоги, inline-клавиатуры, рендеры (узлы/проблемы/метрики), график matplotlib, /admin (subprocess+sudo), фоновый нотификатор |
| `zabbix_api.py` | Тонкий клиент JSON-RPC: Bearer-авторизация, авто-перелогин при 401/истечении сессии, типизированные методы (host.*, hostgroup.get, template.get, problem.get+trigger.get, event.acknowledge, maintenance.*, item.get, history.get, apiinfo.version) |
| `config.py` | Чтение `.env` (dotenv). Нормализация прокси (`_norm_proxy`: socks5h→socks5, голый host:port→http://) |
| `check_proxy.py` | Предзапусковая проверка: .env → прокси/getMe → Zabbix login + host.get + версия API. exit 0 = можно запускать |
| `install.sh` | Установка на новую машину: venv, зависимости, env.example→.env, check_proxy, генерация systemd-юнита. `--no-systemd` — без сервиса |
| `requirements.txt` | `aiogram[proxy]>=3.7,<4`, requests, python-dotenv, matplotlib |
| `env.example` | Шаблон .env со всеми переменными и комментариями |
| `zabbix-tg-bot.service` | Шаблон юнита (install.sh генерирует свой с реальными путями) |
| `README.md` | Полная документация: установка, настройка Zabbix/Telegram, прокси, /admin+sudoers, таблицы проблем |
| `UPDATE.md` | Процедура обновления (варианты git / zip), таблица новых переменных, чек-лист, откат |
| `.gitignore` | `.env`, `*.pyc`, `__pycache__/`, `.venv/`, `venv/`, `*.png` |

## 6. Переменные окружения (.env)

| Переменная | Назначение |
|---|---|
| `TG_TOKEN` | токен бота от @BotFather |
| `ALLOWED_USERS` | белый список Telegram user_id через запятую (ACL-мидлварь пускает только их) |
| `TG_PROXY_URL` | прокси для api.telegram.org: `http://user:pass@host:port` или `socks5://…` |
| `ZABBIX_URL` | фронтенд Zabbix (без /api_jsonrpc.php) |
| `ZABBIX_WEB_URL` | адрес для кнопок «🌐 открыть в Zabbix» (пусто = ZABBIX_URL) |
| `ZABBIX_USER` / `ZABBIX_PASSWORD` | API-пользователь |
| `ZABBIX_VERIFY_SSL` | false для самоподписанного сертификата |
| `HOSTS_PER_PAGE` | узлов на страницу (8) |
| `NOTIFY_ENABLED` / `NOTIFY_MIN_SEVERITY` / `NOTIFY_POLL_SECONDS` | уведомления: вкл, порог 0–5 (дефолт 3), период опроса ≥20 с |
| `ADMIN_USERS` | кому доступен /admin (пусто — раздел выключен) |
| `ZABBIX_SERVICE_NAME` | systemd-сервис Zabbix (zabbix-server) |

## 7. Архитектура бота (bot.py)

- Один `Router`, `Dispatcher`; ACL-мидлварь на message и callback_query
  (проверка `config.allowed_users`).
- Долгие вызовы Zabbix — через `asyncio.to_thread` (requests синхронный).
- `edit_or_answer()` — edit_text с fallback на answer (сообщения старше 48 ч).
- FSM-состояния: `AddHost` (name→ip→group_search→template_search→confirm),
  `EditHost` (name/ip/port/desc/tpl_add), `HostSearch.query`, `AdminReboot.ask`.
- Фоновая задача `problem_notifier` (стартует в `dp.startup`): опрос
  `problem.get` каждые NOTIFY_POLL_SECONDS, диф по eventid → «🆕 Упало» /
  «✅ Восстановлено», подавленные (maintenance) не считаются новыми;
  первая итерация — базлайн без отправки; `/notify on|off|N` меняет порог
  на лету и сбрасывает базлайн.
- `/admin`: `_gate_admin()` по `ADMIN_USERS`; `run_root()` перебирает варианты
  команды с `sudo -n`; рестарт Zabbix → `wait_zabbix_up()` (поллинг
  apiinfo.version до 90 с); ребут — двойное подтверждение (кнопка + слово
  «перезагрузка»); cooldown от даблклика.

## 8. Схема callback_data (НЕ ЛОМАТЬ — кнопки в старых сообщениях)

| Префикс | Формат | Действие |
|---|---|---|
| `hp:` | `hp:<page>:<mode>[:<status>]` | страница списка узлов; mode: v/g/m/l; status: active/stopped/all |
| `hf:` | `hf:<status>:<mode>` | фильтр активности узлов |
| `hse:` | `hse:<mode>` | начать поиск узла |
| `hv:` | `hv:<hostid>:<page>` | карточка узла |
| `he:` | `he:<action>:<hostid>[:<extra>]` | редактирование: m/i/p/n/d/s/t/tr/ta/ts |
| `hl:` `hg:` `hm:` | `<prefix>:<hostid>` | метрики / выбор метрики для графика / обслуживание |
| `hgi:` `hgp:` | `hgi:<itemid>:<vtype>`, `hgp:<itemid>:<vtype>:<hours>` | период графика и сам график |
| `hpr:` | `hpr:<hostid>` | проблемы узла |
| `hmd:` | `hmd:<hostid>:<minutes>` | создать обслуживание |
| `mdel:` | `mdel:<maintenanceid>` | завершить обслуживание |
| `hdel:` / `hdelok:` | `hdel:<hostid>` | удаление узла (подтверждение) |
| `prb:` | `prb:<scope>:<minsev>:<ack>:<hours>` | список проблем; scope=g или h<hostid> |
| `prb1:` | `prb1:<scope>:<minsev>:<ack>:<hours>:<idx>` | просмотр «по одной» |
| `ack:` `ack1:` `ackall:` | см. bot.py | подтверждение проблем |
| `ah:` | `ah:<g|t|ok|cancel>[:<id>]` | диалог добавления узла |
| `adm:` | `adm:<menu|rz|rzok|rb|rbgo|close>` | управление сервером |
| `noop` | — | пустое действие |

Парсеры написаны толерантно к старому формату (без часов/статуса) — сохранять
это при рефакторинге.

## 9. Ключевые функции bot.py

`esc/cut/fmt_age/fmt_left/fmt_dt/fmt_val` — форматирование;
`render_host_list` (фильтры active/stopped/all, поиск, сортировка ярусами:
свежие проблемы → чистые → старые проблемы), `build_host_view`,
`render_problems` (группировка по узлам, окно по умолчанию 24 ч,
HOUR_OPTIONS 1/24/168/0), `render_problem_one` («по одной»),
`render_latest`, `render_edit_menu/render_tpl_menu`, `render_maintenances`,
`draw_graph`, `web_problems_url/web_latest_url` (ссылки на фронтенд Zabbix:
`zabbix.php?action=problem.view|latest.view&filter_set=1&filter_hostids[]=…`),
`diff_problems`, `format_new_problems/format_resolved_problems`.

## 10. Совместимость Zabbix 6.0 ↔ 7.x (важно!)

- Токен передаётся заголовком **`Authorization: Bearer`** — работает на 5.4+.
  Передача `auth` в теле удалена в 7.2 (`Invalid parameter "/": unexpected
  parameter "auth"`). Возвращать `auth` в тело НЕЛЬЗЯ.
- `user.login` шлёт `{"username": …}` — параметр `username` появился в 6.4;
  на 6.0-сервере заказчика работал (видимо, 6.4+). Если на очень старом
  Zabbix будет «unexpected parameter username» — заменить на `user`.
- `problem.get` в 6.0 не имеет selectHosts → имена узлов достаются вторым
  запросом `trigger.get` по objectid (см. `get_problems`).
- `maintenance.create` принимает `hosts:[{hostid}]` (не устаревший hostids).
- `event.acknowledge` с `action:1` (acknowledge) — ок в 6.0 и 7.x.

## 11. Тестирование (как проверяли и как проверять дальше)

Песочница не имеет доступа к Zabbix/Telegram-боту, поэтому:

1. `python3 -m py_compile *.py` + `import bot` (ловит синтаксис/импорт).
2. Мок Zabbix JSON-RPC: `python http.server` на `127.0.0.1:18099`,
   скрипт-тесты с assert'ами (логин, host.create, фильтры проблем, диф
   уведомлений, сортировку узлов, admin-хелперы). Мок «7.2» обязан
   отвергать `auth` в теле и требовать Bearer, отдавать 401 на битый токен.
3. `check_proxy.py` гоняли и на живой API (реальный 401 от Telegram и т.п.).

Перед коммитом: py_compile + `import bot` + мок-тесты новых/затронутых
функций. Реальная проверка — на сервере заказчика через check_proxy.py.

## 12. Процедуры

- **Установка на новую машину:** `git clone` → `./install.sh --no-systemd` →
  заполнить `.env` → `sudo ./install.sh` (check_proxy + systemd).
  Свежая Ubuntu может держать apt-lock из-за unattended-upgrades — просто
  подождать или `apt install -o DPkg::Lock::Timeout=600`.
- **Обновление:** по UPDATE.md: stop → бэкап .env → git pull → diff .env
  env.example → pip install → check_proxy → start → journalctl -f.
- **Откат:** `git checkout <hash>` / распаковать прошлый zip. Данные Zabbix
  бот не трогает.

## 13. Исторические грабли (уже решённые, не наступать снова)

- `python check_proxy.py` системным python без venv → `ModuleNotFoundError:
  aiohttp` (теперь скрипт сам печатает подсказку про `.venv/bin/python`).
- `kill -9 unattended-upgr` не помогает (systemd перезапускает) → ждать lock.
- git push паролем аккаунта не работает — только PAT (scope repo) или SSH.
  Пользователь однажды пушлил не туда: репо `MKgit` вместо `tgbzbx`,
  ветка `main` вместо `master` — проверять `git remote -v` и `git branch`.
- На старом сервере юнит изначально был под venv, но бот запускался нативно —
  юнит переписан под системный python3 (файл в репо — под нативный запуск;
  install.sh генерирует свой вариант).
- 350-дневные «Unavailable by ICMP ping» от выключенных узлов — не баг бота:
  это незакрытые проблемы Zabbix. Отсюда окно 24 ч по умолчанию и скрытие
  остановленных узлов. Лечится в самом Zabbix (остановить/удалить узлы).

## 14. Стиль и конвенции

- Комментарии и docstring — русские, краткие.
- UI: HTML parse mode, всё через `esc()`; сообщения ≤4096 (`cut()`).
- Эмодзи-статусы: 🟢 доступен, 🟡 агент недоступен, ❔ нет данных, 🔧
  обслуживание, ⏸ остановлен, 💥🔴🟠⚠️ℹ️ важность 5..1, 🔥 свежая проблема,
  ✔ подтверждено, 🔇 suppressed, 🆕 упало, ✅ восстановлено.
- Колбэки начинаются с `await cb.answer()` до тяжёлой работы.
- Новые методы Zabbix добавлять в `zabbix_api.py` тонкими обёртками.
- Секреты только в `.env`; в коде/репо/логах их не печатать
  (check_proxy маскирует токены и пароль прокси).

## 15. Бэклог идей (обсуждались, не сделаны)

- Подтверждение проблемы с произвольным текстом сообщения (event.acknowledge
  message через FSM).
- Управление группами узлов (создание/переименование hostgroup).
- Уведомления с настройкой на пользователя (а не общий NOTIFY_*).
- Команда «короткий отчёт за смену» (итоги дня: упало/поднялось).
- Поддержка нескольких Zabbix одним ботом (выбор сервера).

## 16. Приветственный промпт для новой сессии

Скопировать владельцу в начало новой сессии (файл должен лежать в репо):

```
Ты — сопровождающий разработчик Telegram-бота для Zabbix (проект zabbix-tg-bot).
Репозиторий: https://github.com/m0xvi/tgbzbx (ветка master, владелец m0xvi).

Сначала прочитай в корне репозитория файл AGENT_HANDOFF.md — это полный
паспорт проекта: архитектура, схема callback_data, переменные .env,
два сервера развертывания, история проблем. Пока не прочитал — код не менять.

Жёсткие правила:
1) Секреты (токены, пароли, прокси) живут только в .env на серверах —
   никогда не коммитить и не печатать.
2) Интерфейс бота — русский, HTML parse mode, пользовательский текст через esc().
3) Совместимость Zabbix 6.0 и 7.2+ обязательна: токен в заголовке
   Authorization: Bearer (не в теле запроса!).
4) Схему callback_data не ломать без нужды — в чатах живут старые сообщения.
5) Перед коммитом: python -m py_compile + import bot + тесты на мок-сервере
   Zabbix (подход описан в AGENT_HANDOFF.md, п.11).
6) Один Telegram-токен = один работающий инстанс бота (две машины!).
7) Изменения в репо — осмысленными коммитами; обновление серверов — по UPDATE.md.

Задача: <здесь описание задачи>
```
