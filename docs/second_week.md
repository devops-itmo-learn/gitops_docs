# Отчет о проделанной работе за неделю 2

## Обзор

**Период:** Неделя 2  
**Технологии:** Kubernetes, Docker, GitOps, Spring Boot

## Выполненные задачи

### 1. Docker Image Оптимизация

#### Реализована многоступенчатая сборка
Использована двухэтапная сборка Docker образа с разделением на build и runtime стадии:

**Build стадия:**
- Базируется на `eclipse-temurin:21-jdk-alpine`
- Устанавливаются зависимости Gradle с кэшированием
- Выполняется сборка Spring Boot приложения
- Создается JAR файл приложения

**Runtime стадия:**
- Базируется на облегченном `eclipse-temurin:21-jre-alpine`
- Создана безопасная среда с non-root пользователем `appuser`
- Добавлен curl для healthcheck проверок
- Настроены healthcheck для мониторинга состояния приложения
- Используется только JRE, что уменьшает размер финального образа

**Ключевые улучшения:**
- ✅ **Многоступенчатая сборка** - уменьшение размера финального образа
- ✅ **Безопасность** - запуск от non-root пользователя
- ✅ **Health checks** - мониторинг состояния контейнера
- ✅ **Кэширование зависимостей** - ускорение процесса сборки

### 2. Kubernetes Manifests

#### Добавлены следующие манифесты:

**1. Namespace (`namespace-dev.yaml`)**  
Создание изолированного пространства `dev` для развертывания приложения.

**2. ConfigMap (`configmap.yaml`)**  
Хранение конфигурационных параметров приложения:
- Порт сервера (8080)
- Название приложения (demo-app)
- Уровни логирования (INFO, DEBUG)

**3. Secret (`secret.yaml`)**  
Хранение чувствительных данных - учетных данных для доступа к приватному Docker registry.

**4. Deployment (`deployment.yaml`)**  
Развертывание приложения с 3 репликами и настройками:
- Использование `imagePullSecrets` для авторизации в registry
- Node selector для размещения подов на нодах с меткой `zone=dev`
- Лимиты ресурсов (CPU/Memory)
- Liveness и readiness probes для health checks
- Environment variables из ConfigMap

**5. Service (`service.yaml`)**  
Предоставление доступа к приложению через NodePort (30080).

**6. Kind Cluster Configuration (`kind-config.yaml`)**  
Конфигурация локального Kubernetes кластера с:
- Control-plane нодой
- Двумя worker нодами с метками `zone=prod` и `zone=dev`
- Пробросом порта 30080 на localhost

## Инструкция по применению манифестов

### Подготовительные шаги

#### 1. Установка необходимых инструментов
```bash
# Установка Docker (если не установлен)
# Скачайте с https://www.docker.com/products/docker-desktop

# Установка Kind (Kubernetes in Docker)
brew install kind

# Установка kubectl
brew install kubectl
```

#### 2. Создание локального Kubernetes кластера
```bash
# Создание кластера Kind с конфигурацией
kind create cluster --config kind-config.yaml --name gitops-cluster

# Проверка создания кластера
kubectl cluster-info
kubectl get nodes
```

### Порядок применения манифестов

**Важно соблюдать порядок, так как некоторые ресурсы зависят от других:**

#### Шаг 1: Создание namespace
```bash
kubectl apply -f namespace-dev.yaml
```

#### Шаг 2: Создание Secret для доступа к Docker registry
```bash
kubectl apply -f secret.yaml
```

#### Шаг 3: Создание ConfigMap с конфигурацией
```bash
kubectl apply -f configmap.yaml
```

#### Шаг 4: Развертывание приложения
```bash
kubectl apply -f deployment.yaml
```

#### Шаг 5: Создание Service для доступа к приложению
```bash
kubectl apply -f service.yaml
```

### Проверка развертывания

#### Проверка состояния всех ресурсов
```bash
# Проверить все созданные ресурсы в namespace dev
kubectl get all -n dev

# Вывод должен содержать:
# - 3 пода (replicas: 3)
# - 1 deployment
# - 1 service
```

#### Мониторинг развертывания
```bash
# Отслеживать статус подов
kubectl get pods -n dev -w

# Проверить логи приложения
kubectl logs -n dev deployment/demo-app --tail=20

# Проверить подробную информацию о deployment
kubectl describe deployment demo-app -n dev
```

#### Проверка доступности приложения
```bash
# Проверить health endpoint
curl http://localhost:30080/healthz

# Проверить основной endpoint
curl http://localhost:30080/api/hello
```

#### Проверка конфигурации
```bash
# Проверить, что ConfigMap применен
kubectl describe configmap app-config -n dev

# Проверить, что переменные окружения установлены
kubectl exec -n dev deployment/demo-app -- env | grep SPRING
```

### Устранение неполадок

Если возникли проблемы:

#### 1. Проверить события в namespace
```bash
kubectl get events -n dev --sort-by='.lastTimestamp'
```

#### 2. Проверить состояние подов
```bash
# Посмотреть состояние каждого пода
kubectl describe pods -n dev -l app=demo-app

# Проверить логи конкретного пода
kubectl logs -n dev <pod-name>
```

#### 3. Проверить доступность образа
```bash
# Проверить, может ли кластер скачать образ
kubectl describe pod -n dev -l app=demo-app | grep -A 5 "Events"
```

### Удаление развертывания

Для очистки среды:

#### Удаление в правильном порядке
```bash
# 1. Удалить Service
kubectl delete -f service.yaml

# 2. Удалить Deployment
kubectl delete -f deployment.yaml

# 3. Удалить ConfigMap
kubectl delete -f configmap.yaml

# 4. Удалить Secret
kubectl delete -f secret.yaml

# 5. Удалить Namespace
kubectl delete -f namespace-dev.yaml

# Или все сразу
kubectl delete -f ./
```

#### Удаление кластера Kind
```bash
kind delete cluster --name gitops-cluster
```

## Структура репозитория GitOps

gitops_manifests/

├── manifests/ # Основная директория для Kubernetes манифестов

│ ├── namespaces/ # Пространства имен

│ ├── deployments/ # Деплойменты приложений

│ ├── services/ # Сервисы

│ ├── configs/ # ConfigMaps и Secrets

└── README.md # Этот файл
## Конфигурационные особенности

### Health Checks
- **Liveness Probe:** Проверка каждые 10 секунд после 30-секундной начальной задержки
- **Readiness Probe:** Проверка каждые 5 секунд после 10-секундной начальной задержки
- **Endpoint:** `/healthz` на порту 8080

### Управление ресурсами
- **Requests (гарантированные):** 100m CPU, 256Mi Memory
- **Limits (максимальные):** 500m CPU, 512Mi Memory

### Сетевая конфигурация
- **Container Port:** 8080
- **Service Port:** 8080
- **NodePort:** 30080
- **Доступ через:** http://localhost:30080

## Результаты

### ✅ Завершенные задачи недели 2:
- [x] Создание оптимизированного Docker образа
- [x] Публикация образа в DockerHub
- [x] Создание namespace для изоляции окружения
- [x] Создание ConfigMap для управления конфигурацией
- [x] Создание Secret для безопасного хранения учетных данных
- [x] Создание Deployment для развертывания приложения
- [x] Создание Service для предоставления доступа к приложению

### 🚀 Достижения:
1. **Создан полный набор Kubernetes манифестов** для развертывания приложения
2. **Реализована многоступенчатая сборка** Docker образа с оптимизацией размера
3. **Настроена локальная среда** для тестирования развертывания с помощью Kind
4. **Определен порядок применения** манифестов с учетом зависимостей между ресурсами
5. **Создана документация** с инструкциями по развертыванию и устранению неполадок

### 🔧 Ключевые особенности реализации:
- Использование `nodeSelector` для контроля размещения подов на определенных нодах
- Применение `imagePullSecrets` для безопасного доступа к приватному Docker registry
- Конфигурация приложения через ConfigMap для гибкости и переиспользования
- Настройка health checks для обеспечения отказоустойчивости
- Проброс портов в Kind кластере для локального тестирования

Данный набор манифестов позволяет развернуть Spring Boot приложение в локальном Kubernetes кластере с соблюдением базовых практик безопасности и управления конфигурацией.
