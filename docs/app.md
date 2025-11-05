# Отчет о проделанной работе за неделю 1

## APP

**Период:** Неделя 1  
**Технологии:** Java Spring Boot  

### Выполненные задачи

#### 1. Настройка проекта Spring Boot
- Инициализирован новый Spring Boot проект
- Настроена базовая конфигурация приложения
- Добавлены необходимые зависимости в `pom.xml`

#### 2. Реализация REST API endpoints

##### 📍 GET /api/hello
**Назначение:** Базовая проверка работоспособности приложения

##### 📍 POST /api/hi  
**Назначение:** Обработка POST запросов с данными

##### 📍 GET /healthz
**Назначение:** Health-check endpoint для мониторинга

## Docker Hub
- Создан аккаунт [Docker Hub](https://hub.docker.com) gitopsitmo
- Сделаны для всех участников команды Personal Access Token (PAT)

### 2. Аутентификация в аккаунт с устройства

```shell
docker login -u gitopsitmo
```

И вводим PAT

### 3. Сборка и публикация образа

```shell
docker build -t gitopsitmo/gitops-app:0.1.0 app
docker push gitopsitmo/gitops-app:0.1.0
```

Для новых версий - очевидно нужен другой tag

### 4. Образ сделан приватным в Docker Hub

### 5. Подтягивание образа из Docker Hub

```shell
docker pull gitopsitmo/gitops-app:0.1.0
```



