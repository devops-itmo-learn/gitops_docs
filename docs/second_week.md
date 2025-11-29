# Отчет о проделанной работе за неделю 2

## Обзор

**Период:** Неделя 2  
**Технологии:** Kubernetes, Docker, GitOps, Spring Boot  
**Команда:** Коровкин Артём Александрович, Котков Дмитрий Александрович, Баранов Денис Сергеевич, Шайхиев Эльдар Ильхамович

## Выполненные задачи

### 1. Docker Image Оптимизация

#### Многоступенчатая сборка
```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

# Копирование Gradle файлов и настройка разрешений
COPY gradle/ gradle/
COPY gradlew gradlew.bat build.gradle.kts settings.gradle.kts gradle.properties ./
RUN chmod +x ./gradlew

# Загрузка зависимостей
RUN ./gradlew dependencies --no-daemon

# Копирование исходного кода и сборка
COPY src/ src/
RUN ./gradlew bootJar --no-daemon

# Runtime stage
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Установка curl и создание пользователя
RUN apk add --no-cache curl && \
    addgroup -S appgroup && adduser -S appuser -G appgroup

# Копирование JAR файла из build stage
COPY --from=build /app/build/libs/*.jar app.jar

# Настройка прав доступа
RUN chown -R appuser:appgroup /app
USER appuser

# Экспорт порта и healthcheck
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -fsS http://localhost:8080/healthz || exit 1

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

#### Ключевые улучшения:
- ✅ **Многоступенчатая сборка** - уменьшение размера финального образа
- ✅ **Безопасность** - запуск от non-root пользователя
- ✅ **Health checks** - мониторинг состояния приложения
- ✅ **Кэширование зависимостей** - ускорение сборки
- ✅ **Минимальный runtime** - использование JRE вместо JDK

### 2. Kubernetes Manifests

#### Namespace (namespace-dev.yaml)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

#### ConfigMap (configmap.yaml)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
  labels:
    app: demo-app
    component: config
data:
  SERVER_PORT: "8080"
  SPRING_APPLICATION_NAME: "demo-app"
  LOGGING_LEVEL_ROOT: "INFO"
  LOGGING_LEVEL_SPRING_WEB: "DEBUG"
```

#### Secret (secret.yaml)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: docker-registry-secret
  namespace: dev
  labels:
    app: demo-app
    component: registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <encoded-docker-config>
```

#### Deployment (deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: dev
  labels:
    app: demo-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      imagePullSecrets:
      - name: docker-registry-secret
      nodeSelector:
        zone: dev
      containers:
      - name: demo-app
        image: gitopsitmo/gitops-app:0.1.0
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        envFrom:
        - configMapRef:
            name: app-config
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

#### Service (service.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: dev
  labels:
    app: demo-app
spec:
  type: NodePort
  selector:
    app: demo-app
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    protocol: TCP
    nodePort: 30080
```

### 3. Структура репозитория GitOps

```
gitops_manifests/
└── manifests/
    └── k8s/
        ├── configmap.yaml          # Конфигурация приложения
        ├── deployment.yaml         # Деплоймент с 3 репликами
        ├── namespace-dev.yaml      # Namespace для dev окружения
        ├── secret.yaml            # Docker registry credentials
        └── service.yaml           # Service с NodePort 30080
```

### 4. Конфигурация приложения

#### Health Checks:
- **Liveness Probe:** Проверка каждые 10 секунд после 30-секундной задержки
- **Readiness Probe:** Проверка каждые 5 секунд после 10-секундной задержки
- **Endpoint:** `/healthz` на порту 8080

#### Ресурсы:
- **Requests:** 100m CPU, 256Mi Memory
- **Limits:** 500m CPU, 512Mi Memory

#### Сетевые настройки:
- **Container Port:** 8080
- **Service Port:** 8080
- **NodePort:** 30080

## Результаты

### ✅ Завершенные задачи недели 2:
- [x] Создание Docker image
- [x] Публикация Image в DockerHub 
- [x] Создание namespace
- [x] Создание ConfigMap
- [x] Создание secret
- [x] Создание Deployment
- [x] Создание Service

### 🚀 Достижения:
1. **Оптимизированный Docker образ** с лучшими практиками безопасности
2. **Полный набор Kubernetes манифестов** для развертывания приложения
3. **Настроенные health checks** для надежной работы в production
4. **Правильная конфигурация ресурсов** для стабильной работы
5. **Готовое GitOps окружение** для автоматизированного деплоя

### 🔧 Технические особенности:
- Использование `nodeSelector` для контроля размещения pod'ов
- Настройка `imagePullSecrets` для доступа к приватному registry
- Конфигурация через ConfigMap для гибкости настроек
- Многоуровневое логирование через environment variables

Все манифесты готовы к применению в Kubernetes кластере и соответствуют best practices для production-ready развертывания.
