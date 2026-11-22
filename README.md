# devops course 2026

Практические работы по дисциплине «Инструменты DevOps», РТУ МИРЭА.

## О репозитории

Здесь собраны материалы курса: настройка виртуальной машины, статический сайт с доставкой через rsync, приложение на Flask с PostgreSQL в Docker и автоматизация сервера через Ansible.

Виртуальная машина `devops-vm` работает на Ubuntu Server 24.04 (arm64). Параметры машины, сеть, SSH, межсетевой экран, веб-сервер и контейнеры описаны в [docs/server-setup.md](docs/server-setup.md).

Репозиторий также лежит на GitHub как `devops-mirea` и `devops-course-2026-mirror`, содержимое у них одинаковое.

## Что внутри

- **Ansible.** Четыре роли: `base` ставит пакеты и Docker, `ssh` создаёт пользователя, закрывает вход по паролю и включает межсетевой экран, `nginx` настраивает обратный прокси, `app` запускает контейнер `devops-ci-app` из образа в GHCR. Плейбук `verify.yml` только проверяет результат и ничего не меняет.
- **Приложение.** Flask и PostgreSQL в Docker Compose. На главной странице счётчик посещений, `/health` отвечает `ok`. Порт 5000 открыт только на `127.0.0.1`.
- **Сайт.** Одна статическая страница, которую `scripts/deploy.sh` доставляет на сервер через rsync. Скрипт отказывается работать при незафиксированных изменениях, а флаг `--dry-run` показывает, что будет передано.
- **Аудит.** `scripts/audit.sh` проверяет на сервере SSH, межсетевой экран, nginx, сертификат и контейнеры.

## Как запустить

Потребуются Git, Docker и Docker Compose.

Приложение локально:

```bash
git clone https://github.com/vleypi/devops-mirea.git
cd devops-mirea/app
cp .env.example .env
docker compose up --build
```

После запуска приложение доступно по адресу http://localhost:5000. В `.env` нужно заменить пароль `POSTGRES_PASSWORD`.

Настройка сервера через Ansible. Потребуется ansible-core, SSH-ключ `~/.ssh/devops_vm` и доступная машина из `ansible/inventory/prod.ini`:

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml
ansible-playbook verify.yml
```

## Структура

```
devops-course-2026/
├── ansible/
│   ├── inventory/       адреса серверов
│   ├── group_vars/      образ и порты приложения
│   ├── roles/           base, ssh, nginx, app
│   ├── site.yml         настройка и деплой
│   ├── verify.yml       проверка результата
│   └── requirements.yml коллекции Ansible
├── app/                 Flask, PostgreSQL, Docker Compose
├── scripts/
│   ├── deploy.sh        доставка сайта через rsync
│   └── audit.sh         проверка конфигурации сервера
├── site/                статический сайт
├── docs/                описание виртуальной машины
├── svn_comparison.md    сравнение Git и SVN
└── about_me.md          сведения о студенте
```
