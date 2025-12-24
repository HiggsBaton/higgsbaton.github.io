---
layout: post
title:  "Сказ о сервере Minecraft"
date:   2025-11-20
categories: devops family
---

Жил был Володя и было у него три сына.

Как-то раз собрался он с собакой гулять, да дети его остановили, говорят подожди, батюшка, коль собрался ты в дорогу, просьбы есть у нас.

Проорал все уши младшенький, подбежал и говорит ему первый:
- "Мутик хочу!"
Подумал Володя и сказал:
- "Хорошо, малыш, Включу я тебе мультик", и включил.

Протопал по всей квартире как бегемотик средненький и говорит:
- "Папа, дай телефон!"
Снова задумался отец, и подумав мало ли, много ли времени, согласился.
- "Хорошо, сынок, держи мобильный телефон".

Подошёл к нему и старшенький, и говорит:
- "Родной мой батюшка, не нужен мне ни телефон, ни мультики, а нужен мне сервер, чтобы с другом своим школьным в Minecraft поиграть."
Задумался Вова пуще прежнего. Мало ли, много ли времени он думал, неизвестно, а известно что сказал он ему так:
- "Задал ты мне задачку потяжелее братьев своих, да постараюсь и тебе угодить".

И вот, собрался он с мыслями и поднял сервер.

В первой итерации всё было сделано более менее вручную:
1. Поднял сервер в `timeweb`
2. С сайта `minecraft.net` скачал сервер
3. Запустил его, отредактировал настройки, посмотрел, что работает

... вручную не очень удобно и не очень интересно, так что ...

Написал несколько `.sh` скриптов, чтобы имея IP сервера можно было его настроить через использование этих скриптов.

В начале напишем Dockerfile, чтобы потом копировать на сервер образ с нужной версией игры:
Важно отметить, что запускаясь в первый раз, сервер не поднимается и просит принятия EULA, что делается редактурой текстового файлика eula.txt.
Поэтому я взял всю папку сервера, который в первый раз запустил и поместил её в контейнер.
```
FROM ubuntu:24.04
RUN apt update && apt install default-jdk default-jre -y
COPY ./minecraft_server /root/minecraft_server
WORKDIR /root/minecraft_server/
ENTRYPOINT ["java", "-Xmx1024M", "-Xms1024M", "-jar", "server.jar", "nogui"]
```

Собрав этот образ, я сделал вот такой скрипт для копирования образа контейнера с сервером и папки с миром на сервер:
```
scp -i ~/.ssh/id_ed25519_tw ./minecraft.tar root@$1:/root
scp -i ~/.ssh/id_ed25519_tw -r ./world root@$1:/root

# docker must be installed on the machine
#scp -i ~/.ssh/id_ed25519_tw ./docker.sh root@$1:/root/docker.sh

ssh -i ~/.ssh/id_ed25519_tw root@$1 docker load -i minecraft.tar

ssh -i ~/.ssh/id_ed25519_tw root@$1 docker run -d -p 25565:25565 --restart always -v /root/world:/root/minecraft_server/world minecraft:1.21.20
```

В результате сервер определённой версии поднимался и всё работало.
Для сохранения прогресса можно просто копировать папку world на сервер и обратно, чтобы при следующем запуске её можно было использовать.

И тут я подумал, что хранить всё лучше в гите и убрать необходимость ручного создания вервера в `timeweb` и запуска скриптов настройки можно через Gitlab CI/CD.
Так что создал репозиторий.

В этот момент также появилась необходимость менять версии игры, обновляя версию сервера тоже, и использовать разные папки `world` с мирами для того, чтобы играть в adventure-карты.

Получилось, что наш репозиторий должен выполнять вот такие шаги:
```
stages:
  - docker-image # собираем образ с сервером и загружаем в container registry
  - upload-files # загружаем папку world в package registry
  - create-server # создаём сервер и настраиваем
  - destroy-server # удаляем сервер
```

Теперь подробнее про каждый stage.

---
1. docker-image
---

Тут кажется всё понятно, собираем образ latest и загружаем его, но есть небольшой скрипт, который позволяет выбрать ту версию, которая нужна.

Вот сам stage:
```
build-upload-docker-image: # собираем образ с сервером и загружаем в container registry
  stage: docker-image
  before_script:
    - cd docker
  script:
    - echo "${CI_REGISTRY_PASSWORD}" | docker login registry.gitlab.com -u ${CI_REGISTRY_USER} --password-stdin
    - docker build -t registry.gitlab.com/boguslavskiy/myminecraftserver --build-arg MINECRAFT_VERSION=${MINECRAFT_VERSION} .
    - docker push registry.gitlab.com/boguslavskiy/myminecraftserver
  when: manual
```

Всё, что связано со сборкой образа я вынес в папку `docker`.

Для adventure-карт нужно искать определённую версию сервера, на сайте `minecraft.net` я списка серверов с разными версиями не нашёл, но гугл помог найти старницу `mcversions.net`.

Для получения образа с определённой версией игры, я набросал вот такой скрипт:
```
# download the server jar

# if no arguments passed, download latest version from mcversions.net, otherwise look for the required version
if [ "$1" = "latest" ]; then
    REQUIRED_VERSION=$( curl https://mcversions.net | awk -F "Latest Release" '{print $2}' | awk -F "<br>" '{print $1}' | awk -F "</span>" '{print $3}' )
else
    REQUIRED_VERSION=$1
fi

VERSION_CODE=$( curl https://mcversions.net/download/${REQUIRED_VERSION} | awk -F "/server.jar" '{print $1}' | awk -F "https://piston-data.mojang.com/v1/objects/" '{print $2}' )

wget "https://piston-data.mojang.com/v1/objects/${VERSION_CODE}/server.jar"

java -Xmx1024M -Xms1024M -jar server.jar nogui
```

awk тут конечно не очень красиво выглядят, но то, что мне нужно я получаю :)
Ну а запуск сервера этот тоже костыль для того, чтобы потом, запустив его повторно просто указать ему пути к настройкам, принятой EULA и папку world.

Этот скрипт я использую в изменённом Dockerfile, чтобы при сборке образа можно было указать какая версия нам нужна или просто искать самую последнюю:
```
FROM ubuntu:24.04
ARG MINECRAFT_VERSION="latest"
ENV MINECRAFT_VERSION=${MINECRAFT_VERSION}
WORKDIR /root
COPY ./server_jar_download.sh ./server_jar_download.sh
RUN apt update && apt install default-jdk default-jre curl wget -y && ./server_jar_download.sh "${MINECRAFT_VERSION}"
ENTRYPOINT ["java", "-Xmx1024M", "-Xms1024M", "-jar", "server.jar", "nogui"]
```

---
2. upload-files
---

Тут я упаковываю сначала папку с миром игры, а потом и папку с настройками сервера в zip архивы и отправляю их на хранение с указанием версии сервера и названия мира.

```
upload-world: # загружаем папку world в package registry
  stage: upload-files
  script:
    - zip -r world.zip ./configs/${WORLD}/world
    - |
      curl --location --header "JOB-TOKEN: ${CI_JOB_TOKEN}" \
           --upload-file world.zip \
           "https://gitlab.com/api/v4/projects/${CI_PROJECT_ID}/packages/generic/${WORLD}/${MINECRAFT_VERSION}/world.zip"
  when: manual

upload-properties: # загружаем папку с настройками в package registry
  stage: upload-files
  script:
    - zip -r server.properties.zip ./configs/${WORLD}/server.properties
    - |
      curl --location --header "JOB-TOKEN: ${CI_JOB_TOKEN}" \
           --upload-file server.properties.zip \
           "https://gitlab.com/api/v4/projects/${CI_PROJECT_ID}/packages/generic/${WORLD}/${MINECRAFT_VERSION}/server.properties.zip"
  when: on_success
  needs:
    - upload-world
```

Сделал отдельную папку `configs` в репозитории, чтобы было удобнее хранить там разные миры и разные конфиги для серверов.

Выглядит это так:
```
├── configs
│   ├── base_world
│   │   ├── server.properties
│   │   └── world
│   ├── sunny_goo_bloom
│   │   ├── other
│   │   ├── server.properties
│   │   └── world
│   └── whirlpool
│       ├── server.properties
│       └── world
```

---
3. create-server
---

Теперь у нас есть всё, что нужно для того, чтобы включить сервер.
Кроме самого сервера :)
Нам нужно создать и настроить сервер.

Я делал это в `timeweb`, так как изначально пользовался им, чтобы посмотреть, как он работает с ~~`terraform`~~ `opentofu`.

Сначала нужно создать сервер и получить его IP для дальнейшей работы с ним.
Почитав вывод `opentofu` я увидел, там уже указан IP нашего нового сервера, так что снова использую awk, чтобы выделить именно его.
```
create-server:
  stage: create-server
  image: opentofu:latest
  before_script:
    - cd tofu
    - echo "timeweb_key = "'"'"${TIMEWEB_KEY}"'"' > terraform.tfvars
    - echo "" >> terraform.tfvars
    # also we can provide other variables here
  script:
    - tofu init
    - tofu apply -auto-approve > tofu_output
    - IP=$( tail -n 4 tofu_output | awk -F "]" '{print $1}' | awk -F "-" '{print $2}' )
    - echo $IP
    - echo "IP=$IP" >> ../build.env
  artifacts:
    paths:
      - tofu/terraform.tfstate
      - tofu/terraform.tfvars
    reports:
      dotenv: build.env
    when: on_success
  when: manual
```

Для второго шага в `timeweb` нужно настроить ssh-ключ, который добавляется на сервер по умолчанию при его создании, и тогда можно использовать его, чтобы настроить сервер через `ansible `.

```
setup-server:
  stage: create-server
  image: ansible
  before_script:
    # setup rsa key
    - echo "${SSH_PRIVATE_KEY}" > tw_key_coded
    - base64 -d -i tw_key_coded -o ansible/tw_key
    - chmod 700 ansible/tw_key
    # setup community.docker
    - ansible-galaxy collection install community.docker
    # setup ansible variables
    - echo $IP >> ansible/inventory.ini
    - echo "gitlab_registry_url"':'" "'"'registry.gitlab.com'"' > ansible/vars/external_vars.yml
    - echo "gitlab_username"':'" "'"'Boguslavskiy'"' >> ansible/vars/external_vars.yml
    - echo "gitlab_token"':'" "'"'${TOKEN}'"' >> ansible/vars/external_vars.yml
    - echo "image_name"':'" "'"'boguslavskiy/myminecraftserver'"' >> ansible/vars/external_vars.yml
    - echo "image_tag"':'" "'"'latest'"' >> ansible/vars/external_vars.yml
    - echo "gitlab_package_url"':'" "'"'https://gitlab.com/api/v4/projects/${CI_PROJECT_ID}/packages/generic/${WORLD}/${MINECRAFT_VERSION}/'"' >> ansible/vars/external_vars.yml
    - cd ansible
  script:
    - ansible-playbook -i inventory.ini playbook_setup.yml -u root --key-file tw_key
    - ansible-playbook -i inventory.ini playbook_get_image.yml -u root --key-file tw_key
  artifacts:
    paths:
      - ansible/inventory.ini
      - ansible/vars/external_vars.yml
      - ansible/tw_key
  dependencies:
    - create-server
  when: on_success
  needs:
    - create-server
```

`ansible` скачивает образ, папку world и настройки сервера.

Скачиваем папку world и настройки:
```
- name: Install unzip
  ansible.builtin.apt:
    name: unzip

- name: Download the world package file
  ansible.builtin.get_url:
    url: "{{ gitlab_package_url }}/world.zip"
    dest: "/root/world.zip"
    headers:
      Private-Token: "{{ gitlab_token }}"
    mode: "0644"

- name: Unarchive the world package file
  ansible.builtin.unarchive:
    src: /root/world.zip
    dest: /root/
    remote_src: yes

- name: Download the settings package file
  ansible.builtin.get_url:
    url: "{{ gitlab_package_url }}/server.properties.zip"
    dest: "/root/server.properties.zip"
    headers:
      Private-Token: "{{ gitlab_token }}"
    mode: "0644"

- name: Unarchive the world package file
  ansible.builtin.unarchive:
    src: /root/server.properties.zip
    dest: /root/
    remote_src: yes
```

Скачиваем образ:
```
- name: Log in to GitLab Container Registry
  community.docker.docker_login:
    registry_url: "{{ gitlab_registry_url }}"
    username: "{{ gitlab_username }}"
    password: "{{ gitlab_token }}"
    reauthorize: yes

- name: Pull the Docker image
  community.docker.docker_image:
    name: "{{ gitlab_registry_url }}/{{ image_name }}"
    tag: "{{ image_tag }}"
    source: pull
```

Когда с этим закончено, мы можем запустить сервер, указав пути для volumes:
```
start-server:
  stage: create-server
  script:
    - ssh -i ~/.ssh/tw_key root@$IP mv /root/configs/${WORLD}/* /root/
    - |
      ssh -i ~/.ssh/tw_key root@$IP docker run -d -p 25565:25565 --name minecraft --restart always \
        -v /root/world:/root/world \
        -v /root/server.properties/banned-ips.json:/root/banned-ips.json \
        -v /root/server.properties/banned-players.json:/root/banned-players.json \
        -v /root/server.properties/eula.txt:/root/eula.txt \
        -v /root/server.properties/ops.json:/root/ops.json \
        -v /root/server.properties/server.properties:/root/server.properties \
        -v /root/server.properties/usercache.json:/root/usercache.json \
        -v /root/server.properties/whitelist.json:/root/whitelist.json \
        registry.gitlab.com/boguslavskiy/myminecraftserver:latest
  dependencies:
    - create-server
    - setup-server
  when: on_success
  needs:
    - create-server
    - setup-server
```

---
4. destroy-server
---

Удаляю сервер я тоже через `opentofu`, но перед удалением можно загрузить в package registry папку world и настройки, чтобы сохранить изменения.
Вот так выглядит ansible-роль:
```
- name: Install gzip
  ansible.builtin.apt:
    name: gzip

- name: Compress world diretory
  community.general.archive:
    path: "/root/world"
    dest: "/root/world.zip"

- name: Fetch the world.zip
  ansible.builtin.fetch:
    src: /root/world.zip
    dest: world.zip
    flat: yes

- name: Upload package to GitLab Generic Package Registry
  ansible.builtin.uri:
    url: "{{ gitlab_package_url }}/world.zip"
    method: PUT
    headers:
      Private-Token: "{{ gitlab_token }}"
    status_code: [200, 201]
    src: "world.zip"

- name: Compress server.properties diretory
  community.general.archive:
    path: "/root/server.properties"
    dest: "/root/server.properties.zip"

- name: Fetch the server.properties.zip
  ansible.builtin.fetch:
    src: /root/server.properties.zip
    dest: server.properties.zip
    flat: yes

- name: Upload package to GitLab Generic Package Registry
  ansible.builtin.uri:
    url: "{{ gitlab_package_url }}/server.properties.zip"
    method: PUT
    headers:
      Private-Token: "{{ gitlab_token }}"
    status_code: [200, 201]
    src: "server.properties.zip"
```

А вот jobs:
```
prepare-destroy:
  stage: destroy-server
  image: ansible
  before_script:
    - ansible-galaxy collection install community.general
    - cd ansible
  script:
    - ansible-playbook -i inventory.ini playbook_save_setup.yml -u root --key-file tw_key
  dependencies:
    - setup-server
  needs:
    - setup-server
  when: manual

destroy-server:
  stage: destroy-server
  image: opentofu:latest
  before_script:
    - cd tofu
  script:
    - tofu init
    - tofu destroy -auto-approve
  dependencies:
    - create-server
  when: manual
```

---

Посмотреть как выглядит весь репозиторий можно [тут]()

Чтобы подытожить, напишу, что пока ещё процесс меняется, сохранение прогресса (папки world и настроек) оказалось не очень нужным, а вот выбор версии сервера и заменяемость миров (для adventure-карт) оказались очень удобны.

Следующий этап - установка OBS (и/или другого ПО для записи геймплея) и запись прохождений карт :)

Вот и сказочке конец, а кто читал - тот молодец!
