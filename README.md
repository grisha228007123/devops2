# CI/CD Pipeline Documentation

## Цели Pipeline

Основные цели нашего CI/CD pipeline:
- **Автоматизация тестирования** - обеспечение качества кода
- **Непрерывная интеграция** - регулярное слияние изменений
- **Безопасность** - проверка зависимостей и сканирование кода
- **Деплой** - автоматизированное развертывание приложения

---

## Этапы Pipeline

### Основные этапы выполнения:
1. **Сборка** - компиляция и подготовка артефактов
2. **Тестирование** - запуск unit и интеграционных тестов
3. **Сканирование безопасности** - проверка уязвимостей
4. **Деплой** - развертывание в целевое окружение

---

## Детализация шагов pipeline

| Шаг | Описание | Триггеры | Время выполнения |
|-----|-----------|----------|------------------|
| **Lint** | Проверка синтаксиса и стиля кода | Push, Pull Request | ~2 мин |
| **Unit Tests** | Запуск модульных тестов | Push, Pull Request | ~5 мин |
| **Build** | Сборка Docker образа | Push в main | ~7 мин |
| **Security Scan** | Сканирование на уязвимости | Push в main | ~4 мин |
| **Deploy to Staging** | Деплой на staging окружение | Push в main | ~10 мин |
| **Integration Tests** | Запуск интеграционных тестов | Успешный деплой на staging | ~8 мин |
| **Deploy to Production** | Деплой на production | Ручной запуск, тег | ~12 мин |

---

## Пример конфигурационного файла

```yaml
# .github/workflows/ci-cd-pipeline.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  release:
    types: [ published ]

env:
  NODE_VERSION: '18.x'
  DOCKER_IMAGE: my-app

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run unit tests
        run: npm test
        env:
          CI: true

  build-and-scan:
    runs-on: ubuntu-latest
    needs: lint-and-test
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Build Docker image
        run: |
          docker build -t $DOCKER_IMAGE:${{ github.sha }} .
      
      - name: Scan for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: $DOCKER_IMAGE:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-and-scan
    environment: staging
    
    steps:
      - name: Deploy to staging
        run: |
          ./scripts/deploy.sh staging ${{ github.sha }}
        env:
          KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}
