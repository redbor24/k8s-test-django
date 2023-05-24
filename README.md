# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

## Запуск в Docker

Внутри контейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две 
функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. 
Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI.
[Подробнее про Nginx Unit](https://unit.nginx.org/).

Запустите базу данных и сайт:

```shell
docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell
docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт 
docker-образ, Сделано это, чтобы избежать конфликта имён. Внутри `docker-compose.yaml` настраиваются сразу несколько 
образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов 
к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри 
файла [`docker-compose.yml`](./docker-compose.yml).

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, 
важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. 
[Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт 
ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. 
[Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. 
[Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Запуск сайта в Kubernetes

### Переходим в каталог Django-проекта
```shell
cd k8s-test-django\backend_main_django
```

### Создаём образ Django-приложения
```shell
docker build -t django_app .
```

### Создаём на [DockerHub](https://hub.docker.com/) публичный репозиторий с именем `<username>/django_app`

### Заливаем на [DockerHub](https://hub.docker.com/) наш свежесозданный образ
```shell
docker tag django_app:latest <username>/django_app:latest
docker push <username>/django_app:latest
```
> вместо `<username>` подставьте своё имя пользователя DockerHub.

### Задаём значения ConfigMap и Secrets для нашего проекта

#### Development и Production конфигурации
Разработческая `dev` и продакшен `prod` среды имеют одни и те же, но с разными значениями параметры. 

Для создания конфигурационных файлов для [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) 
и [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) используйте следующие файлы-шаблоны:
- [django-secrets-EXAMPLE.yaml](backend_main_django/django-secrets-EXAMPLE.yaml)
- [django-app-cm-EXAMPLE.yaml](backend_main_django/django-app-cm-EXAMPLE.yaml)

Создайте на их основе отдельно файлы для `prod` и `dev` сред с различными значениями параметров.
Имена файлов могут быть любыми, но, крайне желательно, чтобы они отражали своё содержание. 

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, 
важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. 
[Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт 
ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. 
[Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts). Для prod-среде обычно имеет значение `*`. 

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. 
[Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

##### Для продакшен среды
```shell
kubectl apply -f django-app-cm-prod.yaml
kubectl apply -f django-secrets-prod.yaml
```

##### Для среды разработчика
```shell
kubectl apply -f django-app-cm-dev.yaml
kubectl apply -f django-secrets-dev.yaml
```

### Человеческий доступ к сайту 
В среде разработчика (на `minikube`) к сайту можно также обращаться по имени в URL. 
Для этого нужен _Kubernetes-service_ **Ingress**. В `minikube` он по умолчанию выключен. Проверяем:
```shell
minikube addons list
```
Ищем значение `STATUS` для строки `ingress`. Если оно `disabled`, то выполняем:
```shell
minikube addons enable ingress
```

_На "большом" Kubernetes **Ingress** обычно включён. За деталями обратитесь к документации хостинга._

Получите IP-адрес вашего minikube
```shell
minikube ip
```
Добавьте в файл `hosts` строку
```
<minikube ip-address> star-burger.info
```
_В Windows это `C:\Windows\System32\drivers\etc\hosts`, в Linux - `/etc/hosts`_

Теперь сайт доступен по ссылке [star-burger.info](http://star-burger.info)

### Запускаем образ в Kubernetes
```shell
kubectl apply -f django-app.yaml
```

### Создаём суперпользователя Django
#### Подключаемся к деплою *django-deploy*
```shell
kubectl exec -it deploy/django-deploy -- bash
```
Создаём суперпользователя
```shell
root@django-deploy-<pod_id>:/code# python manage.py createsuperuser
```
Завершаем сессию
```shell
root@django-deploy-<pod_id>:/code# exit
```

### Применяем миграции
```shell
kubectl apply -f django-migrate.yaml
```

### Автоматическое удаление сессий Django
Информацию о посетителях сайта Django хранит в базе данных внутри специальной модели Session. Она является частью 
стандартного приложения Django django.contrib.sessions. Каждая запись в таблице Session – это сессия одного посетителя 
сайта.

Для каждого пользователя, зашедшего на ваш сайт, автоматически создаётся новая сессия и записывается в БД. Даже если 
кто-то случайно кликнул на ссылку на ваш сайт и сразу закрыл, то для него будет создана своя сессия. Из-за этого они 
довольно быстро копятся в базе данных.

Чтобы не хранить гигабайты устаревших сессий, включим их автоматическое удаление раз в сутки
```shell
kubectl apply -f clear-sessions-cronjob.yaml
```

### Обновление кода сайта в _Kubernetes_

Переходим в каталог Django-проекта
```shell
cd k8s-test-django\backend_main_django
```

Создаём образ Django-приложения с указанием новой версии сборки, например `v2` 
```shell
docker build -t django_app:v2 .
```

Переименовываем образ для отправки в _DockerHub_ с указанием версии
```shell
docker tag django_app:v2 <username>/django_app:v2
```

Отправляем его в _DockerHub_
```shell
docker push <username>/django_app:v2
```

Обновляем образ в _Kubernetes_
```shell
kubectl set image deployment django-deploy django-app=<username>/django_app:v2
```
