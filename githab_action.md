### Гайдлайн по настройке CI/CD для Flutter Web App с использованием GitHub Actions, Docker и Nginx

Этот гайдлайн описывает процесс настройки Continuous Integration (CI) и Continuous Deployment (CD) для веб-приложения на Flutter. Мы используем GitHub Actions для автоматизации сборки, тестирования и развертывания. Стек: Flutter для веб-приложения, Docker для контейнеризации, Nginx как веб-сервер для обслуживания статических файлов Flutter web build.

#### Предварительные требования
1. **Репозиторий на GitHub**: У вас должен быть репозиторий с кодом Flutter web app. Если нет, создайте новый и добавьте код приложения.
2. **VPS**: Доступ к VPS (например, на Ubuntu 22.04 или выше). Установите Docker и Docker Compose:
    - Установите Docker: `sudo apt update && sudo apt install docker.io -y && sudo systemctl start docker && sudo systemctl enable docker`.
    - Установите Docker Compose: `sudo apt install docker-compose -y`.
    - Добавьте пользователя в группу Docker: `sudo usermod -aG docker $USER` (перелогиньтесь).
3. **SSH-доступ к VPS**: Сгенерируйте SSH-ключ на локальной машине (`ssh-keygen`), добавьте публичный ключ в `~/.ssh/authorized_keys` на VPS.
4. **Docker Registry**: Используйте Docker Hub или GitHub Container Registry (GHCR) для хранения Docker-образов. Зарегистрируйтесь и настройте аутентификацию.
5. **GitHub Secrets**: В настройках репозитория (Settings > Secrets and variables > Actions) добавьте секреты:
    - `DOCKER_USERNAME`: Имя пользователя Docker Hub/GHCR.
    - `DOCKER_PASSWORD`: Пароль или токен для Docker Hub/GHCR.
    - `SSH_HOST`: IP или домен VPS.
    - `SSH_USERNAME`: Пользователь VPS (например, ubuntu).
    - `SSH_PRIVATE_KEY`: Приватный SSH-ключ (в формате PEM, без пароля).
    - `SSH_PORT`: Порт SSH (по умолчанию 22).

#### Шаги настройки CI/CD
1. **Подготовка репозитория**:
    - Добавьте в корень репозитория файлы: `Dockerfile`, `nginx.conf`, `docker-compose.yml` (см. ниже).
    - Убедитесь, что в `pubspec.yaml` Flutter указана поддержка web: `flutter: sdk: flutter` и зависимости актуальны.

2. **Настройка GitHub Actions**:
    - Создайте директорию `.github/workflows/` в репозитории.
    - Добавьте файл `ci-cd.yml` (см. ниже). Он будет запускаться на push в main:
        - CI: Сборка Flutter web, тестирование (если есть тесты).
        - Build Docker-образа.
        - Push образа в registry.
        - CD: SSH на VPS, pull образа, перезапуск контейнера с помощью Docker Compose.

3. **Настройка VPS для деплоя**:
    - Создайте директорию на VPS, например `/app/myapp`.
    - Скопируйте `docker-compose.yml` на VPS в эту директорию (используйте SCP или вручную).
    - Убедитесь, что Docker запущен.
    - Для первого запуска: `cd /app/myapp && docker-compose up -d`.
    - Nginx будет слушать на порту 80 (или 443 для HTTPS, настройте сертификаты отдельно).

4. **Тестирование**:
    - Push в main: Проверьте Actions в GitHub (Actions tab).
    - После деплоя: Доступ к приложению по IP VPS (или домену) на порту 80.
    - Логи: `docker logs <container_name>` на VPS.

5. **Мониторинг и улучшения**:
    - Добавьте тесты в Flutter: `flutter test`.
    - Для HTTPS: Настройте Let's Encrypt с certbot в Nginx.
    - Масштабирование: Используйте reverse proxy если нужно.
    - Ошибки: Проверьте логи Actions и Docker.

#### Возможные проблемы и советы
- **Flutter build ошибки**: Убедитесь, что версия Flutter в workflow совпадает с вашей (используем 3.24.x как стабильную на 2025).
- **Docker push**: Если GHCR, используйте `ghcr.io/${{ github.repository_owner }}/myapp:latest`.
- **SSH**: Если ключ с паролем, используйте ssh-agent, но лучше без.
- **Безопасность**: Не храните секреты в коде. Ограничьте доступ к VPS.
- **Кастомизация**: Замените `myapp` на имя вашего проекта, `yourusername` на ваше имя в Docker Hub.

### Необходимые файлы
Ниже приведено содержимое всех необходимых файлов. Добавьте их в репозиторий (кроме `docker-compose.yml`, который скопируйте на VPS).

#### 1. `.github/workflows/ci-cd.yml` (GitHub Actions workflow)
```yaml
name: CI/CD for Flutter Web App

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.24.1'  # Актуальная стабильная версия
        channel: 'stable'

    - name: Install dependencies
      run: flutter pub get

    - name: Run tests
      run: flutter test  # Если тесты есть; иначе закомментируйте

    - name: Build web
      run: flutter build web --release

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ secrets.DOCKER_USERNAME }}/myapp:latest  # Замените myapp на имя образа

    - name: Deploy to VPS
      uses: appleboy/ssh-action@v1.0.3
      with:
        host: ${{ secrets.SSH_HOST }}
        username: ${{ secrets.SSH_USERNAME }}
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        port: ${{ secrets.SSH_PORT }}
        script: |
          cd /app/myapp  # Путь на VPS
          docker-compose pull
          docker-compose up -d --force-recreate
          docker image prune -f  # Очистка старых образов
```

#### 2. `Dockerfile` (Для сборки образа: multi-stage build)
```dockerfile
# Stage 1: Build Flutter web
FROM debian:bookworm-slim AS builder

RUN apt-get update && apt-get install -y curl git unzip xz-utils zip libglu1-mesa

RUN git clone https://github.com/flutter/flutter.git /flutter
ENV PATH="$PATH:/flutter/bin"
RUN flutter doctor
RUN flutter channel stable
RUN flutter upgrade

WORKDIR /app
COPY . /app
RUN flutter pub get
RUN flutter build web --release

# Stage 2: Nginx to serve static files
FROM nginx:alpine

COPY --from=builder /app/build/web /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### 3. `nginx.conf` (Конфигурация Nginx для обслуживания Flutter web)
```nginx
server {
    listen 80;
    server_name _;  # Или ваш домен

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

#### 4. `docker-compose.yml` (Для запуска на VPS; скопируйте на VPS)
```yaml
version: '3.8'

services:
  web:
    image: yourusername/myapp:latest  # Замените на ваш Docker Hub образ
    container_name: myapp_web
    ports:
      - "80:80"  # Или 443:443 для HTTPS
    restart: always
```

После добавления файлов, commit и push в main — CI/CD запустится автоматически. Если нужны доработки (например, для HTTPS или тестов), уточните!
