## DevOps Lab 11 — CI/CD Pipeline

GitOps-подход: Git → GitHub Actions → Docker Hub → ArgoCD → Kubernetes

 CI/CD Status

### 📋 Содержание

```
- Архитектура
- Структура проекта
- CI Pipeline (dev)
- CD Pipeline (master)
- ArgoCD + Kubernetes
- Как это работает
- Проверка работы
```

### 🏗 Архитектура

```
plain
┌─────────────┐     git push      ┌──────────────────┐
│ Разработчик │ ────────────────► │   GitHub (dev)   │
└─────────────┘                   └────────┬─────────┘
                                           │
                                           ▼
                              ┌────────────────────────┐
                              │  GitHub Actions (CI)   │
                              │  • lint (pylint)       │
                              │  • unit-tests (pytest) │
                              │  • build-test (docker) │
                              └───────────┬────────────┘
                                          │ PR approved
                                          ▼
                              ┌────────────────────────┐
                              │   GitHub (master)      │
                              └───────────┬────────────┘
                                          │ push trigger
                                          ▼
                              ┌────────────────────────┐
                              │  GitHub Actions (CD)   │
                              │  • docker build & push │
                              │  • patch manifest      │
                              │  • push to release     │
                              └───────────┬────────────┘
                                          │
                                          ▼
                              ┌────────────────────────┐
                              │     Docker Hub         │
                              │  (devops-psu:latest)   │
                              └───────────┬────────────┘
                                          │
                                          ▼
                              ┌────────────────────────┐
                              │      ArgoCD            │
                              │  (watch release branch)│
                              └───────────┬────────────┘
                                          │ auto-sync
                                          ▼
                              ┌────────────────────────┐
                              │   Kubernetes Cluster   │
                              │  (minikube / prod)     │
                              └────────────────────────┘
```

### 📁 Структура проекта

```
plain
devops-lab11-cicd/
├── .github/
│   └── workflows/
│       ├── cicd.yml              # CI: lint + test + build-check (dev)
│       ├── release.yml           # CD: docker build/push + manifest patch (master)
│       └── devops_course_pipeline.yml  # Демо пайплайн
│
├── server/
│   ├── application.py            # Python HTTP-сервер (порт 8000)
│   ├── index.html                # Статическая страница (version 2)
│   ├── dockerfile                # Dockerfile (Alpine-based)
│   └── test_application.py       # Pytest юнит-тесты
│
├── server-k8s-manifests/
│   └── devops-psu.yml            # K8s Deployment + Service (LoadBalancer)
│
├── requirements.txt              # Python зависимости (pylint, pytest)
└── README.md                     # Этот файл
```
🔧 CI Pipeline (dev)
Триггер: push в ветку dev

### Table

| Job        | Описание                         | Зависимости      |
| ---------- | -------------------------------- | ---------------- |
| lint       | Pylint проверка кода             | —                |
| unit-tests | Pytest юнит-тесты                | —                |
| build-test | Сборка Docker-образа + curl-тест | lint, unit-tests |

```yaml
# .github/workflows/cicd.yml
name: LINT-TEST-BUILD-CHECK
on:
  push:
    branches: [dev]

jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.10' }
      - run: pip install -r requirements.txt
      - run: pylint server/application.py

  unit-tests:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.10' }
      - run: pip install -r requirements.txt
      - run: pytest

  build-test:
    needs: [lint, unit-tests]
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t test-image ./server
      - run: docker run -d -p 8000:8000 --name app test-image
      - run: sleep 15 && curl -s http://localhost:8000
```

### 🚀 CD Pipeline (master)

Триггер: push / merge в ветку master
| Job                  | Описание                                             |
| -------------------- | ---------------------------------------------------- |
| `push_to_registry`   | Логин в Docker Hub → build → push `latest`           |
| `touch-k8s-manifest` | Патчит `release-date` в манифесте → push в `release` |


```yaml
# .github/workflows/release.yml
name: Publish Docker image
on:
  push:
    branches: [master]

jobs:
  push_to_registry:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: ./server/
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/devops-psu:latest

  touch-k8s-manifest:
    needs: [push_to_registry]
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Insert release date
        run: |
          sed -i "s/release-date: .*$/release-date: $(date +%s)/g" \
            ./server-k8s-manifests/devops-psu.yml
      - name: Commit manifest
        run: |
          git config --global user.name 'Release Runner'
          git config --global user.email 'runner@users.noreply.github.com'
          git commit -am "Release"
          git push --force origin master:release
```

### ☸️ ArgoCD + Kubernetes

K8s Манифест

```yaml
# server-k8s-manifests/devops-psu.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-psu
  namespace: devops-psu
spec:
  replicas: 1
  selector:
    matchLabels:
      app: devops-psu
  template:
    metadata:
      labels:
        app: devops-psu
        release-date: "RELEASE-DATE"   # Патчится CD-пайплайном
    spec:
      containers:
        - name: devops-psu-server
          image: asokolovinterprogram/devops-psu:latest
          imagePullPolicy: Always       # Всегда тянуть свежий образ
          ports:
            - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: service-devops
  namespace: devops-psu
spec:
  type: LoadBalancer
  selector:
    app: devops-psu
    svc: frontend
  ports:
    - port: 12345
      targetPort: 8000
  externalIPs:
    - 10.0.2.15                      # IP ВМ VirtualBox
```

### ArgoCD Application
| Параметр        | Значение                                                         |
| --------------- | ---------------------------------------------------------------- |
| **Repository**  | `git@github.com:AndreySokolovInterprogram/devops-lab11-cicd.git` |
| **Branch**      | `release`                                                        |
| **Path**        | `server-k8s-manifests`                                           |
| **Namespace**   | `devops-psu`                                                     |
| **Sync Policy** | Automatic + Self Heal                                            |


### 🔄 Как это работает
Полный цикл обновления (version 1 → version 2)
```bash
# 1. Разработка в dev-ветке
git checkout dev
echo "Hello! I'm version 2" > server/index.html
# Обновляем dockerfile: ADD ./index.html ...
git add .
git commit -m "version 2"
git push origin dev

# 2. GitHub Actions запускает CI (lint → test → build-test)
#    Все проверки зелёные ✅

# 3. Создаём Pull Request: dev → master
#    Тимлид проводит код-ревью и мержит

# 4. Merge в master триггерит CD-пайплайн:
#    • Собирает Docker образ с index.html
#    • Пушит в Docker Hub
#    • Патчит release-date в манифесте
#    • Пушит манифест в ветку release

# 5. ArgoCD (следит за release) обнаруживает изменение
#    → Автосинхронизация → Новый Pod с version 2

# 6. Проверяем
curl http://10.0.2.15:12345
# Hello! I'm version 2
```

### ✅ Проверка работы
```bash
# Проверить статус подов
kubectl get pods -n devops-psu

# Проверить содержимое index.html внутри пода
kubectl exec -n devops-psu -it $(kubectl get pod -n devops-psu \
  -l app=devops-psu -o jsonpath='{.items[0].metadata.name}') \
  -- cat /usr/local/http-server/index.html

# Проверить ответ приложения
curl http://10.0.2.15:12345
```
### Ожидаемый результат
```plain
Hello! I'm version 2
```
### 🛠 Используемые технологии
| Технология                | Назначение                                   |
| ------------------------- | -------------------------------------------- |
| **Git + GitHub**          | Хранение кода и манифестов (source of truth) |
| **GitHub Actions**        | CI/CD оркестрация                            |
| **Docker + Docker Hub**   | Контейнеризация и хранилище образов          |
| **Kubernetes (minikube)** | Оркестрация контейнеров                      |
| **ArgoCD**                | GitOps-инструмент непрерывного деплоя        |
| **Python 3.10 + Alpine**  | Runtime приложения                           |
| **pylint + pytest**       | Линтинг и тестирование                       |


### 📌 Branch Protection
```
Ветка master защищена правилами:
❌ Прямой push запрещён
✅ Только через Pull Request
✅ Требуется прохождение CI-проверок
```
### 🎓 Лабораторная работа
```
Курс: DevOps (Л11 — CI/CD)
Тема: Непрерывная интеграция, доставка и развёртывание
Подход: GitOps (Git как единый источник правды)
Результат: Полностью автоматизированный пайплайн от коммита до продакшена.
```
## Автор

AndreySokolovInterprogram