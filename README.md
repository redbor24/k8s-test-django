# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Запуск сайта в Kubernetes

### Переходим в каталог Django-проекта
```shell
cd k8s-test-django\backend_main_django
```

### Создаём образ Django-приложения
```shell
docker build -t django_app .
```

### Создаём на [DockerHub](https://hub.docker.com/) публичный репозиторий с именем `<username>/gjango_app`

### Заливаем на [DockerHub](https://hub.docker.com/) наш свежесозданный образ
```shell
docker tag django_app:latest <username>/django_app:latest
docker push <username>/django_app:latest
```
* _Вместо `<username>` подставьте имя своего пользователя_

### Создаём конфигурацию в ConfigMap для нашего проекта
```shell
kubectl create configmap django-config --from-env-file=.env
```

### Запускаем образ в Kubernetes
```shell
kubectl apply -f django-app.yaml
```

### Включаем доступ к сайту через *Ingress*
```shell
minikube addons enable ingress
kubectl apply -f django-app.yaml
```
Получите IP-адрес вашего minikube
```shell
minikube ip
```
Добавьте в файл `hosts` строку
```
<minikube ip-address> star-burger.info
```
В Windows это `C:\Windows\System32\drivers\etc\hosts`, в Linux - `/etc/hosts`.

Теперь сайт доступен по ссылке [star-burger.info](http://star-burger.info)

### Автоматическое удаление сессий Django
Информацию о посетителях сайта Django хранит в базе данных внутри специальной модели Session. Она является частью стандартного приложения Django django.contrib.sessions. Каждая запись в таблице Session – это сессия одного посетителя сайта.

Для каждого пользователя, зашедшего на ваш сайт, автоматически создаётся новая сессия и записывается в БД. Даже если кто-то случайно кликнул на ссылку на ваш сайт и сразу закрыл, то для него будет создана своя сессия. Из-за этого они довольно быстро копятся в базе данных.

Чтобы не хранить гигабайты устаревших сессий, включим их автоматическое удаление раз в сутки
```shell
kubectl apply -f clear-sessions-cronjob.yaml
```

### Применение миграций
```shell
kubectl apply -f django-migrate.yaml
```

