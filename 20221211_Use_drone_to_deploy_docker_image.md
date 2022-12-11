---
author: Maksim Panin
datetime: 2022-12-11 15:24:21
title: Deploying docker image with drone
slug: deploying-docker-image-with-drone
featured: true
draft: true
tags:
  - docker
  - drone
  - ci/cd
  - devops
ogImage: ""
description:
  
---

# Using drone to deploy docker image

For my small pet project to learn about RabbitMQ I also wanted to create CI/CD pipeline using Drone on my home server. So I have two very simple microservice written in Python. One of them every day calculates how many days past since I meet my girlfriend and sends this information to RabbitMQ queue. The second one reads this information from the queue and sends it to Telegram chat.

For the home server, I'm using an old Macbook Pro. I installed Docker, Gitea, Nexus, and Drone on it.

So basically, my pipeline should do the following after pushing code to Gitea:

1. Run tests
2. Build docker image
3. Push docker image to private Nexus docker registry
4. Deploy docker image to the same server.

## 1. Set up drone config. 

Drone stores all pipelines in .drone.yml file. This file should be in the root of your repository. I have two repositories for my microservices. So I have two .drone.yml files. They are very similar, so I'll focus on one of them.

My pipeline is very straightforward. It began with basic information about the pipeline. Then I defined the pipeline steps. I have three steps:

```yaml
kind: pipeline
name: default

steps:
    - name: test
    - name: docker-build
    - name: deploy-ssh
```

## 2. Run tests

Drone has a lot of plugins, so for the running tests, I will use a python plugin where I will install poetry(I use it for managing as dependencies manager), all the dependencies, and run `pytest``

To do this, I will add the following to `drone.yml`:

```yaml
- name: test
    image: python
    commands:
      - python -m pip install poetry
      - python -m poetry install
      - python -m poetry run pytest
```

Pretty simple. Huh?)

## 3. Build docker image and push it to Nexus

In this step, I'm using standard plugin `plugins/docker`. Also, for this step I will need to authorize in my Nexus registry. So I will create secrets in Drone web-interface, which will be available only for this project.

```yaml
- name: docker-build
    image: plugins/docker
    settings:
      insecure: true
      repo: 100.90.71.128:8082/nax/days_count
      registry: 100.90.71.128:8082
      username:
        from_secret: nexus_username
      password:
        from_secret: nexus_password
```

## 4. Deploy image to server and run it

If you deploy to a production server, you probably will use Kubernetes or something like this. But for I need to deploy to my home server with only docker installed. I decided to try and use `plugins/ssh` plugin. There are several tutorials in the Internet how to use it. But I had some problems with it. I had to collect by pieces all the information from different sources. So here is my solution.

First of all I put Nexus credentials to secrets in Drone web-interface. I also added to secrets information needed to start Docker image with all the settings. It will pass inside as ENV variables.

Also we need to execute some shell commands. I will 

So basic configuration looks this way:

```yaml
- name: deploy-ssh
    image: appleboy/drone-ssh
    environment:
      NEXUS_USER:
        from_secret: nexus_username
      NEXUS_PASSWORD:
        from_secret: nexus_password
      ENV_VAR:
        from_secret: env_file
    settings:
      host:
        - 100.90.71.128
      username: nax
      password:
        from_secret: ssh_password
      port: 22
      command_timeout: 2m
      envs: [ NEXUS_USER, NEXUS_PASSWORD, ENV_VAR ]
      script:
        - export PATH="/usr/local/bin:$PATH"
        - touch env.config
        - echo $ENV_VAR >> env.config
        - docker login -u $NEXUS_USER -p $NEXUS_PASSWORD 100.90.71.128:8082
        - docker pull 100.90.71.128:8082/nax/days_count:latest
        - docker image prune -f
        - docker stop days_count
        - docker container prune -f
        - docker run -d --env-file env.config -v /etc/localtime:/etc/localtime:ro --restart always --name days_count 100.90.71.128:8082/nax/days_count:latest
        - rm env.config
```