# Правила работы с репозиториями

## Репозитории
- [**gitops_source**](https://github.com/devops-itmo-learn/gitops_source) - исходный код приложения, Dockerfile, CI/CD.
- [**gitops_docs**](https://github.com/devops-itmo-learn/gitops_docs) - документация, шаблоны, DevSecOps-инструкции.

## Задачи
Задачи выбираются [здесь](https://github.com/orgs/devops-itmo-learn/projects/1/views/1). При выборе задачи, не забывайте нажать **Assign yourself** и изменить статус задачи на **In progress**. Для каждой задачи создаётся ветка.

## Ветки
- Основная ветка: `main` (защищена, без прямых пушей).
- Рабочие ветки создаются от `main`:
  - `feature/<task>` — новые фичи (feature/add-healthcheck)
  - `fix/<task>` — исправления (fix/dockerfile-path)
  - `docs/<task>` — документация (docs/update-readme)

`<task>` должен кратко отражать над чем вы работаете в этой ветке.

## Pull requests
После завершения работы над задачей создаётся **pull request** в `main` ветку.

- Создаётся для каждой ветки перед merge в `main`
- Название: `feat: ...`, `fix: ...`, `docs: ...`
- В описании: краткое изменение + ссылка на Issue
- Merge только после:
  - прохождения CI (в будущем)
  - одного approve

Шаблон **pull request** можно найти [здесь](.github/PULL_REQUEST_TEMPLATE.md).