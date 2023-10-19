# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).
На примере установки используется адрес`http://star-burger.test`

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

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это, чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](test/docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgresSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Деплой проекта:
1. Установить:

[Kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)
[minikube](https://minikube.sigs.k8s.io/docs/)
[VirtualBox](https://www.virtualbox.org) - для Linux\Windows\MacOS на процессорах Интел.
[DockerDesktop](https://www.docker.com) - MacOS для Mac на M1.

Создать файл psql_config.yml:
```
auth:
  enablePostgresUser: true
  postgresPassword: ваш   postgres пароль
  username: юзер
  password: пароль для юзера
  database: название базы данных
```
3. Установите [Helm](https://helm.sh/)
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgres -f psql_config.yaml bitnami/postgresql
```
Helm установит и запустит `postgresql`, а также создаст пользователя по параметрам, указанным в `psql_config.yaml`.

4. Внести переменные окружения:
  - Создаем файл с названием `psql_config.properties`
```
SECRET_KEY=Ваш секретный ключ
DEBUG=False или True
DATABASE_URL=Ссылка на вашу базу данных PostgreSQL
ALLOWED_HOSTS=Список ваших хостов
```

Ссылка для подключения к базе: postgres://username:password@host:5432/db_name

  - Далее прописываем наш `ConfigMap` в системе с помощью команды:
```shell
kubectl create configmap psql_config --from-env-file=psql_config.properties
```
5. Настройте Ingress:
- Установите
```shell
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```
- В файле `ingress.yml` Меняем `host` на свой
- Пропишите на своей машине в `/etc/hosts` (для linux) домен `star-burger.test`, сопоставить с IP виртуальной машины 
узнать ip c помощью команды:

```shell
- minikube ip
```
6. Запустить манифесты:
```shell
kubectl apply -f deployment.yml
```
7. Проверить запущенные поды:
```shell
kubectl get pods
```
8. Зайти в shell 
```shell
kubectl exec --stdin --tty <pod_name> -- sh
```
9. Сделать миграцию
```python
./manage.py migrate
```
10. Создать суперпользователя
```python
./manage.py createsuperuser
```
11. Создайте cronjob, который очищает сессии django каждый месяц.
```shell
kubectl apply -f clear-sessions.yml
```
## Деплой проекта в кластере Kubernetes:
Пример сайта, работающего в кластере [Kubernetes](https://edu-clever-yalow.sirius-k8s.dvmn.org/admin/)

Для развертывания проекта в Kubernetes вам потребуется [cерверная инфраструктура:](https://sirius-env-registry.website.yandexcloud.net/edu-clever-yalow.html)

Для отладки вам пригодится  графический клиент: [Lens Desktop Personal](https://store.k8slens.dev/products/lens-desktop-personal) – бесплатная версия Lens.

Получите K8s Namespace из серверной инфрастуктуры "K8s Namespace"

Подключитесь к кластеру и выполните следующие команды: (замените K8s Namespace на свой)
```shell
kubectl -n <K8s Namespace> apply -f k8s-yandex/configmap.yaml
kubectl -n <K8s Namespace> apply -f k8s-yandex/django.yaml
kubectl -n <K8s Namespace> apply -f k8s-yandex/migrate.yaml
kubectl -n <K8s Namespace> apply -f k8s-yandex/clear-sessions.yaml
```

Получите имя пода. <pod-name> для подключения к базе данных.
```shell
kubectl get pods -n <K8s Namespace> 
```

Запустите shell в кластере
```shell
kubectl -n <K8s Namespace> exec -it <pod-name>   -- /bin/sh

```

Cоздайте пользователя:
```shell
python manage.py createsuperuser

```

Перейдите по ссылке на домен из [cерверной инфраструктуры ](https://sirius-env-registry.website.yandexcloud.net/edu-clever-yalow.html)

Проект выполнен в учебных целях на курсах [Devman](https://dvmn.org/)
























