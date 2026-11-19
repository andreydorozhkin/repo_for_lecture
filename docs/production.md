# Продакшен: Compose за SafeLine WAF

Локальная разработка идёт без WAF (`npm run up`). На сервер ставят тот же стек с оверлеем `infra/docker-compose.prod.yml` и отдельным проектом SafeLine.

## Что открыто наружу

```text
интернет ── 80/443 ──> SafeLine tengine (network_mode: host)
                          ├── app.example.com   ──> 127.0.0.1:3000  Next.js
                          ├── api.example.com   ──> 127.0.0.1:8000  Kong (Auth, REST, Realtime, Storage, Functions)
                          └── sonar.example.com ──> 127.0.0.1:9000  SonarQube
```

| Порт на хосте | Кто слушает | Доступ |
| --- | --- | --- |
| 80, 443 | SafeLine | Интернет |
| 22 | sshd | Интернет, только по ключу |
| 127.0.0.1:3000 | Next.js | Только SafeLine |
| 127.0.0.1:8000 | Kong | Только SafeLine |
| 127.0.0.1:9000 | SonarQube | Только SafeLine |
| 127.0.0.1:9443 | Консоль SafeLine | SSH-туннель |
| 127.0.0.1:54323 | Studio, только с `--profile admin` | SSH-туннель |

Postgres 5432 в проде на хост не публикуется. Он доступен только контейнерам внутри сети Compose. База SonarQube тоже остаётся внутри сети: наружу смотрит только веб-интерфейс.

Docker публикует порты в обход ufw и firewalld. Поэтому защищает не фаервол, а привязка к `127.0.0.1` в compose-файлах. CI-джоба `compose` падает, если какой-то сервис публикует порт на другом адресе.

## Сервер

- Linux, Docker с Compose v2. Режим `network_mode: host` у tengine работает только на Linux.
- Памяти с запасом: Supabase, SafeLine и SonarQube. У SonarQube три кучи по 512 МБ, вместе со стеком это заметно больше, чем локальный запуск.
- Три DNS-имени, указывающие на сервер: `app.example.com`, `api.example.com`, `sonar.example.com`.
- Фаервол как второй рубеж:

```bash
ufw default deny incoming
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

## Запуск

На сервере, в клоне репозитория:

```bash
npm run keys:prod -- --app-url https://app.example.com --api-url https://api.example.com --sonar-url https://sonar.example.com
```

Скрипт создаёт два файла с правами 0600, оба в `.gitignore`:

- `infra/.env.prod` — переменные Supabase, приложения и `SONAR_PUBLIC_URL`. Публичные URL уже прописаны, автоподтверждение почты выключено.
- `infra/safeline/.env` — переменные SafeLine. Файл отдельный, потому что у SafeLine свои `POSTGRES_PASSWORD` и `REGION`, и в общем файле они затёрли бы переменные Supabase.

Ключи генерируйте на самом сервере. `ARCH_SUFFIX` выбирается по архитектуре машины, на которой запущен скрипт: на arm64 это `-arm`, на amd64 пусто.

Заполните `SMTP_*` в `infra/.env.prod`. В проде регистрация подтверждается письмом, и без SMTP стек не стартует: compose остановится с ошибкой `required variable SMTP_HOST is missing`.

```bash
npm run up:prod
npm run waf:up
npm run waf:reset-admin   # печатает логин и пароль консоли SafeLine
```

SafeLine хранит данные в `SAFELINE_DIR` (по умолчанию `/data/safeline`). Его запускают отдельным compose-проектом `safeline`, поэтому `npm run up:prod` при деплое приложения WAF не перезапускает.

## Настройка SafeLine

Консоль открывается только через туннель:

```bash
ssh -L 9443:127.0.0.1:9443 user@server
# в браузере https://localhost:9443, сертификат самоподписанный
```

1. Добавьте три приложения (сайта), все на порту 443 с HTTPS:
   - `api.example.com`, upstream `http://127.0.0.1:8000`;
   - `app.example.com`, upstream `http://127.0.0.1:3000`;
   - `sonar.example.com`, upstream `http://127.0.0.1:9000`.
2. Сертификаты: загрузите свои или выпустите бесплатный в консоли. Включите редирект с 80 на 443.
3. Режим защиты переключите с наблюдения на блокировку, когда убедитесь, что обычные сценарии (вход, создание заметки, открытие SonarQube) не режутся.
4. Включите ограничение частоты запросов. В первую очередь оно нужно для `api.example.com/auth/v1/*`, где подбирают пароли.
5. Не включайте JS-проверку ботов и аутентификацию SafeLine на `api.example.com` и на `sonar.example.com`. Клиент Supabase и сканер SonarQube ходят через обычный HTTP и такую проверку не пройдут. Картинки в комментарии pull request GitHub тоже качает с `sonar.example.com` без браузера. На `app.example.com` проверку можно включать.
6. Проверьте Realtime: он работает по WebSocket через `api.example.com/realtime/v1/`.

## SonarQube и GitHub

SonarQube здесь один на всех. Локально `npm run up` его не поднимает: личный инстанс не виден остальным и не принимает отчёты из GitHub Actions. CI ходит на `SONAR_HOST_URL`, это публичный адрес за SafeLine.

После первого старта сервер несколько минут поднимает индексы. Первый вход: `admin` / `admin`, пароль сразу смените.

`SONAR_PUBLIC_URL` compose кладёт в `sonar.core.serverBaseURL`. Ссылки в комментариях GitHub берутся отсюда. Поле в интерфейсе (Administration > Configuration > General settings > General > Server base URL) контейнер перебивает своим значением. Менять адрес нужно в `infra/.env.prod` и пересоздавать контейнер.

Дальше:

1. В SonarQube: My Account > Security, токен. В секреты репозитория GitHub положите `SONAR_TOKEN` и `SONAR_HOST_URL` (`https://sonar.example.com`, тот же адрес, что `SONAR_PUBLIC_URL`). Ключ проекта берут из `sonar-project.properties`. В шаблоне это `template-for-mvp`, в новом репозитории его меняют на имя репозитория, иначе анализы разных сервисов смешаются. Первый анализ создаёт проект, если у токена есть право создавать проекты.
2. Administration > Configuration > General settings > DevOps platform integrations > GitHub > Create GitHub App. Укажите организацию и поставьте приложение на репозиторий. Базовый URL к этому моменту уже публичный, иначе GitHub вернёт браузер на localhost. Кнопка есть благодаря webapp из плагина ветвей: в чистом Community её нет.
3. Сначала должен пройти анализ ветки по умолчанию (push в `main`). Плагин ветвей не оформляет pull request, пока целевая ветка ни разу не проанализирована. На pull request CI передаёт `sonar.scm.revision`, без него комментарий в GitHub не появляется.
4. Если в комментарии pull request нет картинок, GitHub не смог скачать их с сервера. Тогда в General > Pull Request > Images base URL поставьте `https://raw.githubusercontent.com/mc1arke/sonarqube-community-branch-plugin/master/src/main/resources/static`.

## Обслуживание

Studio в проде по умолчанию не запущен:

```bash
docker compose --env-file infra/.env.prod -f infra/docker-compose.yml -f infra/docker-compose.prod.yml --profile admin up -d studio
ssh -L 54323:127.0.0.1:54323 user@server   # http://localhost:54323
```

Studio не спрашивает пароль, если открыть его напрямую через туннель. Пока он запущен, он доступен ещё и через Kong на `api.example.com/` под basic auth из `DASHBOARD_USERNAME` и `DASHBOARD_PASSWORD`. После работы остановите его: `docker stop mvp-studio-1`.

Обновление приложения: `git pull && npm run up:prod`. Новые миграции применит сервис `migrate`.

## Что проверено, а что нет

Проверено:

- `docker compose config` для дев- и прод-вариантов;
- что в проде нет публикации 5432 и Studio, а SonarQube слушает только `127.0.0.1:9000`;
- что без SMTP прод не стартует;
- что пароли Postgres у SafeLine и Supabase разные;
- что сервисы SafeLine поднимаются и консоль отвечает на `127.0.0.1:9443`.

Не проверено: tengine в host-сети на Linux-сервере, настройка сайтов и TLS в консоли SafeLine, работа блокировки и rate limit на живом трафике. Шаги в разделе «Настройка SafeLine» написаны по документации SafeLine 9.4. Названия пунктов в интерфейсе могут отличаться.
