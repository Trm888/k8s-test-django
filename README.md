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

### Запуск приложения в Kubernetes minikube

Запустите minikube в virtualbox:
```
minikube start --driver=virtualbox
```
Включите ingress:
```
minikube addons enable ingress
```

Заполните манифест файл ConfigMap.yaml:

`SECRET_KEY`

`DEBUG`

`ALLOWED_HOSTS`

`DATABASE_URL`

установить Helm

Добавите репозиторий Bitnami
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```
Установите PostgreSQL с определенным паролем и определенным именем пользователя:
```
helm install my-postgresql bitnami/postgresql --set postgresqlPassword=mysecretpassword,postgresqlUsername=myuser
```
Применяем манифесты:
```
kubectl apply -f configmap.yaml
kubectl apply -f django-app.yaml
kubectl apply -f ingress-hosts.yaml
kubectl apply -f clearsessions-cronjob.yaml
```

### Запуск приложения с использованием production-кластера

Работающая версия сайта: [edu-elated-rosalind.sirius-k8s.dvmn.org](https://edu-elated-rosalind.sirius-k8s.dvmn.org)


Переключаемся на кластер

```commandline
kubectl config use-context your_klaster
```
установить Helm

Добавите репозиторий Bitnami
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Установите PostgreSQL:

```
helm install my-postgresql bitnami/postgresql --namespace your_namespace
```

Получаем пароль от бд:
для Windows:

```
$jsonOutput = kubectl get secret --namespace your_namespace my-postgresql -o json | ConvertFrom-Json
$passwordBase64 = $jsonOutput.data.'postgres-password'
$POSTGRES_PASSWORD = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($passwordBase64))
Write-Output $POSTGRES_PASSWORD
```

Для Linux:
```
encoded_password=$(kubectl get secret my-postgresql -o json | jq -r '.data["postgres-password"]')
decoded_password=$(echo $encoded_password | base64 --decode)
echo $decoded_password
```
Полученный пароль используем в ConfigMap.yaml в качестве значения переменной DATABASE_URL

Применяем манифесты:
```
kubectl apply -f configmap.yaml
kubectl apply -f django-app.yaml
kubectl apply -f ingress-hosts.yaml
kubectl apply -f clearsessions-cronjob.yaml
```

Эти команды создадут Deployment, Service и Ingress для вашего приложения в вашем кластере Kubernetes.
