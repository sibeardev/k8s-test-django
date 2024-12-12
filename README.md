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


# Пошаговое руководство по запуску проекта в Minikube

Это руководство поможет вам развернуть проект на Minikube с использованием Kubernetes в локальной среде.

## Шаг 1: Установка Minikube и необходимых инструментов

### 1.1 Установка Minikube

#### Для macOS
1. Скачайте и установите Minikube с помощью Homebrew:
   ```bash
   brew install minikube
    ```
   
#### Для Windows и Linux

Следуйте инструкциям на [официальной странице Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Fx86-64%2Fstable%2Fbinary+download).

### 1.2 Установка kubectl

kubectl — это инструмент командной строки для взаимодействия с Kubernetes. Установите его, если у вас его нет.

#### Для macOS

```bash
brew install kubectl
```

#### Для Windows и Linux

Следуйте инструкциям на [официальной странице Kubernetes](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/).

### 1.3 Установка Helm

Helm — это пакетный менеджер для Kubernetes, который позволяет легко развертывать приложения и сервисы. Необходим для развертывания бд postgresql 

#### Для установки Helm:

```bash
brew install helm
```

Для других систем, следуйте инструкциям на [официальной странице Helm](https://helm.sh/docs/intro/install/).

## Шаг 2: Запуск Minikube

После установки Minikube, запустите кластер:

```bash
minikube start
```

Убедитесь, что Minikube работает правильно:

```bash
kubectl cluster-info
```

Вы должны увидеть URL-адреса для Kubernetes Master и других сервисов, доступных в вашем кластере.

### Шаг 3. Установите PostgreSQL

Установите Helm-чарт С параметрами по умолчанию:
```bash
helm install my-postgresql bitnami/postgresql
```
Найдите автоматически созданный пароль PostgreSQL:
```bash
kubectl get secret --namespace default my-postgresql -o jsonpath="{.data.postgres-password}" | base64 --decode
```
Получите имя сервиса PostgreSQL:
```bash
kubectl get svc
```

## Шаг 4: Применение манифестов из репозитория

Все необходимые манифесты для развертывания компонентов уже содержатся в папке kubernetes репозитория.

4.1 Применение манифестов
Перейдите в папку kubernetes, если вы еще не в ней:

```bash
cd kubernetes
```
Внесите изменения в django-secrets.yml.
Значения необходимо закодировать, пример:

```bash
echo -n postgres://username:pass@my-postgresql:5432/db_name | base64
```

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно.

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`.

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов.

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


Теперь примените манифесты Kubernetes:

```bash
kubectl apply -f django-secrets.yml
kubectl apply -f django-deployment.yml
kubectl apply -f ingress.yml
```

Применение миграций Django

```bash
kubectl apply -f django-migrate.yml
```

Запуск автоматического удаления устаревших сессий

```bash
kubectl apply -f django-clearsession.yml
```

### 4.2 Проверка развертывания

Чтобы проверить, что все поды и сервисы развернуты правильно, выполните:

```bash
kubectl get pods
kubectl get svc
```
Вы должны увидеть поды и сервисы для PostgreSQL, Django и CronJob для удаления устаревших сессий.
