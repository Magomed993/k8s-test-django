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
## Создание кластера Minikube
Скачайте [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download) на ПК. \
Запустите кластер Minikube:
```shell
minikube start --driver=kvm2
```
У Minikube работает свой формат Docker. Для сопряжения с Docker установленным локально пропишите команду в Linux:
```shell
eval $(minikube docker-env)
```
## Создание БД внутри кластера
Для создания БД внутри кластера для начала устанавливаем [helm](https://helm.sh/) в ОС. \
Далее производим установку пакета postgresql:
```bash
helm install name \
--set auth.username=<your_user_name> \
--set auth.password=<password_db> \
--set auth.database=<name_db> \
oci://<REGISTRY_NAME>/<REPOSITORY_NAME>/postgresql
```
## Создание Secret через kubectl
Для создания секретных файлов в k8s используется несколько методов:
1. Создание Secret через yaml файл. Пример `secret.yaml`:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: dotfile-secret
    type: Opaque
    stringData:
      SECRET_KEY: "secret_data"
    ```
    Далее задаём команду в месте где находится наш файл:
    `kubectl apply -f secret.yaml`
2. Создание Secret через командную строку. Создаём файл `.env`, прописываем команду в месте где находится файл `.env`:
    ```shell
   kubectl create secret generic django-secret --from-env-file=.env
    ```
## Создание Ingress
Создаём файл `ingress.yaml` для работы с доменами:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: your_name

spec:
  rules:
    - host: your_host
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: your_service_name
                port:
                  number: your_number_port
```
Далее активируем данный файл:
```shell
kubectl apply -f ingress.yaml
```
Если работаем через minikube локально, то необходимо активировать NGINX Ingress Controller:
```shell
minikube addons enable ingress
```
Локально, у себя на компьютере находим файл `/etc/hosts` и прописываем:
```text
<minikube_ip> <domen>
```
## Создание CronJob
Создаём файл `clearsession_cronjob.yaml` для ежемесячной очистки сессии Django:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: django-clearsessions-once # Наименование CronJob
spec:
  schedule: "0 0 1 * *" # Указывается первый день начала каждого месяца в 00ч 00мин
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: name_container
            image: your_image
            command: ["python", "manage.py", "clearsessions"]
            envFrom:
              - secretRef:
                  name: django-secret
          restartPolicy: OnFailure
```
Применяем:
```shell
kubectl apply -f clearsession_cronjob.yaml
```
Если ждать первое число долго, но проверить задачу хочется, то создаём `job` из имеющегося `cronjob`:
```shell
kubectl create job clearsession-job --from=cronjob/django-clearsessions-once
```