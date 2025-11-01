# Настройка аккаунта Docker Hub и публикация образа

## 1. Создание аккаунта Docker Hub
- Создан аккаунт [Docker Hub](https://hub.docker.com) gitopsitmo
- Сделаны для всех участников команды Personal Access Token (PAT)

## 2. Аутентификация в аккаунт с устройства

```shell
docker login -u gitopsitmo
```

И вводим PAT

## 3. Сборка и публикация образа

```shell
docker build -t gitopsitmo/gitops-app:0.1.0 app
docker push gitopsitmo/gitops-app:0.1.0
```

Для новых версий - очевидно нужен другой tag

## 4. Образ сделан приватным в Docker Hub

## 5. Подтягивание образа из Docker Hub

```shell
docker pull gitopsitmo/gitops-app:0.1.0
```