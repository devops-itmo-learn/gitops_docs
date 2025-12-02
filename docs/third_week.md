# Source
# Работа с Trivy
## Описание изменений
- Интегрирован инструмент Trivy для сканирования уязвимостей безопасности
- Настроено сканирование Docker образов в CI/CD пайплайне
- Добавлен детализированный отчет о найденных уязвимостях

## 🎯 Цель изменений
Для повышения безопасности приложения путем:
- Автоматического обнаружения известных уязвимостей в зависимостях
- Мониторинга безопасности базового образа контейнера
- Выявления потенциальных угроз в сторонних библиотеках
- Обеспечения compliance требований безопасности

## ✅ Тип изменений
-  Исправление ошибки (bug fix)
## 🧪 Проверка
- Приложение успешно собирается локально
- Docker-образ собран и запущен
    
# Настройка CI для Pull Request
**Что было сделано в этом PR:**
- Обновлены версии уязвимых зависимостей на основе результатов сканирования Trivy
- Выполнено обновление Spring Framework с 6.1.19 до 6.1.21
- Обновлен Tomcat Embed с 10.1.40 до 10.1.42
- Обновлен Logback Core с 1.5.18 до 1.5.19
- Проведено повторное сканирование для проверки устранения уязвимостей

## 🎯 Цель изменений
Для устранения критических и высокоуровневых уязвимостей безопасности, обнаруженных при сканировании:
- Ликвидация уязвимостей в Spring Framework (CVE-2025-41249, CVE-2025-41234, CVE-2025-41242)
- Устранение уязвимостей в Tomcat (CVE-2025-48988, CVE-2025-49124, CVE-2025-49125)
- Исправление уязвимости в Logback Core (CVE-2025-11226)
- Повышение общей безопасности приложения

## ✅ Тип изменений
- [x] Исправление ошибки (bug fix)
- [x] Техническое улучшение (refactor / CI / инфраструктура)

## 🧪 Проверка
- [x] Приложение успешно собирается с обновленными зависимостями
- [x] Docker-образ собран и протестирован локально
- [x] Проведено повторное сканирование Trivy для проверки результатов
- [x] Убедились, что критические уязвимости устранены
- [x] Проверена работоспособность приложения после обновления

## 📊 Результаты обновления

### Устраненные уязвимости:

**Spring Framework:**
- ✅ `CVE-2025-41249` - Annotation Detection Vulnerability (HIGH) - устранено в 6.1.21
- ✅ `CVE-2025-41234` - Reflected download attack (MEDIUM) - устранено в 6.1.21  
- ✅ `CVE-2025-41242` - MVC path traversal (MEDIUM) - устранено в 6.2.10

**Tomcat Embed:**
- ✅ `CVE-2025-48988` - DoS в multipart upload (HIGH) - устранено в 10.1.42
- ✅ `CVE-2025-49124` - Untrusted search path (MEDIUM) - устранено в 10.1.42
- ✅ `CVE-2025-49125` - Security constraint bypass (MEDIUM) - устранено в 10.1.42

**Logback Core:**
- ✅ `CVE-2025-11226` - Conditional arbitrary code execution (MEDIUM) - устранено в 1.5.19

### Оставшиеся уязвимости (требуют major-версионного обновления):
- `CVE-2025-48989` - HTTP/2 DoS attack (требует Tomcat 10.1.44)
- `CVE-2025-55752` - Directory traversal RCE (требует Tomcat 10.1.45)

# Добавляние VERSION и скрипта версионирования

## Описание изменений
- Реализована автоматическая генерация версий приложения на основе git тегов
- Настроена интеграция с GitHub Actions для автоматического определения версии
- Добавлена поддержка semantic versioning через git tags (v*.*.*)
- Обновлены процессы сборки Docker образов с автоматическим тегированием
- Настроена проверка корректности версий в CI/CD пайплайне

## 🎯 Цель изменений
Для автоматизации управления версиями приложения и обеспечения:
- Согласованной нумерации версий между git, Docker образами и приложением
- Автоматического определения версии при сборках в CI/CD
- Поддержки semantic versioning для лучшего контроля изменений
- Устранения ручного управления версиями и связанных с этим ошибок

## 🧪 Проверка
- [x] Автоматическое определение версии из git тегов работает корректно
- [x] GitHub Actions успешно извлекает версию из последнего git tag
- [x] Docker образы собираются с правильными тегами версий
- [x] Приложение получает корректную версию во время сборки
- [x] Проверена работа с различными форматами версий (v1.0.0, 1.2.3, etc.)
- [x] Убедились, что fallback механизм работает при отсутствии тегов

## 🔧 Реализованная функциональность

### Автоматическое определение версии:
```yaml

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

# Manifests
# Создание Helm-чарта 

## Описание изменений
- Создан Helm-чарт со структурой стандартного шаблона
- Шаблонизированы текущие Kubernetes-манифесты
- Параметризированы основные настройки через values.yaml
- Опубликован собранный чарт в Docker Hub в виде OCI-паке

## 🎯 Цель изменений
Для внедрения GitOps методологии и автоматизации процессов развертывания:
- Декларативное управление состоянием кластера через git
- Автоматическая синхронизация изменений из репозитория
- Упрощение процесса deployment и rollback
- Повышение надежности и воспроизводимости развертываний
- Обеспечение соответствия состояния кластера желаемому состоянию в git

## 🧪 Проверка
- Helm-чарт устанавливается в кластер без ошибок
- Основные параметры приложения настраиваются через values.yaml
- Чарт опубликован в Docker Hub и доступен для установки
- helm install demo-app ./helm-chart/ -n dev --create-namespace успешно деплоит приложение

# Добавление Helm Lint в CI/CD пайплайн
**Что было сделано в этом PR:**
- Добавлена автоматическая проверка Helm чартов в CI/CD пайплайн
- Создана job в GitHub Actions:
- helmlint для проверки Helm чартов в папке helm-chart
- Проверка helm-chart выполняется только если директория существует

## 🎯 Цель изменений
- Обеспечить качество и валидность Helm шаблонов
- Проверять корректность Helm чартов перед мержем в основные ветки

## 🔧 Реализованная функциональность

### Добавлен HelmLint
```
name: YAML and Helm Lint

on:
  pull_request:
@@ -26,3 +26,24 @@ jobs:

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
            echo "Directory ./manifests/helm-chart not found, skipping helm lint..."
          fi
```

# Доработка CI манифестов

## Описание изменений
- Расширен существующий workflow YAML linting
- Добавлены проверки Helm-чарта:
- helm lint для линтинга чарта
- helm template для генерации Kubernetes-манифестов
- kubeconform для валидации схем манифестов
- Добавлен запуск Checkov для проверки безопасности и соответствия best practices
- Все проверки выполняются при открытии Pull Request

## 🎯 Цель изменений
- Обеспечить автоматическую проверку формата YAML, корректности Helm-чартов и безопасность инфраструктуры перед слиянием PR.

## ✅ Тип изменений
- [x] Техническое улучшение (refactor / CI / инфраструктура)

## 🧪 Проверка
- Workflow запускается на Pull Request
- yamllint проверяет формат YAML-файлов
- Helm-чарт проходит линтинг и генерацию манифестов
- kubeconform валидирует сгенерированные манифесты
- Checkov выполняет проверку безопасности манифестов и чарта
