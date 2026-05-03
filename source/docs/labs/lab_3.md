# Лабораторная работа 3
## Тема: CI/CD для статического сайта в SourceCraft

---

## 1. Цель работы
Научиться настраивать автоматическое развертывание статического сайта на двух платформах:
- **SourceCraft** 
- **GitHub Pages** (через GitHub Actions)

С использованием единого локального репозитория и двух удалённых репозиториев (`origin` для GitHub, `sourcecraft` для SourceCraft).

---

## 2. Реализация

### 2.1 Структура проекта
```
Aqua223.github.io/
├── .github/workflows/ci.yaml    # GitHub Actions workflow
├── .sourcecraft/
│    ├── ci.yaml
│    └── sites.yaml             # SourceCraft workflow
├── source/
│   ├── mkdocs.yml              # Конфигурация MkDocs
│   ├── docs/                   # Исходные файлы документации
│   └── ...                     
├── requirements.txt            # Зависимости
└── README.md
```
### 2.2 GitHub Actions Workflow
```commandline
name: Publish docs via GitHub Pages

on:
  push:
    branches:
      - main
    paths:
      - 'source/docs/**'
      - 'source/mkdocs.yml'
      - 'requirements.txt'
      - '.github/workflows/ci.yaml'
  workflow_dispatch:

jobs:
  build:
    name: Deploy docs
    runs-on: ubuntu-latest
    steps:
      - name: Checkout main
        uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0

      - name: Python setup
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt

      - name: Build site
        run: |
          cd source
          mkdocs build -d ../docs

      - name: Deploy to release branch
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./docs 
          publish_branch: release
          force_orphan: true
          enable_jekyll: false
          user_name: 'github-actions[bot]'
          user_email: 'github-actions[bot]@users.noreply.github.com'
          commit_message: "feat: automatic site update"
```
### 2.3 Sourcecraft Actions Workflow
- .sourcecraft/ci.yaml
```commandline
on:
  push:
    workflows: build-site
    filter:
      branches: main

workflows:
  build-site:
    tasks:
      - name: build-and-publish-site
        cubes:
          - name: build-mkdocs-site
            image: docker.io/library/python:3.13-slim
            script:
              - python -m pip install --upgrade pip
              - "if [ -f requirements.txt ]; then pip install -r requirements.txt; fi"
              - cd source
              - mkdocs build -d ../docs
              - echo "Сайт собран в папке /docs"
              - ls -la ../docs

          - name: publish-to-release-branch
            script:
              - git checkout -b release
              - git add .
              - "git commit -m \"feat: автоматическое обновление сайта\""
              - "git push origin release -f"
```
- .sourcecraft/sites.yaml
```commandline
site:
  root: docs
  ref: release
```

### 2.3 Создание и настройка репозиториев

**Sourcecraft**

- Создание публичной организации

- Создание пустого репозитория

- Создание персонального токена (Maintainer, срок 1 год)

- Привязка удаленного репозитория к локальному:
```bash
git remote add sourcecraft  https://baibakovalmaz:<token>@git.sourcecraft.dev/baibakovalmaz/static-site.git
```

- Проверка списка удаленных репозиториев:
```bash
git remote -v
```
```commandline
origin  https://github.com/Aqua223/static-site.git (fetch)
origin  https://github.com/Aqua223/static-site.git (push)
sourcecraft     https://baibakovalmaz:<token>@git.sourcecraft.dev/baibakovalmaz/static-site.git (fetch)
sourcecraft     https://baibakovalmaz:<token>@git.sourcecraft.dev/baibakovalmaz/static-site.git (push)

```
**GitHub**

- Создан публичный репозиторий

- В настройках репозитория GitHub Pages выбрано развертывание из ветки.
Исходники будут храниться в ветке release в корневой папке (root).


### 2.4 Механика деплоя
**Sourcecraft**
```bash
git push sourcecraft main
```
**GitHub**
```bash
git push origin main
```

### 3. Выводы
В ходе работы выполнены сценарии автоматического развертывания статического сайта на mkdocs с использованием
SourceCraft и GitHub Actions. В одном локальном репозитории настроены два удаленных. Деплой на SourceCraft
выполняется через push папки site в ветку pages. Деплой на GitHub выполняется автоматически через GitHub Actions 
при push в ветку main.

Теперь нам не нужно задумываться о сборке сайта каждый раз, когда мы его редактируем. 
С настроенным CI/CD пайплайном нам лишь нужно редактировать md файлы, а сборка и развертывание
сайта будет производиться автоматически после того, как мы запушим изменения.