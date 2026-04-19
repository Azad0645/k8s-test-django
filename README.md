# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Структура манифестов

k8s.yaml — Deployment и Service Django

ingress.yaml — Ingress для star-burger.test

migrate.yaml — pod для python manage.py migrate

clearsessions-cronjob.yaml — регулярная очистка сессий

## Star Burger в Minikube

1. Включение Ingress: `minikube addons enable ingress`
2. Соберите образ Django и загрузите его в Minikube: `docker compose build web`, `minikube image load django_app:latest`
3. Подготовьте секреты приложения и примените манифесты: `kubectl apply -f k8s/`
4. Дождитесь готовности Deployment: `kubectl rollout status deployment/django`
5. Пропишите домен star-burger.test в hosts, чтобы он указывал на IP Minikube: `minikube ip`
6. Откройте сайт по адресу: http://star-burger.test

## Регулярная очистка сессий

Применить CronJob: `kubectl apply -f k8s/clearsessions-cronjob.yaml`

Принудительно запустить job из cronjob: `kubectl create job django-clearsessions-once --from=cronjob/django-clearsessions` 

Проверка cronjob: `kubectl get cronjob`

## База PostgreSQL (Helm)

1. Добавьте репозиторий Helm и обновите его: `helm repo add bitnami https://charts.bitnami.com/bitnami`, `helm repo update`
2. Поднимите PostgreSQL: 
```bash
helm install postgres bitnami/postgresql \
  --set fullnameOverride=postgres \
  --set auth.username=test_k8s \
  --set auth.password=OwOtBep9Frut \
  --set auth.database=test_k8s
```
3. Обновите секрет приложения под новую БД: `kubectl apply -f k8s/secret.yaml`
4. Примените миграции: `kubectl delete pod django-migrate --ignore-not-found`, `kubectl apply -f k8s/migrate.yaml`, `kubectl logs -f django-migrate`
5. Перезапустите приложение: `kubectl rollout restart deployment/django`, `kubectl rollout status deployment/django`