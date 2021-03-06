# loktionovam_microservices
loktionovam microservices repository

## Homework-12: Технология контейнеризации. Введение в Docker

Основные задания:

- Установка docker

- Базовая работа с консольной утилитой docker - создание, запуск, останов, удаление контейнеров, создание образов из контейнеров, удаление образов

Задание со *: описание отличий между container и image

Про образы.

- Docker image состоит из набора read-only слоев, например

```json
            "Layers": [
                "sha256:711e4cb62f50eb37292a0df824072b6a818211e3f9b3aae75a2067e65fa32eca",
                "sha256:a0e188d0e2789aeb82bd848958f432fbc19704bfa3337e0cad00545b43550f51",
                "sha256:6eaddaf493f101a62bb0875185e706424a33636db5d8a55945d63f6f870d7dd2",
                "sha256:07b9c3c04cbdf53317cb326fe3db42073366add5a8919f590e1574252ac6f8c9",
                "sha256:709bdd00b1a496a5ee836b8bc3e0f371fa114e9ff4cc524c0862696bbfc69c9c",
                "sha256:11266888239dc2bbd77759fb2c8df2eb2284983f817c745605d42d46402258a9"
     ]
```

- Каждый слой - это diff от предыдущего слоя.

- Разные images используют общие слои (не нужно загружать каждый слой каждый раз если он уже есть).

- В образах есть описание контейнеров (см. ниже), которые запущены поверх него

```json

        "Container": "b8fc5d0067d0ebc04a61b579250dc1753597e75fcc7554b2bebb6aefeb7abcp "${REPO_PATH}"/",
        "ContainerConfig": {
            "Hostname": "b8fc5d0067d0",
```

Про контейнеры.

- Создание нового контейнера создает COW read/write слой поверх image, этот слой называется container layer. В нем отличие контейнера от image (когда удаляется контейнер, то удаляется только его r/w слой, нижележащий image остается)

- При записи в файл контейнер просматривает слои (сверху-вниз, от самого нового, до самого старого). Файл копируется в writable слой и вся запись происходит в этот скопированный файл при этом контейнер не видит нижележащие read-only копии этого файла.

- В контейнерах есть состояние связанное с его runtime (pid, аппаратные ресурсы - память, процессор, описание сетевых интерфейсов), например

```json
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
...
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
```

Контейнеры это про runtime+r/w слой, images - это про хранение/доставку приложений.

## Homework-13: Docker контейнеры. Docker под капотом

### 13.1 Что было сделано

Основные задания:

- Создание docker host в GCP через docker machine

- Создание образа docker контейнера otus-reddit:1.0

- Создание репозитория на docker hub и загрузка в него образа otus-reddit:1.0

Задания со *:

- В docker-monolith/infra/ansible добавлена конфигурация ansible для настройки docker host (роль docker_host) и настроено dynamic inventory через gce.py

- В docker-monolith/infra/packer добавлена конфигурация для создания образа с уже установленным docker

- В docker-monolith/infra/terraform добавлена конфигурация для подъема инстансов docker-host-xxx, количество которых, задается переменной count


### 13.2 Как запустить проект

### 13.2.1 Шпаргалка по командам docker, docker machine. Создание образа otus-reddit

```bash
docker-machine create --driver google  --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts  --google-machine-type n1-standard-1   --google-zone europe-west1-b  docker-host

docker-machine ls

docker-machine env docker-host
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://ip:port"
export DOCKER_CERT_PATH="/some/path/to/.docker/machine/machines/docker-host"
export DOCKER_MACHINE_NAME="docker-host"
# Run this command to configure your shell:
# eval $(docker-machine env docker-host)

eval $(docker-machine env docker-host)

# Запустить htop (PID из container namespaces)
docker run --rm -ti tehbilly/htop

# Запустить htop (PID из host namspaces)
docker run --rm --pid host -ti tehbilly/htop

docker run --help | grep "\-\-pid "
      --pid string                     PID namespace to use


# -t - tag (--tag=)
docker build -t reddit:latest .

# -d background mode (--detach=true)
# --network - use host network
docker run --name reddit -d --network=host reddit:latest

# Создать  tag алиас на reddit:latest
docker tag reddit:latest loktionovam/otus-reddit:1.0
# Запушить образ в docker registy
docker login
docker push loktionovam/otus-reddit:1.0

# Запустить существующий контейнер
docker stop reddit
docker start reddit

# Логи
docker logs reddit -f

# Запустить bash
docker exec -it reddit bash

# Удалить контейнер
docker rm reddit

# Запуск контейнера без запуска приложения
docker run --name reddit --rm  -it loktionovam/otus-reddit:1.0 bash
root@96aba478ca67:/# ps ax | grep start
   16 pts/0    S+     0:00 grep --color=auto start

# Полная информация об образе
docker inspect loktionovam/otus-reddit:1.0 

# Часть информации об образе
docker inspect loktionovam/otus-reddit:1.0 -f '{{.ContainerConfig.Cmd}}'
[/bin/sh -c #(nop)  CMD ["/start.sh"]]

#Вывод списка измененных файлов и каталогов в контейнере
docker diff reddit
```

### 13.2.2 Настройка infra. Подготовительные действия - создание ssh ключей и сервисного аккаунта в GCP

```bash
export REPO_PATH=$(pwd)
export GCP_PROJECT=docker-project-name-here
ssh-keygen -t rsa -f ~/.ssh/docker-user -C 'docker-user' -q -N ''

gcloud iam service-accounts create docker-user --display-name docker-user

gcloud projects add-iam-policy-binding "${GCP_PROJECT}" --member serviceAccount:docker-user@"${GCP_PROJECT}".iam.gserviceaccount.com --role roles/editor
```

### 13.2.2 Настройка infra. Создание образа docker-host-base с помощью packer

```bash
# Создание образа хоста с предустановленным docker engine
cd "${REPO_PATH}"/docker-monolith/infra
packer build -var-file=packer/variables.json packer/docker_host.json
```

### 13.2.3 Настройка infra. Создание инстансов docker host через Terraform

```bash
cd "${REPO_PATH}"/docker-monolith/infra/terraform
# Перед выполением команд нужно настроить terrafrom.tfvars файл для создания remote backend
terraform init
terraform apply

cd "${REPO_PATH}"/stage
# Перед выполением команд нужно настроить terrafrom.tfvars файл для создания инстансов docker host и правил файерволла
terraform init
terrafrom apply
```

### 13.2.2 Настройка infra. Запуск reddit app

```bash
cd "${REPO_PATH}"/docker-monolith/infra/ansible
# Скопировать файл с секретными данными от service account
gcloud iam service-accounts keys create environments/stage/gce-service-account.json --iam-account docker-user@"${GCP_PROJECT}".iam.gserviceaccount.com

# Настроить gce dynamic inventory
ansible-playbook playbooks/gce_dynamic_inventory_setup.yml

# Развернуть reddit app на инстансах созданных ранее, через terraform
ansible-playbook playbooks/site.yml
```

### 13.3 Как проверить проект

- Приложение будет доступно в веб-браузере по адресу http://ip_address:9292, где список ip_address можно узнать командами

```bash
# Получить список всех ip адресов
cd "${REPO_PATH}"/docker-monolith/infra/terraform/stage
terraform output
```

## Homework-14: Docker образы. Микросервисы

### 14.1 Что было сделано

Основные задания:

- Созданы docker образы для микросервисов comment, ui, post
- Создана docker сеть для приложения reddit
- Создан docker том для данных mongodb
- Запущены контейнеры на основе созданных образов

Задания со *:

- Изменение сетевых алиасов, использование env переменных

```bash
# Пример решения
docker run -d --network=reddit --network-alias=post_db_alias --network-alias=comment_db_alias mongo:latest
docker run -d --network=reddit --network-alias=post_alias -e 'POST_DATABASE_HOST=post_db_alias' loktionovam/post:1.0
docker run -d --network=reddit --network-alias=comment_alias -e 'COMMENT_DATABASE_HOST=comment_db_alias' loktionovam/comment:1.0
docker run -d --network=reddit --network-alias=ui_alias -e 'POST_SERVICE_HOST=post_alias' -e 'COMMENT_SERVICE_HOST=comment_alias' loktionovam/ui:1.0
```

Задания со *:

- Уменьшены размеры образов comment, ui, post

```bash
# Размеры образов ui
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
loktionovam/ui      3.2                 b92f70287088        11 seconds ago      34.5MB # alpine, multi-stage build, cache cleaning
loktionovam/ui      3.1                 8e94d1738f8f        4 minutes ago       37.5MB # alpine, multi-stage build
loktionovam/ui      3.0                 0054caafcade        14 minutes ago      210MB # alpine
loktionovam/ui      2.0                 5f494c53de90        23 minutes ago      460MB # ubuntu 16.04
loktionovam/ui      1.0                 1dc4afe3d94c        26 minutes ago      777MB # ruby 2.2
```

```bash
# Размеры образов post
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
loktionovam/post    2.2                 e13a2539eef3        6 minutes ago       35.1MB # alpine:3.8, multi-stage build, venv packages cleaning, pyc files removing
loktionovam/post    2.1                 a4978e47831e        13 minutes ago      57.5MB # alpine:3.8, multi-stage build, venv packages cleaning
loktionovam/post    2.0                 324095723244        21 minutes ago      62.6MB # alpine:3.8, multi-stage build
loktionovam/post    1.0                 228a932d5c0d        3 hours ago         102MB # python:3.6.0-alpine
```

```bash
# Размеры образов comment
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
loktionovam/comment   2.0                 d42d889bed54        18 seconds ago      30.1MB # alpine, multi-stage build, cache cleaning
loktionovam/comment   1.0                 d1e034889328        4 hours ago         769MB # ruby 2.2
```

### 14.2 Как запустить проект

- Предполагается, что перед запуском проекта уже существует **docker-host** и имеет адрес **docker_host_ip**

```bash
docker-machine ls
NAME          ACTIVE   DRIVER   STATE     URL                       SWARM   DOCKER        ERRORS
docker-host   *        google   Running   tcp://docker_host_ip:2376           v18.06.0-ce

eval $(docker-machine env docker-host)
```

- Создание docker образов микросервисов comment, post, ui

```bash
cd src
docker build -t loktionovam/comment:2.0  ./comment
docker build -t loktionovam/post:2.2  ./post-py
docker build -t loktionovam/ui:3.2  ./ui
```

- Запуск проекта

```bash
docker network create reddit
docker volume create reddit_db

docker run -d --network=reddit --network-alias=post_db \
 --network-alias=comment_db -v reddit_db:/data/db mongo:latest

docker run -d --network=reddit --network-alias=post loktionovam/post:2.2

docker run -d --network=reddit --network-alias=comment loktionovam/comment:2.0

docker run -d --network=reddit -p 9292:9292 loktionovam/ui:3.2
```

### 14.3 Как проверить проект

После запуска, reddit приложение будет доступно по адресу <http://docker_host_ip:9292>, при этом можно создать пост и оставить комментарий.

## Homework-15: Docker сети, docker-compose

Основные задания:

- Работа с none, host, bridge сетями docker
- Установка docker-compose
- Сборка образов приложения reddit с помощью docker-compose
- Запуск приложения reddit с помощью docker-compose
- Изменение префикса в docker-compose

В docker-compose имена префиксов контейнеров задаются env переменной **COMPOSE_PROJECT_NAME**, которая по умолчанию равна названию каталога с проектом:

```bash
cd /some/path/to/project_directory
basename $(pwd)
```

Эту переменную можно переопределить в **.env** файле:

```bash
# Defaul setting COMPOSE_PROJECT_NAME to the basename
# Change this value to setup containers prefix name
# COMPOSE_PROJECT_NAME=
```

Задания со *:

- Создание docker-compose.override.yml, который позволяет изменять код приложений без пересборки docker образов и запускать ruby приложения в режиме отладки с двумя воркерами.

### 15.1 Что было сделано

- Создан контейнер с none network driver, проверена конфигурация его сетевых интерфейсов

- Создан контейнер с host network driver, проверена конфигурация его сетевых интерфейсов

- Созданы контейнеры с bridge network driver, которым были присвоены сетевые алиасы

- Созданы docker сети front_net (подключены контейнеры ui, post, comment), back_net (подключены контейнеры mongo, post, comment)

- Проверена работа сетевого стека linux (net namespaces, iptables) при работе с docker

- Установлен docker-compose, написан docker-compose.yml для автоматического создания и запуска docker ресурсов

- Создан .env файл для настройки docker-compose через environment переменные (версии образов, префикс проекта и т. д.)

- Создан docker-compose.override.yml позволяющий изменять код приложения без пересоздания образа и запускать puma сервер в режиме отладки. Соответственно, изменены Dockerfile в микросервисах comment, ui, post для поддержки docker-compose.override.yml

### 15.2 Как запустить проект

- Предполагается, что перед запуском проекта уже существует **docker-host** и имеет адрес **docker_host_ip**, а также установлен **docker-compose**

```bash
docker-machine ls
NAME          ACTIVE   DRIVER   STATE     URL                       SWARM   DOCKER        ERRORS
docker-host   *        google   Running   tcp://docker_host_ip:2376           v18.06.0-ce

eval $(docker-machine env docker-host)
```

```bash
cd src
docker-compose up -d
```

### 15.3 Как проверить проект

После запуска, reddit приложение будет доступно по адресу <http://docker_host_ip:9292>, при этом можно создать пост и оставить комментарий.

## Homework-16: Gitlab CI. Построение процесса непрервыной интеграции

Основные задания: подготовить инсталляцию Gitlab CI, подготовить репозиторий с кодом приложения, описать для приложения этапы непрервыной интеграции

Задания со *: автоматизировать развертывание и регистрацию gitlab runner, добавить интеграцию pipeline со slack чатом

### 16.1 Что было сделано

- Добавлен boot disk size параметр в конфигурацию модуля docker_host в terraform, добавлена конфигурация файерволла для docker host в terraform

- Добавлена поддержка docker compose для ansible роли docker_host

- Пересобран packer образ для поддержки docker compose docker-host-base

- Добавлена gitlab_omnibus ansible роль для автоматического развертывания сервера с gitlab, для роли написаны тесты с использованием molecula и testinfra, добавлен healthchek в конфигурацию docker compose

- Добавлен gitlab_omnibus.yml плейбук для развертывания инстанса gitlab в GCP

- Добавлена конфигурация пайплана gitlab

- Добавлена gitlab_runner ansible роль для автоматического развертывания gitlab runner, для роли написаны тесты с использованием molecula и testinfra

- Добавлен gitlab_runner.yml плейбук для автоматического развертывания и регистрации gitlab runner

- Добавлено reddit приложение и тесты для него, настроен запуск тестов в gitlab CI/CD

- Добавлена интеграция со slack чатом <https://devops-team-otus.slack.com/messages/CB4BAETU5/>

### 16.2 Как запустить проект

Предполагается, что уже существует конфигурация terraform настроенная в рамках выполнения **Homework-13**

- Собрать новый образ docker-host-base с поддержкой docker compose через packer

```bash
cd infra
packer build -var-file=packer/variables.json packer/docker_host.json
```

- Настроить boot size в terraform и переразвенуть docker хост

```bash
cd infra/terraform/stage
# настроить size = 50 в terraform.tfvars
# и пересоздать docker host
terraform taint -module=docker_host google_compute_instance.docker_host
terraform apply -auto-approve
terraform output
```

- Установить gitlab сервер

```bash
cd gitlab-ci/ansible
ansible-playbook playbooks/gitlab_omnibus.yml
```

- Настроить пользователя, проект и т. д. в gitlab сервере

- Установить и зарегистрировать gitlab runner. Перед запуском плейбука необходимо настроить в `~/.ansible/gilab_runner_credentials.yml` переменные `gitlab_runner_token` и `gitlab_runner_coordinator_url`

```bash
cd gitlab-ci/ansible
ansible-playbook playbooks/gitlab_runner.yml
```

### 16.3 Как проверить проект

После настройки CI/CD в gitlab, можно запушить изменения и проверить, что статус пайплайна будет **passed**

```bash
git remote add gitlab http://<your-vm-ip>/homework/example.git
git push gitlab gitlab-ci-1
```

## Homework-17: Устройство Gitlab CI. Непрерывная поставка

Основные задания: расширить существующий пайплайн в gilab ci, определить окружения

Задания со *: при пуше новой ветки должен создаваться сервер для окружения с возможностью удалить его кнопкой

Задания с **: в шаг build добавить сборку контейнера с приложением reddit, контейнер деплоить на созданный для ветки сервер

### 17.1 Что было сделано

- В конвейер непрервывной поставки были добавлены шаги **staging, production** использующие различные окружения

- В ansible создана роль **reddit_monolith** для деплоя контейнера на docker хост

- Изменен плейбук **reddit_app.yml** с учетом **reddit_monolith**

- Написан Dockerfile для сборки docker образа reddit приложения

- Написан Dockerfile для сборки docker образа провижинера приложения, содержащий в себе terraform, ansible

- В docker_host module (terraform) добавлены ресурсы для провижинга приложения

- В шаг build добавлена сборка образов reddit, app_provision с их последующим сохранением в docker registry

- В конвейер непрервывной поставки был добавлен шаг **branch_start_review** для автоматического создания серверов для каждой ветки и деплоя приложения reddit. При этом, для каждой ветки через terrafrom workspace создается инстанс docker хоста, на который плейбуком reddit_app.yml деплоится reddit приложение

- В конвейер непрервывной поставки был добавлен шаг branch_stop_review для удаления серверов

### 17.2 Как запустить проект

Предполагается, что уже настроен gitlab сервер и зарегистрирован gitlab runner

В настройках CI/CD gitlab server нужно добавить следующие переменные:

- `CI_GOOGLE_CREDENTIALS` - переменная для подключения terraform к GCP

- `DOCKER_REGISTRY_USER` - пользователь docker hub registry

- `DOCKER_REGISTRY_PASSWORD` - пароль docker hub registry

- `GCE_SERVICE_ACCOUNT` - содержимое credentials файла от сервисного аккаунта GCP. Используется для настройки динамического инвентори через gce.py (роль ansible gce_py)

- `GCP_PROJECT` - название проекта в GCP

- `SSH_PRIVATE_KEY`, `SSH_PUBLIC_KEY` - пара ssh ключей для подключения к docker host (используются при провиженге через ansible)

После настройки environment переменных при пуше новой ветки в gitlab будет автоматичкски создано окружение и на него развернуто reddit приложение

### 17.3 Как проверить проект

После успешного выполнения шага **branch_start_review**, приложение будет доступно по адресу <http://docker_host_external_ip:9292>

```bash

Outputs:

docker_host_external_ip = [
    35.240.19.232
]
Job succeeded
```

После проверки приложения, сервер можно удалить нажав на **branch_stop_review**

## Homework-18: Введение в мониторинг. Системы мониторинга

Основное задание: запуск, конфигурация Prometheus; мониторинг состояния микросервисов; сбор метирк хоста с использованием экспортера

Задание со *: добавить в Prometheus мониторинг mongodb, зафиксировав версию образа экспортера на последнюю стабильную

Задание со *: добавить в Prometheus blackbox мониторинг сервисов comment, post, ui, зафиксировав версию образа экспортера на последнюю стабильную

Задание со *: добавить Makefile, которые умеет собирать и пушить образы docker контейнеров

### 18.1 Что было сделано

- Изменена структура каталогов проекта, добавлены каталоги docker, monitoring

- В конфигурацию docker compose добавлен сервис prometheus

- В конфигурацию prometheus и docker compose добавлен node-exporter

- Добавлен Dockerfile для сборки mongodb prometheus exporter от percona (результирующий образ построен на scratch), добавлен мониторинг mongodb в prometheus и docker compose

- Добавлен Dockerfile для сборки blackbox exporter; добавлен мониторинг микросервисов ui (blackbox-http), post (blackbox-tcp), comment (blackbox-tcp)

- Добавлен Makefile для сборки образов docker контейнеров и их пуша в docker hub; добавлена возможность старта и удаления микросервисов через Makefile

### 18.2 Как запустить проект

- Предполагается, что уже установлен **docker, docker compose**

- Установить make

```bash
apt-get update && apt-get install make
```

- Собрать образы контейнеров и локально запустить их через docker compose можно командой

```bash
make up
```

- Собрать образы контейнеров и запушить их в docker hub репозиторий можно командой

```bash
make all
```

- Остановить и удалить запущенные контейнеры

```bash
make down
```

Примеры работы с отдельными микросервисами:

- Собрать docker образ микросервиса (ui) можно командой

```bash
make ui_build
```

- Запушить docker образ микросервиса (ui) можно командой

```bash
make ui_push
```

- Собрать и запушить образ отдельного микросервиса (ui)

```bash
make ui
```

### 18.3 Как проверить проект

- В корне репозитория выполнить

```bash
make up
```

После сборки и запуска контейнеров по адресу <http://localhost:9292> будет доступно reddit приложение

По адресу <http://localhost:9090> будет доступен интерфейс prometheus

- В корне репозитория выполнить

```bash
make all
```

После сборки и пуша, образы контейнеров будут доступны в dockerhub по ссылке: <https://hub.docker.com/r/loktionovam>

## Homework-19: Мониторинг приложения и инфраструктуры

Основное задание: мониторинг docker контейнеров; визуализация метрик; сборк метрик работы приложения и бизнес метрик; настройка и проверка алертинга

Задание со *:

<https://grafana.com/dashboards/1229>
```bash
# Docker native metrics
curl http://172.17.0.1:9323/metrics 2>/dev/null| grep -E "^# " -v | wc -l
339
```

```bash
# Cadvisor metrics
curl http://localhost:8080/metrics 2>/dev/null| grep -v "^# " | wc -l
3950

total: 58

```

```bash
# telegraf docker metrics
grep -v "#"  metrics  | grep docker | wc -l 
926
```

### 19.1 Что было сделано

### 19.2 Как запустить проект

### 19.3 Как проверить проект
