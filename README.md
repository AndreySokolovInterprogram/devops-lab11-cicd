# DevOps Lab 11 — CI/CD Pipeline

## Описание

Полный CI/CD pipeline для cloud-native приложения с использованием:
- **GitHub Actions** — CI/CD автоматизация
- **Docker Hub** — хранилище образов
- **Kubernetes (minikube)** — оркестрация контейнеров
- **ArgoCD** — GitOps непрерывная доставка

## Структура проекта
devops-lab11-cicd/
├── .github/workflows/        # GitHub Actions пайплайны
├── server/                   # Исходники приложения
├── server-k8s-manifests/     # Kubernetes манифесты
├── requirements.txt          # Python-зависимости
├── PROJECT_CONTEXT.md        # План и прогресс
└── README.md                 # Этот файл
plain

## CI/CD Pipeline

### CI (ветка dev)
- Линтинг кода (`pylint`)
- Юнит-тесты (`pytest`)
- Сборка и тест Docker-образа

### CD (ветка master)
- Сборка и публикация образа в Docker Hub
- Обновление Kubernetes манифеста
- Форс-пуш в ветку `release`

### ArgoCD
- Следит за веткой `release`
- Автоматически синхронизирует состояние кластера
- Откатывает ручные изменения (self-heal)

## Технологии

- Python 3.7
- Docker
- GitHub Actions
- Kubernetes
- ArgoCD

## Автор

AndreySokolovInterprogram