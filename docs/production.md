# SS14.Admin для StarHorizon

## Назначение

Веб-панель администрирования основного сервера StarHorizon. Панель читает и изменяет ту же PostgreSQL-базу, что и игровой сервер: показывает игроков, подключения, персонажей и admin logs, управляет банами, role bans и whitelist.

Публичный адрес: `https://admin.сс14.рф` (`admin.xn--14-nmca.xn--p1ai` в Nginx).

## Совместимость

Ветка `codex/starhorizon` основана на SS14.Admin v1.8.0 и собирается с `Content.Server.Database` из закреплённого подмодуля `StarHorizon14/StarHorizon`.

Не заменять базовый код или образ на upstream v1.9.x без миграции и отдельной проверки: v1.9.x ожидает новую схему таблиц банов, которой пока нет в StarHorizon.

## Данные и секреты

- PostgreSQL: контейнер `ss14-database`, база `ss14` в сети `ss14_default`.
- Панель должна использовать отдельного пользователя `ss14_admin`.
- OAuth выполняется через официальный аккаунт Space Station 14.
- Секреты находятся только в `deploy/.env` на сервере. Файл не коммитится.
- Ключи ASP.NET Data Protection сохраняются в Docker volume `ss14_admin_keys`; без него OAuth-сессии перестанут расшифровываться после пересоздания контейнера.
- OAuth callback: `https://admin.xn--14-nmca.xn--p1ai/signin-oidc` (ASCII/punycode-форма адреса, необходимая для точного сопоставления `redirect_uri`).

## Подготовка

1. Создать DNS A/AAAA-запись `admin.сс14.рф` на production-хост.
2. Зарегистрировать OAuth-приложение с указанным callback и homepage `https://admin.xn--14-nmca.xn--p1ai`.
3. Сделать резервную копию базы.
4. Создать отдельную роль PostgreSQL и выдать только необходимые права на базу `ss14`.
5. Скопировать `deploy/.env.example` в `deploy/.env` и заполнить секреты.

## Развёртывание

```bash
git clone --recurse-submodules git@github.com:LerkOFF/SS14.Admin.git /home/star-horizon/containers/ss14-admin
cd /home/star-horizon/containers/ss14-admin
git switch codex/starhorizon
git submodule update --init --recursive
cp deploy/.env.example deploy/.env
# Заполнить deploy/.env, не печатая значения в журналы.
docker compose --env-file deploy/.env -f deploy/compose.yml build
docker compose --env-file deploy/.env -f deploy/compose.yml up -d
```

Nginx-конфигурация находится в `deploy/nginx.conf.example`. После её установки выпустить сертификат штатным Certbot и проверить конфигурацию до reload.

## Проверка

```bash
docker compose --env-file deploy/.env -f deploy/compose.yml ps
docker compose --env-file deploy/.env -f deploy/compose.yml logs --since 5m ss14-admin
curl -fsS http://127.0.0.1:27689/healthz
```

После OAuth-входа проверить под тестовым администратором:

1. Просмотр списка игроков, подключений и персонажей.
2. Поиск и фильтрацию admin logs.
3. Создание и снятие временного тестового бана.
4. Создание и снятие тестового role ban.
5. Добавление и удаление тестовой записи whitelist.
6. Отказ во входе для пользователя без записи в таблице `admin`.

## Обновление и откат

Обновлять только после сборки и проверки с актуальным commit подмодуля StarHorizon. Не использовать плавающий официальный тег `:1`.

Перед обновлением сделать резервную копию PostgreSQL и сохранить предыдущий digest образа. Для отката вернуть прежний commit ветки и поднять ранее собранный образ. Панель не должна выполнять миграции игровой БД самостоятельно.
