# Orion deploy

Универсальное Ansible-приложение для автоматического деплоя Docker-приложений на удаленные серверы. Поддерживает автоматическую установку Docker, сборку и публикацию образов, а также деплой через docker-compose.

## Возможности

- ✅ Автоматическая установка Docker на удаленных серверах (Debian/Ubuntu и RedHat/CentOS)
- ✅ Сборка Docker-образов из Git-репозиториев
- ✅ Публикация образов в Docker Registry (Docker Hub, приватные registry)
- ✅ Деплой приложений через docker-compose
- ✅ Поддержка пользовательских шаблонов docker-compose
- ✅ Автоматическая очистка временных файлов
- ✅ Минимальная конфигурация - все настройки в `inventory`

## Требования

### На локальной машине (откуда запускается Ansible)

- Python 3.6+
- Ansible 2.9+
- SSH-ключ для доступа к удаленным серверам
- Ansible коллекция `community.docker` (для модулей работы с Docker: логин в registry, управление образами и docker-compose):
  ```bash
  ansible-galaxy collection install community.docker
  ```
  
  **Примечание:** Сборка Docker-образов всегда выполняется на удаленных серверах, поэтому Docker/Buildx на локальной машине не требуются.

### На удаленных серверах

- Linux (Debian/Ubuntu или RedHat/CentOS/Fedora)
- SSH-доступ с правами sudo
- Доступ к интернету (для установки Docker, клонирования репозиториев и публикации образов)
- Доступ к Git-репозиторию (SSH или HTTPS)
- Git установлен (для клонирования репозиториев)

## Установка

1. Клонируйте репозиторий:
   ```bash
   git clone https://github.com/vadkvadrad/orion-deploy.git
   cd orion-deploy
   ```

2. Установите необходимые Ansible коллекции:
   ```bash
   ansible-galaxy collection install community.docker
   ```
   
   **Важно:** Коллекция `community.docker` нужна для модулей деплоя (`docker_compose_v2`, `docker_image`, `docker_login`), которые используются в ролях `build-and-push` и `deploy-with-compose`. Встроенная роль `docker` только устанавливает Docker на серверах и не требует этой коллекции.

3. Настройте `inventory` (см. раздел "Настройка" ниже)

## Структура проекта

```
orion-deploy/
├── deploy.yml                    # Главный playbook
├── inventory/                    # Пользовательская директория (не коммитится в git)
│   ├── hosts.ini                 # Список серверов
│   ├── group_vars/
│   │   └── remote_servers.yml    # Переменные для серверов
│   └── templates/                # Опционально: пользовательские шаблоны
│       └── docker-compose.yml.j2
└── roles/
    ├── docker/                   # Установка Docker
    ├── build-and-push/           # Сборка и публикация образов
    └── deploy-with-compose/      # Деплой через docker-compose
```

## Настройка

### 1. Настройка inventory/hosts.ini

Укажите ваши удаленные серверы:

```ini
[remote_servers]
192.168.1.100 ansible_user=deploy ansible_ssh_private_key_file=~/.ssh/id_rsa
192.168.1.101 ansible_user=deploy ansible_ssh_private_key_file=~/.ssh/id_rsa
```

Или используйте доменные имена:

```ini
[remote_servers]
app1.example.com ansible_user=deploy
app2.example.com ansible_user=deploy
```

### 2. Настройка inventory/group_vars/remote_servers.yml

#### Обязательные переменные

```yaml
# === Build settings ===
app_git_repository: "git@github.com:username/repo.git"  # или https://github.com/username/repo.git
docker_image_name: "username/app-name"                  # Имя образа в registry
docker_username: "dockerhub_username"                   # Логин в Docker registry
docker_password: "dockerhub_token_or_password"         # Пароль или токен

# === Deploy settings ===
docker_image_name: "username/app-name"                  # Должно совпадать с build settings
```

#### Опциональные переменные

```yaml
# === Build settings (опционально) ===
app_git_branch: "main"                                  # Ветка для сборки (по умолчанию: main)
docker_registry_url: "https://index.docker.io/v1/"      # URL registry (по умолчанию: Docker Hub)
docker_platforms: "linux/amd64"                        # Платформы для сборки
docker_image_tag: "latest"                             # Тег образа

# === Deploy settings (опционально) ===
compose_project_dir: "/home/appuser/myapp"             # Путь на сервере (если пусто - используется /tmp)
compose_user: "appuser"                                # Пользователь-владелец
docker_image_tag: "latest"                             # Тег для деплоя

# Настройки docker-compose (для стандартного шаблона)
compose_environment:                                    # Переменные окружения
  NODE_ENV: "production"
  DATABASE_URL: "postgresql://..."
compose_ports:                                          # Проброс портов
  - "8080:8080"
  - "443:443"
compose_volumes:                                        # Тома
  - "/host/data:/app/data"
  - "/host/config:/app/config"
compose_networks:                                       # Сети
  - "frontend"
  - "backend"
```

## Использование

### Базовый деплой

```bash
ansible-playbook deploy.yml
```

Playbook выполнит три этапа:

1. **Install Docker** - установка Docker на удаленных серверах (если не установлен)
2. **Build and push Docker image** - сборка образа из Git-репозитория и публикация в registry на удаленных серверах
3. **Deploy application** - деплой приложения через docker-compose

**Важно:** Сборка всегда выполняется на удаленных серверах. Убедитесь, что серверы имеют доступ к Git-репозиторию и Docker registry.

### Проверка конфигурации перед запуском

```bash
ansible-playbook deploy.yml --check
```

### Запуск с дополнительными параметрами

```bash
# Увеличенная детализация вывода
ansible-playbook deploy.yml -v

# Запуск только на определенном сервере
ansible-playbook deploy.yml --limit 192.168.1.100

# Запуск без проверки SSH host key
ansible-playbook deploy.yml -e "ansible_ssh_common_args='-o StrictHostKeyChecking=no'"
```

## Примеры конфигурации

### Пример 1: Простое веб-приложение

```yaml
# inventory/group_vars/remote_servers.yml
app_git_repository: "git@github.com:username/myapp.git"
docker_image_name: "username/myapp"
docker_username: "dockerhub_user"
docker_password: "your_token"

compose_project_dir: "/opt/myapp"
compose_user: "appuser"
compose_ports:
  - "80:8080"
compose_environment:
  NODE_ENV: "production"
```

### Пример 2: Приложение с базой данных и томами

```yaml
# inventory/group_vars/remote_servers.yml
app_git_repository: "git@github.com:username/myapp.git"
docker_image_name: "username/myapp"
docker_username: "dockerhub_user"
docker_password: "your_token"

compose_project_dir: "/opt/myapp"
compose_user: "appuser"
compose_ports:
  - "8080:8080"
compose_volumes:
  - "/opt/myapp/data:/app/data"
  - "/opt/myapp/logs:/app/logs"
compose_environment:
  DATABASE_URL: "postgresql://db:5432/mydb"
  REDIS_URL: "redis://redis:6379"
```

### Пример 3: Использование временной директории (автоочистка)

Если не указать `compose_project_dir`, будет использована временная директория в `/tmp`, которая автоматически очистится после деплоя:

```yaml
# inventory/group_vars/remote_servers.yml
app_git_repository: "git@github.com:username/myapp.git"
docker_image_name: "username/myapp"
docker_username: "dockerhub_user"
docker_password: "your_token"

# compose_project_dir не указан - будет использован /tmp/ansible-compose-{timestamp}
compose_ports:
  - "8080:8080"
```


## Кастомизация шаблонов docker-compose

### Использование пользовательского шаблона

1. Создайте файл `inventory/templates/docker-compose.yml.j2`:

```yaml
version: '3.8'

services:
  app:
    image: "{{ docker_image_name }}:{{ docker_image_tag | default('latest') }}"
    container_name: "my-custom-app"
    restart: "unless-stopped"
    ports:
      - "{{ compose_ports[0] | default('8080:8080') }}"
    environment:
      - NODE_ENV=production
    volumes:
      - app_data:/app/data

volumes:
  app_data:
```

2. Шаблон будет автоматически использован вместо стандартного.

### Использование шаблона с другим именем

Если ваш шаблон называется иначе, укажите путь в переменных:

```yaml
# inventory/group_vars/remote_servers.yml
compose_template_path: "my-custom-compose.yml.j2"
```

## Переменные и их значения

### Build and Push роль

| Переменная | Обязательная | По умолчанию | Описание |
|-----------|--------------|--------------|----------|
| `app_git_repository` | ✅ | - | URL Git-репозитория |
| `docker_image_name` | ✅ | - | Имя образа в registry |
| `docker_username` | ✅ | - | Логин в Docker registry |
| `docker_password` | ✅ | - | Пароль/токен для registry |
| `app_git_branch` | ❌ | `main` | Ветка для сборки |
| `docker_registry_url` | ❌ | Docker Hub | URL registry |
| `docker_platforms` | ❌ | `linux/amd64` | Платформы для сборки |
| `docker_image_tag` | ❌ | `latest` | Тег образа |

### Deploy with Compose роль

| Переменная | Обязательная | По умолчанию | Описание |
|-----------|--------------|--------------|----------|
| `docker_image_name` | ✅ | - | Имя образа для деплоя |
| `compose_project_dir` | ❌ | `/tmp/ansible-compose-*` | Путь на сервере (если пусто - временный) |
| `docker_image_tag` | ❌ | `latest` | Тег образа |
| `compose_user` | ❌ | `appuser` | Пользователь-владелец |
| `compose_template_path` | ❌ | `docker-compose.yml.j2` | Путь к пользовательскому шаблону |
| `restart_policy` | ❌ | `unless-stopped` | Политика перезапуска контейнера |
| `compose_ports` | ❌ | `[]` | Список портов |
| `compose_environment` | ❌ | `{}` | Переменные окружения |
| `compose_volumes` | ❌ | `[]` | Тома |
| `compose_networks` | ❌ | `[]` | Сети |

## Устранение неполадок

### Ошибка: "Failed to connect to the host via ssh"

- Проверьте SSH-доступ к серверу: `ssh user@server`
- Убедитесь, что SSH-ключ указан правильно в `hosts.ini`
- Проверьте, что пользователь имеет права sudo

### Ошибка: "Docker build failed"

- Убедитесь, что Docker установлен на удаленном сервере (роль `docker` установит его автоматически)
- Проверьте, что в репозитории есть `Dockerfile`
- Проверьте доступ к Git-репозиторию с удаленного сервера

### Ошибка: "Failed to push image"

- Проверьте `docker_username` и `docker_password`
- Убедитесь, что у вас есть права на публикацию в registry
- Для Docker Hub используйте Personal Access Token вместо пароля

### Контейнер не запускается

- Проверьте логи: `docker logs <container_name>` на сервере
- Убедитесь, что порты не заняты другими приложениями
- Проверьте переменные окружения в `compose_environment`

## Безопасность

⚠️ **Важно:**

- Никогда не коммитьте `inventory/` в git (она уже в `.gitignore`)
- Используйте Ansible Vault для хранения паролей:
  ```bash
  ansible-vault encrypt inventory/group_vars/remote_servers.yml
  ansible-playbook deploy.yml --ask-vault-pass
  ```
- Используйте Personal Access Tokens вместо паролей для Docker Hub
- Ограничьте права пользователя на сервере (не используйте root)

## Лицензия

[Укажите вашу лицензию]

## Поддержка

[Укажите контакты для поддержки]

