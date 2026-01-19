# File Exchange - Простой обмен файлами

Легковесный веб-сервер для обмена файлами на базе FileBrowser с автоматическим HTTPS через Traefik.

## Описание

File Exchange - это простое решение для временного обмена файлами между контрактором и клиентом (банком). Использует FileBrowser с веб-интерфейсом для загрузки и скачивания файлов через защищённое HTTPS соединение.

## Возможности

- **Веб-интерфейс** - простая загрузка и скачивание файлов через браузер
- **Автоматический HTTPS** - SSL сертификаты через Let's Encrypt (Traefik)
- **Управление пользователями** - встроенная система авторизации
- **Загрузка больших файлов** - поддержка файлов любого размера
- **Просмотр файлов** - встроенный просмотр изображений, видео, текстовых файлов
- **Поиск** - быстрый поиск по файлам и папкам

## Быстрый старт

### Предварительные требования

- Docker и Docker Compose
- Работающий Traefik (из jenkins-installer)
- Домен с DNS записью

### Локальный деплой

```bash
# 1. Клонировать репозиторий
git clone https://github.com/qwq-qwq/file-exchange.git
cd file-exchange

# 2. Проверить/создать сеть Traefik
docker network create traefik_web-network || true

# 3. Настроить домен в docker-compose.yml (по умолчанию: files.perek.rest)
nano docker-compose.yml

# 4. Запустить
docker-compose up -d

# 5. Проверить статус
docker-compose ps
docker-compose logs -f
```

### Деплой через Jenkins

```bash
# 1. Настройте credentials в jenkins-installer/.env
FILEBROWSER_ADMIN_USERNAME=admin
FILEBROWSER_ADMIN_PASSWORD=your_secure_password_here

# 2. Перезапустите Jenkins для применения credentials
cd /path/to/jenkins-installer
docker-compose restart jenkins

# 3. Проект уже добавлен в jenkins-installer/config/jenkins/casc/jobs.groovy
# Просто запустите job 'file-exchange' в Jenkins UI
```

**Важно:** Credentials для FileBrowser автоматически настраиваются из переменных окружения Jenkins:
- `FILEBROWSER_ADMIN_USERNAME` - имя администратора (по умолчанию: admin)
- `FILEBROWSER_ADMIN_PASSWORD` - пароль администратора

Эти credentials автоматически применяются при деплое через Jenkins pipeline.

## Первый вход

После запуска откройте в браузере: `https://files.perek.rest` (ваш домен)

**Данные по умолчанию:**
- Логин: `admin`
- Пароль: `admin`

**ВАЖНО:** Сразу смените пароль после первого входа!

## Смена пароля администратора

1. Войдите как admin
2. Нажмите на иконку настроек (⚙️) в правом верхнем углу
3. Выберите "Settings" → "User Management"
4. Выберите пользователя admin
5. Измените пароль
6. Сохраните изменения

## Добавление новых пользователей

### Через веб-интерфейс

1. Войдите как admin
2. Settings → User Management
3. Нажмите "+ New User"
4. Заполните данные:
   - Username: имя пользователя
   - Password: пароль
   - Scope: `/srv` (по умолчанию)
   - Permissions: настройте права доступа
5. Сохраните

### Права доступа для пользователей банка

Рекомендуемые настройки для клиента (банка):

```
Username: bank_user
Scope: /srv
Permissions:
  ✓ Download (скачивание)
  ✓ Create (загрузка новых файлов)
  ✓ Rename (переименование)
  ✗ Delete (удаление запретить)
  ✗ Admin (админ права запретить)
  ✓ Share (создание ссылок для скачивания)
```

## Использование

### Загрузка файлов

1. Нажмите кнопку "Upload" в верхней панели
2. Перетащите файлы или выберите через диалог
3. Дождитесь завершения загрузки

### Скачивание файлов

1. Выберите файл
2. Нажмите кнопку "Download" или правой кнопкой → Download
3. Файл сохранится в папку загрузок браузера

### Создание ссылок для скачивания

1. Выберите файл
2. Нажмите "Share" (иконка цепочки)
3. Настройте параметры:
   - Expires: время жизни ссылки (по умолчанию без ограничений)
   - Password: пароль для доступа (опционально)
4. Скопируйте ссылку и отправьте клиенту

## Структура файлов

```
file-exchange/
├── docker-compose.yml          # Docker Compose конфигурация
├── Jenkinsfile                 # Pipeline для автоматического деплоя
├── config/
│   ├── settings.json           # Настройки FileBrowser
│   └── filebrowser.db          # База данных (создаётся автоматически)
├── data/                       # Папка с файлами (доступна в /srv)
└── README.md                   # Этот файл
```

## Настройка домена

Измените в `docker-compose.yml`:

```yaml
labels:
  - "traefik.http.routers.filebrowser.rule=Host(`your-domain.com`)"
```

После изменения:
```bash
docker-compose down
docker-compose up -d
```

## Ограничение размера загружаемых файлов

По умолчанию FileBrowser не ограничивает размер файлов. Для добавления ограничения отредактируйте `config/settings.json`:

```json
{
  ...
  "defaults": {
    ...
    "commands": [
      {
        "name": "upload",
        "maxSize": 1073741824
      }
    ]
  }
}
```

`maxSize` в байтах (1073741824 = 1GB)

## Безопасность

### Рекомендации для банка

1. **Сильные пароли** - используйте минимум 16 символов
2. **Отдельные пользователи** - создайте пользователей для каждого сотрудника
3. **Ограничение прав** - не давайте права Delete без необходимости
4. **Логирование** - проверяйте логи доступа: `docker-compose logs -f`
5. **Временные ссылки** - при создании Share ссылок устанавливайте Expires
6. **Firewall** - настройте файрволл на сервере, разрешите только 80, 443 порты
7. **Резервные копии** - регулярно делайте backup папки `data/`

### Просмотр логов доступа

```bash
# Просмотр всех логов
docker-compose logs -f

# Только последние 100 строк
docker-compose logs --tail=100

# Логи за последний час
docker-compose logs --since 1h
```

## Резервное копирование

### Ручное копирование

```bash
# Создать backup
tar -czf file-exchange-backup-$(date +%Y%m%d).tar.gz data/ config/

# Восстановить backup
tar -xzf file-exchange-backup-20240115.tar.gz
```

### Автоматическое копирование

Добавьте в crontab:

```bash
# Backup каждый день в 2:00 ночи
0 2 * * * cd /opt/projects/file-exchange && tar -czf /backup/file-exchange-$(date +\%Y\%m\%d).tar.gz data/ config/
```

## Остановка и удаление

```bash
# Остановить контейнер
docker-compose down

# Остановить и удалить данные
docker-compose down -v

# Полная очистка
docker-compose down -v
rm -rf data/ config/
```

## Траблшутинг

### Контейнер не запускается

```bash
# Проверить логи
docker-compose logs

# Проверить что Traefik запущен
docker ps | grep traefik

# Проверить что сеть существует
docker network ls | grep traefik_web-network
```

### SSL сертификат не получается

```bash
# Проверить что домен направлен на сервер
dig +short your-domain.com

# Проверить что порты 80, 443 открыты
sudo netstat -tlnp | grep -E ':(80|443)'

# Проверить Traefik логи
docker logs traefik
```

### Не могу войти (забыл пароль)

```bash
# Удалить базу данных (сбросит все пароли к admin/admin)
docker-compose down
rm config/filebrowser.db
docker-compose up -d
```

### Файлы не загружаются

```bash
# Проверить права на папку data
ls -la data/

# Дать права контейнеру
chmod -R 755 data/
```

## Интеграция с jenkins-installer

Проект автоматически интегрирован в jenkins-installer:

1. Job добавлен в `config/jenkins/casc/jobs.groovy`
2. Использует общую Traefik сеть `traefik_web-network`
3. Автоматический деплой при push в GitHub

Для применения изменений в Jenkins:

```bash
cd /path/to/jenkins-installer
docker-compose restart jenkins
```

## Лицензия

MIT

## Поддержка

Для вопросов и проблем создавайте issue в GitHub репозитории.