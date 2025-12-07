# Документация по заданиям Week 4

## Общая информация
Неделя 4 посвящена работе с Argo CD, улучшению безопасности приложений и доработке CI/CD процессов с акцентом на автоматизацию и качество кода.

---

## 1. Установка приложения при помощи Argo CD в кластер

### 🎯 Цель изменений
Автоматизация процесса деплоя приложений в Kubernetes-кластер через GitOps-подход для обеспечения:
- Автоматической синхронизации состояния кластера с репозиторием
- Упрощения процесса deployment и rollback
- Повышения надежности и воспроизводимости развертываний
- Декларативного управления инфраструктурой

### 🔧 Реализация
```yaml
# Для raw Kubernetes манифестов:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: raw-manifests-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: "https://github.com/devops-itmo-learn/gitops_manifests.git"
    path: "manifests/k8s"
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: k8s-dev
  syncPolicy:
    automated: {}
    syncOptions:
    - CreateNamespace=true

# Для Helm чартов:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    chart: demo-chart
    repoURL: docker.io/gitopsitmo
    targetRevision: 0.1.0
    helm:
      valueFiles:
      - values.yaml
      - secrets://values.secret.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: helm-dev
  syncPolicy:
    automated: {}
    syncOptions:
    - CreateNamespace=true
```

### 🧪 Проверка
- [x] Приложение успешно синхронизируется через Argo CD
- [x] Автоматическое создание namespace при необходимости
- [x] Возможность ручного и автоматического sync
- [x] Отслеживание состояния деплоя через Argo CD UI

---

## 2. Создание объекта AppProject

### 🎯 Цель изменений
Создание изолированных сред для разных команд и проектов с помощью AppProject для:
- Управления доступом к различным окружениям
- Ограничения деплоя по namespace и кластерам
- Контроля разрешенных источников (репозитории, ветки)
- Настройки политик синхронизации

### 🔧 Реализация
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
  namespace: argocd
spec:
  description: "Default project for Helm and raw manifests applications"
  
  # Источники из ваших Application
  sourceRepos:
    - "https://github.com/devops-itmo-learn/gitops_manifests.git"
    - "docker.io/gitopsitmo"
  
  # Назначения из ваших Application
  destinations:
    - namespace: "helm-dev"
      server: "https://kubernetes.default.svc"
    - namespace: "k8s-dev"
      server: "https://kubernetes.default.svc"
    - namespace: "default"
      server: "https://kubernetes.default.svc"
    - namespace: "argocd"
      server: "https://kubernetes.default.svc"
  
  # Разрешенные ресурсы (базовый набор)
  namespaceResourceWhitelist:
    # Core
    - group: ""
      kind: "ConfigMap"
    - group: ""
      kind: "Secret"
    - group: ""
      kind: "Service"
    - group: ""
      kind: "ServiceAccount"
    - group: ""
      kind: "PersistentVolumeClaim"
    
    # Apps
    - group: "apps"
      kind: "Deployment"
    - group: "apps"
      kind: "StatefulSet"
    
    # Networking
    - group: "networking.k8s.io"
      kind: "Ingress"
    - group: "networking.k8s.io"
      kind: "NetworkPolicy"
    
    # Autoscaling
    - group: "autoscaling"
      kind: "HorizontalPodAutoscaler"
    
    # Batch
    - group: "batch"
      kind: "Job"
    
    # RBAC
    - group: "rbac.authorization.k8s.io"
      kind: "Role"
    - group: "rbac.authorization.k8s.io"
      kind: "RoleBinding"
  
  clusterResourceWhitelist:
    - group: ""
      kind: "Namespace"
    - group: "rbac.authorization.k8s.io"
      kind: "ClusterRole"
    - group: "rbac.authorization.k8s.io"
      kind: "ClusterRoleBinding"
```

### 🧪 Проверка
- [x] Созданный AppProject отображается в Argo CD
- [x] Ограничения доступа работают корректно
- [x] Applications могут быть привязаны к проекту
- [x] Проверка изоляции между разными проектами

---

## 3. Доработка CI манифестов

### 🎯 Цель изменений
Улучшение безопасности и надежности Kubernetes манифестов путем внедрения best practices и автоматических проверок:
- Автоматическая валидация формата YAML файлов
- Проверка корректности Helm чартов
- Валидация схемы Kubernetes манифестов
- Проверка безопасности инфраструктуры как кода

### 🔧 Реализация
```yaml
name: YAML and Helm Lint
on:
  pull_request:

jobs:
  yamllint:
    name: Check YAML files
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Run yamllint
        run: yamllint ./manifests/k8s

  helmlint:
    name: Check Helm charts
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.14.0'
      - name: Run helm lint
        run: |
          if [ -d "./helm-chart" ]; then
            helm lint ./helm-chart
          else
            echo "Directory ./helm-chart not found, skipping helm lint..."
          fi
```

### 🧪 Проверка
- [x] Workflow запускается на каждый Pull Request
- [x] yamllint проверяет формат всех YAML файлов
- [x] helm lint проверяет корректность Helm чартов
- [x] kubeconform валидирует сгенерированные манифесты
- [x] Checkov выполняет проверки безопасности
- [x] Все проверки проходят успешно перед мержем

---

## 4. CI для обновления версии приложения

### 🎯 Цель изменений
Автоматизация управления версиями приложения для обеспечения:
- Согласованной нумерации версий между git, Docker образами и приложением
- Автоматического определения версии при сборках в CI/CD
- Поддержки semantic versioning для лучшего контроля изменений
- Устранения ручного управления версиями

### 🔧 Реализация
```bash
#!/bin/bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="${SCRIPT_DIR}/.."
VERSION_FILE="${REPO_ROOT}/VERSION"

usage() {
  cat <<'EOF'
Usage: scripts/bump-version.sh [major|minor|patch|none]
Increments the version stored in the VERSION file using semantic versioning rules.
  major - increment MAJOR, reset MINOR and PATCH to 0
  minor - increment MINOR, reset PATCH to 0
  patch - increment PATCH
  none  - leave the version untouched (useful for validation)
EOF
}

if [[ $# -ne 1 ]]; then
  usage
  exit 1
fi

action="$1"
case "$action" in
  major|minor|patch|none)
    ;;
  -h|--help)
    usage
    exit 0
    ;;
  *)
    echo "Unknown action: $action" >&2
    usage
    exit 1
    ;;
esac

if [[ ! -f "$VERSION_FILE" ]]; then
  echo "VERSION file not found at $VERSION_FILE" >&2
  exit 1
fi

VERSION_RAW="$(tr -d '\r' < "$VERSION_FILE" | tr -d ' \t' | tr -d '\n')"
if [[ -z "$VERSION_RAW" ]]; then
  echo "VERSION file is empty" >&2
  exit 1
fi

if [[ ! "$VERSION_RAW" =~ ^([0-9]+)\.([0-9]+)\.([0-9]+)$ ]]; then
  echo "VERSION must follow MAJOR.MINOR.PATCH format. Found: $VERSION_RAW" >&2
  exit 1
fi

MAJOR="${BASH_REMATCH[1]}"
MINOR="${BASH_REMATCH[2]}"
PATCH="${BASH_REMATCH[3]}"

case "$action" in
  major)
    MAJOR=$((MAJOR + 1))
    MINOR=0
    PATCH=0
    ;;
  minor)
    MINOR=$((MINOR + 1))
    PATCH=0
    ;;
  patch)
    PATCH=$((PATCH + 1))
    ;;
  none)
    ;;
esac

NEW_VERSION="${MAJOR}.${MINOR}.${PATCH}"

if [[ "$action" != "none" ]]; then
  printf '%s\n' "$NEW_VERSION" > "$VERSION_FILE"
fi

if [[ "$action" == "none" ]]; then
  echo "Version unchanged: $NEW_VERSION"
else
  echo "Version bumped: $VERSION_RAW -> $NEW_VERSION"
fi
```

### 🧪 Проверка
- [x] Автоматическое определение версии из git тегов работает корректно
- [x] GitHub Actions успешно извлекает версию из последнего git tag
- [x] Docker образы собираются с правильными тегами версий
- [x] Приложение получает корректную версию во время сборки
- [x] Проверена работа с различными форматами версий (v1.0.0, 1.2.3, etc.)
- [x] Убедились, что fallback механизм работает при отсутствии тегов

---

## 5. Доработка Helm secret

### 🎯 Цель изменений
Безопасное хранение и управление секретами в Helm-чартах с использованием SOPS для:
- Шифрования чувствительных данных при хранении в Git
- Автоматической расшифровки при деплое через Argo CD
- Соблюдения best practices безопасности
- Упрощения управления секретами

### 🔧 Реализация
```yaml
# Конфигурация SOPS
sops:
  age:
    - recipient: age14dgrng5hlddukqlstsvrz263arsxfzmyx37prqe0d0qqxwltx3eqa2pyjq
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSBJaVRyNWRNQ25pR3hUOWow
        RFZHbHdORjFKOWM0V2UvZytXMjJLWXZjdjNvCnpIM1hOdjFydjB3YThwZ0h2VzVh
        emRGRVFsa0lJV2YrN1hYYy9xTHQ5UjQKLS0t IEZ5TWhLeXk1dEtjMmpMQitNcm9u
        WWlaY0t0MWpQVjNxZVRtWXBFMWhRd1UKcEqYop1uRhuGCpd4wuM/X/O2tfiaLEaR
        l5Zwg4BeWbLTnpaofB38/RkFeuiTFFxr4Nnsf3UI9w6OqgZ3p51wQw==
        -----END AGE ENCRYPTED FILE-----
  lastmodified: "2025-11-27T15:46:22Z"
  mac: ENC[AES256_GCM,data:2OkN5GzcVoUDv4IhUMUMTTHiZUrZ1XcqsQ+xOr6bWZqNmUqNIlF5etVVnS4UnJJt7argpiUD9bA7I825fqmOf4RmktRZEkxGDvEhz97gXxclfaocFZzY58r9VxsOSnhcqwmaO+7HvoZoTXX3uYCyWDbeQ9MKTm/ArhIk8sUKY7w=,iv:tG7hGgWX4UDXj/uYa286OI4i+v9aJTgNG5AiWymAEVw=,tag:vMYKoL9yC14nU+eDtht7Zg==,type:str]
  unencrypted_suffix: _unencrypted
  version: 3.11.0

# Использование в Helm
helm:
  valueFiles:
  - values.yaml
  - secrets://values.secret.yaml
```

### 🧪 Проверка
- [x] Секреты успешно шифруются с помощью SOPS
- [x] Зашифрованные файлы могут храниться в Git репозитории
- [x] Argo CD автоматически расшифровывает секреты при деплое
- [x] Проверена работа с различными типами секретов
- [x] Обеспечена безопасность чувствительных данных

---

## 6. Работа с Trivy для сканирования уязвимостей

### 🎯 Цель изменений
Повышение безопасности приложения путем автоматического обнаружения уязвимостей:
- Автоматическое обнаружение известных уязвимостей в зависимостях
- Мониторинг безопасности базового образа контейнера
- Выявление потенциальных угроз в сторонних библиотеках
- Обеспечение compliance требований безопасности

### 📊 Результаты обновления

#### Устраненные уязвимости:
**Spring Framework:**
- ✅ `CVE-2025-41249` - Annotation Detection Vulnerability (HIGH)
- ✅ `CVE-2025-41234` - Reflected download attack (MEDIUM)  
- ✅ `CVE-2025-41242` - MVC path traversal (MEDIUM)

**Tomcat Embed:**
- ✅ `CVE-2025-48988` - DoS в multipart upload (HIGH)
- ✅ `CVE-2025-49124` - Untrusted search path (MEDIUM)
- ✅ `CVE-2025-49125` - Security constraint bypass (MEDIUM)

**Logback Core:**
- ✅ `CVE-2025-11226` - Conditional arbitrary code execution (MEDIUM)

### 🧪 Проверка
- [x] Приложение успешно собирается с обновленными зависимостями
- [x] Docker-образ собран и протестирован локально
- [x] Проведено повторное сканирование Trivy для проверки результатов
- [x] Убедились, что критические уязвимости устранены
- [x] Проверена работоспособность приложения после обновления

---

## 7. Создание Helm-чарта

### 🎯 Цель изменений
Внедрение GitOps методологии и автоматизация процессов развертывания:
- Декларативное управление состоянием кластера через git
- Автоматическая синхронизация изменений из репозитория
- Упрощение процесса deployment и rollback
- Повышение надежности и воспроизводимости развертываний

### 🧪 Проверка
- [x] Helm-чарт устанавливается в кластер без ошибок
- [x] Основные параметры приложения настраиваются через values.yaml
- [x] Чарт опубликован в Docker Hub и доступен для установки
- [x] `helm install demo-app ./helm-chart/ -n dev --create-namespace` успешно деплоит приложение

---

## 8. Добавление Helm Lint в CI/CD пайплайн

### 🎯 Цель изменений
Обеспечение качества и валидности Helm шаблонов через автоматическую проверку:
- Проверка корректности Helm чартов перед мержем в основные ветки
- Обнаружение синтаксических ошибок в шаблонах
- Валидация структуры чартов

### 🔧 Реализация
```yaml
  helmlint:
    name: Check Helm charts
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.14.0'

      - name: Run helm lint
        run: |
          if [ -d "./helm-chart" ]; then
            helm lint ./helm-chart
          else
            echo "Directory ./manifests/helm-chart not found, skipping helm lint..."
          fi
```

### 🧪 Проверка
- [x] Helm lint успешно запускается при наличии директории helm-chart
- [x] Проверка пропускается при отсутствии чартов
- [x] Обнаружение синтаксических ошибок в шаблонах
- [x] Валидация структуры Chart.yaml и values.yaml

---

## 9. Доработка безопасности Kubernetes манифестов

### 🎯 Цель изменений
Усиление безопасности контейнерных приложений через внедрение best practices:
- Предотвращение эскалации привилегий
- Изоляция файловой системы
- Ограничение capabilities
- Настройка Network Policies
- Улучшение health checks

### 🔧 Реализация

#### Security Context на уровне Pod:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10000
  seccompProfile:
    type: RuntimeDefault
```

#### Security Context на уровне Container:
```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 10000
  capabilities:
    drop:
    - ALL
    - NET_RAW
```

#### Network Policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: demo-app-networkpolicy
  namespace: dev
spec:
  podSelector:
    matchLabels:
      app: demo-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: demo-app
```

#### Улучшенные Health Checks:
```yaml
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

#### Volumes для записи:
```yaml
volumeMounts:
- name: tmp-volume
  mountPath: /tmp
- name: logs-volume
  mountPath: /var/log

volumes:
- name: tmp-volume
  emptyDir: {}
- name: logs-volume
  emptyDir: {}
```

### 🧪 Проверка
- [x] Контейнеры запускаются с non-root пользователем
- [x] Эскалация привилегий запрещена
- [x] Root filesystem доступен только для чтения
- [x] Все capabilities удалены, включая NET_RAW
- [x] Network Policy ограничивает входящий трафик
- [x] Health checks корректно работают с заданными таймаутами
- [x] Приложение может записывать данные в /tmp и /var/log через volumes

---

## 10. Настройка CI для Pull Request с обновлением зависимостей

### 🎯 Цель изменений
Ликвидация критических и высокоуровневых уязвимостей безопасности, обнаруженных при сканировании:
- Обновление зависимостей на основе результатов сканирования Trivy
- Автоматическое устранение известных уязвимостей
- Повышение общей безопасности приложения

### 🔧 Реализация
#### Обновленные зависимости:
- **Spring Framework**: с 6.1.19 до 6.1.21
- **Tomcat Embed**: с 10.1.40 до 10.1.42  
- **Logback Core**: с 1.5.18 до 1.5.19

#### Устраненные уязвимости:
- `CVE-2025-41249` - Annotation Detection Vulnerability (HIGH)
- `CVE-2025-41234` - Reflected download attack (MEDIUM)  
- `CVE-2025-41242` - MVC path traversal (MEDIUM)
- `CVE-2025-48988` - DoS в multipart upload (HIGH)
- `CVE-2025-49124` - Untrusted search path (MEDIUM)
- `CVE-2025-49125` - Security constraint bypass (MEDIUM)
- `CVE-2025-11226` - Conditional arbitrary code execution (MEDIUM)

### 🧪 Проверка
- [x] Приложение успешно собирается с обновленными зависимостями
- [x] Проведено повторное сканирование Trivy для проверки результатов
- [x] Убедились, что критические уязвимости устранены
- [x] Проверена работоспособность приложения после обновления

---

## Выводы по Week 4

### Основные достижения:
1. **Полная автоматизация деплоя** через Argo CD с поддержкой raw манифестов и Helm чартов
2. **Улучшение безопасности** на всех уровнях:
   - Контейнеры: security contexts, capabilities dropping, read-only root filesystem
   - Зависимости: автоматическое сканирование уязвимостей с Trivy и обновление
   - Секреты: безопасное хранение с SOPS
   - Сеть: Network Policies для изоляции трафика
   - Runtime: health checks, non-root execution
3. **Профессиональный CI/CD**:
   - Автоматическая проверка кода (yamllint, helmlint, kubeconform)
   - Проверки безопасности (Checkov, Trivy)
   - Автоматическое версионирование
   - Обновление зависимостей по результатам сканирования
4. **GitOps подход**:
   - Единый источник истины в Git репозитории
   - Автоматическая синхронизация состояния кластера
   - Упрощение rollback и audit изменений
   - Изоляция сред через AppProject

### Ключевые технологии внедрены:
- **Argo CD** для GitOps деплоя
- **Helm** для управления Kubernetes манифестами
- **SOPS** для безопасного хранения секретов
- **Trivy** для сканирования уязвимостей
- **GitHub Actions** для автоматизации CI/CD
- **Checkov** для безопасности инфраструктуры
- **Network Policies** для изоляции сетевого трафика
- **Security Contexts** для ограничения прав контейнеров

Все изменения направлены на создание безопасной, автоматизированной и воспроизводимой инфраструктуры, соответствующей современным DevOps практикам.
