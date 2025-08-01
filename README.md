# Домашнее задание к занятию "`GitLab`" - `Ларионов Сергей`

### Задание 1

`Развертывание GitLab и настройка Runner`

1. `Установка зависимостей`

```
# Обновление системы
sudo apt update && sudo apt upgrade -y

# Установка VirtualBox
sudo apt install virtualbox -y

# Установка Vagrant
sudo apt install vagrant -y

# Установка Docker
sudo apt install docker.io -y
sudo systemctl enable --now docker

# Установка GitLab Runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install gitlab-runner -y
```

2. `Настройка Vagrant для GitLab - Создаем Vagrantfile`

```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "gitlab.localdomain"
  
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "forwarded_port", guest: 443, host: 8443
  config.vm.network "forwarded_port", guest: 22, host: 2222
  
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4096"
    vb.cpus = 2
  end
  
  config.vm.provision "shell", inline: <<-SHELL
    # Установка Docker
    curl -fsSL https://get.docker.com | sh
    
    # Запуск GitLab в Docker
    docker run -d \
      --hostname gitlab.localdomain \
      --publish 443:443 --publish 80:80 --publish 22:22 \
      --name gitlab \
      --restart always \
      --volume /srv/gitlab/config:/etc/gitlab \
      --volume /srv/gitlab/logs:/var/log/gitlab \
      --volume /srv/gitlab/data:/var/opt/gitlab \
      gitlab/gitlab-ce:latest
  SHELL
end
```

3. `Запускаем виртуальную машину`

```
vagrant up
```

4. `Регистрация GitLab Runner`

`После запуска GitLab (может занять несколько минут), заходим в веб-интерфейс (http://localhost:8080), создаем проект и получаем регистрационный токен и регистрируем Runner`

```
docker run --rm -it \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  gitlab/gitlab-runner register \
  --non-interactive \
  --executor "docker" \
  --docker-image alpine:latest \
  --url "http://gitlab.localdomain" \
  --registration-token "YOUR_PROJECT_TOKEN" \
  --description "docker-runner" \
  --tag-list "docker,runner" \
  --run-untagged="true" \
  --locked="false" \
  --access-level="not_protected"
```

5. `Запускаем Runner`

```
docker run -d \
  --name gitlab-runner \
  --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

`Вместо скриншотов лучшем решением будет показать работу с последовательными действиями
[Домашняя работа](https://docs.google.com/document/d/1Midc2AmLL-5gOH28fHwvYc7PandWz5tq/edit?usp=drive_link&ouid=115067957742774085873&rtpof=true&sd=true)`


---

### Задание 2

`Настройка CI/CD pipeline`

1. `Создаем файл .gitlab-ci.yml`

```
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TAG: latest
  CI_DEBUG_TRACE: "false"

cache:
  paths:
    - node_modules/
    - build/

before_script:
  - echo "Starting pipeline for $CI_PROJECT_NAME"
  - docker info

build_job:
  stage: build
  tags:
    - docker
  script:
    - echo "Building Docker image..."
    - docker build -t $CI_REGISTRY_IMAGE:$DOCKER_TAG .
    - docker tag $CI_REGISTRY_IMAGE:$DOCKER_TAG $CI_REGISTRY_IMAGE:latest
  artifacts:
    paths:
      - build/
    expire_in: 1 week

unit_tests:
  stage: test
  tags:
    - docker
  script:
    - echo "Running unit tests..."
    - docker run --rm $CI_REGISTRY_IMAGE npm test
  allow_failure: false

integration_tests:
  stage: test
  tags:
    - docker
  script:
    - echo "Running integration tests..."
    - docker-compose -f docker-compose.test.yml up --abort-on-container-exit
  allow_failure: false
  needs: ["unit_tests"]

deploy_staging:
  stage: deploy
  tags:
    - docker
  script:
    - echo "Deploying to staging..."
    - kubectl apply -f k8s/staging.yaml
  when: manual
  only:
    - main

deploy_prod:
  stage: deploy
  tags:
    - docker
  script:
    - echo "Deploying to production..."
    - kubectl apply -f k8s/production.yaml
  when: manual
  only:
    - tags
```

2. `Команды для работы с репозиторием`

```
# Клонирование репозитория
git clone http://gitlab.localdomain/username/project.git
cd project

# Изменение origin (если нужно)
git remote set-url origin http://gitlab.localdomain/username/project.git

# Добавление файлов и коммит
touch README.md
echo "# My Project" >> README.md
git add .
git commit -m "Initial commit"

# Отправка изменений
git push -u origin main
```
 
`Проверка работы`

```
После push в репозиторий автоматически запустится pipeline

В интерфейсе GitLab (http://localhost:8080) можно увидеть:

Список pipeline во вкладке CI/CD -> Pipelines

Детали выполнения каждого этапа

Артефакты сборки (если определены)

Для успешного выполнения заданий убедитесь, что:

Виртуализация включена в BIOS

Достаточно ресурсов (минимум 4GB RAM)

Порт 8080 не занят на хостовой машине

Docker правильно установлен и текущий пользователь добавлен в группу docker
```

`Вместо скриншотов лучшем решением будет показать работу с последовательными действиями
[Домашняя работа](https://docs.google.com/document/d/1Midc2AmLL-5gOH28fHwvYc7PandWz5tq/edit?usp=drive_link&ouid=115067957742774085873&rtpof=true&sd=true)`


---
