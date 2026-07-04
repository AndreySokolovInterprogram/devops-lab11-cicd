# DevOps Lab 11 — CI/CD Pipeline

## Цель
Создать полный CI/CD pipeline: от кода приложения до автоматического деплоя в Kubernetes через ArgoCD.

## Структура проекта
devops-lab11-cicd/
├── .github/
│   └── workflows/
│       ├── cicd.yml                    # CI: lint → unit-tests → build-test (push в dev)
│       └── release.yml                 # CD: docker build/push + patch manifest (push в master)
├── server/
│   ├── application.py                  # Python HTTP-сервер + класс TestMe
│   ├── test_application.py             # Pytest юнит-тесты
│   └── dockerfile                      # Dockerfile для сборки образа
├── server-k8s-manifests/
│   └── devops-psu.yml                  # Deployment + Service для приложения
├── requirements.txt                    # Зависимости (pylint, pytest)
├── PROJECT_CONTEXT.md                  # Этот файл — план и прогресс
└── README.md                           # Описание проекта
plain

## Git-ветки

| Ветка | Назначение |
|-------|-----------|
| `dev` | Разработка. Сюда пушат разработчики. Запускает CI (lint, test, build). |
| `master` | Проверенный код. Запускает CD (docker push + обновление манифеста). |
| `release` | Служебная ветка. CD-пайплайн форс-пушит сюда обновлённый манифест. ArgoCD следит за ней. |

## Прогресс выполнения

### Этап 1: Приложение и тесты
- [x] Создать `server/application.py` — HTTP-сервер
- [x] Создать `server/test_application.py` — pytest тесты
- [x] Создать `requirements.txt` — зависимости

### Этап 2: Докеризация
- [x] Создать `server/dockerfile`

### Этап 3: CI — GitHub Actions
- [ ] Создать `.github/workflows/cicd.yml`
- [ ] Создать ветку `dev`, запушить, проверить CI

### Этап 4: Защита веток
- [ ] Настроить Branch Protection для `master`

### Этап 5: CD — Docker Hub
- [ ] Зарегистрироваться в Docker Hub
- [ ] Создать репозиторий для образов
- [ ] Добавить Secrets в GitHub: `DOCKER_USERNAME`, `DOCKER_TOKEN`

### Этап 6: CD — GitHub Actions
- [ ] Создать `.github/workflows/release.yml`
- [ ] Создать `server-k8s-manifests/devops-psu.yml`

### Этап 7: Kubernetes
- [ ] Запустить minikube
- [ ] Применить манифест вручную, проверить работу

### Этап 8: ArgoCD
- [ ] Установить ArgoCD в minikube
- [ ] Настроить доступ к GitHub репо (SSH ключ)
- [ ] Создать Application в ArgoCD
- [ ] Проверить авто-синхронизацию

### Этап 9: Полный тест
- [ ] Изменить код → push в dev → PR → merge в master
- [ ] Убедиться, что CD отработал, ArgoCD применил изменения
- [ ] Проверить curl на приложение

---

## Статус: 🟡 В процессе — Этап 1 завершён, переходим к Этапу 3 (CI)