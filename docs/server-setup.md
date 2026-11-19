# Конфигурация виртуальной машины devops-vm

## 1. Параметры машины

| Параметр | Значение |
|---|---|
| Средство виртуализации | VirtualBox 7.2, хост macOS (Apple Silicon) |
| Гостевая ОС | Ubuntu Server 24.04.5 LTS (arm64) |
| Имя узла | `devops-vm`, полное имя `devops-vm.devops.local` |
| Оперативная память | 2048 МБ |
| Процессор | 2 ядра |
| Диск | 25 ГБ, VDI, динамически расширяемый, разметка LVM |

## 2. Сетевые интерфейсы

| Интерфейс | Адаптер VirtualBox | Адрес | Назначение |
|---|---|---|---|
| `enp0s8` | Адаптер 1, NAT | `10.0.2.15/24` (DHCP) | Выход в интернет, доступ с хоста через проброс порта |
| `enp0s9` | Адаптер 2, виртуальная сеть (Host-only) `HostNetwork` | `192.168.56.3/24` (DHCP) | Прямой доступ с хостовой системы по имени `devops.local` |

Интерфейс `enp0s9` настроен файлом `/etc/netplan/60-hostonly.yaml`.
DNS-серверы `8.8.8.8` и `1.1.1.1` заданы в `/etc/systemd/resolved.conf.d/dns.conf`.
На хостовой системе в `/etc/hosts` добавлена запись `192.168.56.3 devops.local`.

## 3. Правило проброса портов

| Имя | Протокол | Адрес хоста | Порт хоста | Порт гостя |
|---|---|---|---|---|
| ssh | TCP | 127.0.0.1 | 2222 | 2222 |

Изначально порт гостя был 22. Он изменён на 2222 после переноса службы SSH на нестандартный порт.

## 4. Учётные записи

| Имя | Группы | Способ аутентификации |
|---|---|---|
| `student` | `sudo` и стандартные группы установщика | Создана при установке. Вход по SSH запрещён директивой `AllowUsers` |
| `devops` | `devops`, `sudo`, `users` | Вход по SSH только по ключу ED25519 (`~/.ssh/devops_vm` на хосте). Пароль используется только для `sudo` |
| `root` | | Вход по SSH запрещён (`PermitRootLogin no`) |

## 5. Служба SSH

Порт: `2222`.

Изменённые директивы находятся в файле `/etc/ssh/sshd_config.d/99-hardening.conf`. Основной файл `/etc/ssh/sshd_config` не изменялся, его копия сохранена в `/etc/ssh/sshd_config.backup`.

| Директива | Значение |
|---|---|
| `Port` | `2222` |
| `PermitRootLogin` | `no` |
| `PasswordAuthentication` | `no` |
| `PubkeyAuthentication` | `yes` |
| `PermitEmptyPasswords` | `no` |
| `MaxAuthTries` | `3` |
| `LoginGraceTime` | `30` |
| `AllowUsers` | `devops` |
| `X11Forwarding` | `no` |
| `ClientAliveInterval` | `300` |
| `ClientAliveCountMax` | `2` |

В файле `/etc/ssh/sshd_config.d/50-cloud-init.conf` строка `PasswordAuthentication yes` закомментирована.
Юнит `ssh.socket` отключён, служба запускается через `ssh.service`.

## 6. Правила межсетевого экрана

Политики по умолчанию: `deny (incoming)`, `allow (outgoing)`, `disabled (routed)`. Журналирование: `medium`.

| Порт | Действие | Комментарий |
|---|---|---|
| `2222/tcp` | `LIMIT IN` | SSH rate-limited |
| `80/tcp` | `ALLOW IN` | HTTP |
| `443/tcp` | `ALLOW IN` | HTTPS |

Те же правила действуют для IPv6.

## 7. Снимки состояния

| Снимок | Момент создания |
|---|---|
| `01-clean-install` | 29.09.2026 20:37, после установки ОС и обновления пакетов |
| `02-keys-configured` | 15.10.2026, ветка от 01-clean-install, повторное выполнение задания 1 ПР № 5 |
| `03-ssh-hardened` | 29.09.2026, после настройки входа по ключу и усиления защиты SSH |
| `04-nginx-https` | 06.10.2026, после установки nginx, перевода ресурса на HTTPS и первого деплоя (ПР № 6) |

## 8. Веб-сервер

Пакет: nginx 1.24.0 из репозитория Ubuntu (sudo apt install -y nginx).

| Параметр | Значение |
|---|---|
| Конфигурация ресурса | /etc/nginx/sites-available/devops-site, ссылка в /etc/nginx/sites-enabled/ |
| Стандартный ресурс | ссылка /etc/nginx/sites-enabled/default удалена, конфигурация сохранена |
| Каталог ресурса | /var/www/devops-site, владелец devops:devops |
| Права ресурса | каталоги 755, файлы 644 (rsync --chmod=D755,F644) |
| Сертификат | /etc/ssl/certs/devops.crt, root:root, права 644 |
| Закрытый ключ | /etc/ssl/private/devops.key, root:root, права 600 |
| Протоколы TLS | TLSv1.2, TLSv1.3 |
| Перенаправление | HTTP (80) на HTTPS (443), код 301 |
| Журналы | /var/log/nginx/devops-site.access.log, /var/log/nginx/devops-site.error.log |

Команда формирования сертификата:

    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout /etc/ssl/private/devops.key \
      -out /etc/ssl/certs/devops.crt \
      -subj "/CN=devops.local" \
      -addext "subjectAltName=DNS:devops.local"

Срок действия: 365 дней, с 06.10.2026 по 06.10.2027.
На хостовой системе сертификат сохранён в ~/devops.crt и передаётся curl параметром --cacert.

## 9. Контейнеризация

Docker Engine 29.9.2 установлен из официального репозитория download.docker.com (пакеты docker-ce, docker-ce-cli, containerd.io, docker-buildx-plugin, docker-compose-plugin), пакет docker.io не используется. Плагин Docker Compose v5.6.1.

| Параметр | Значение |
|---|---|
| Участники группы `docker` | `devops` (имеет право `sudo`) |
| Проект на сервере | `/home/devops/app`, доставляется командой `rsync -avz --delete --exclude=.env app/ devops:app/` |
| Файл `.env` | `/home/devops/app/.env`, владелец `devops`, права `600`, пароль СУБД сформирован `openssl rand -hex 16`; шаблон `app/.env.example` |
| Сервис `db` | `postgres:16-alpine`, проверка `pg_isready`, порт 5432 не публикуется |
| Сервис `app` | образ `devops-app:1.0` на базе `python:3.12-slim`, gunicorn, пользователь `app` (uid 999) |
| Публикуемые порты | `127.0.0.1:5000` → `5000` контейнера `app` |
| Сеть | `app_backend` (bridge), СУБД доступна приложению по имени `db` |
| Именованный том | `app_db_data` → `/var/lib/postgresql/data` |
| Обратный прокси | в HTTPS-блоке `/etc/nginx/sites-available/devops-site` расположение `location /app/` с `proxy_pass http://127.0.0.1:5000/` и заголовками `Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`; копия конфигурации до изменения `devops-site.bak-pr6` |

## 10. Ansible

С ПР № 11 VM настраивается не вручную, а плейбуком Ansible. Перед первым запуском VM откатывается к снимку `01-clean-install`, поэтому настройки из разделов 3, 5, 8 и 9 (NAT-проброс 2222, HTTPS, Compose-проект) на ней больше не действуют. Нужное конечное состояние описано в ролях.

| Параметр | Значение |
|---|---|
| Расположение проекта | каталог `ansible/` этого репозитория; команды запускаются из него (там же лежит `ansible.cfg`) |
| Управляющий узел | хостовый Mac (Apple Silicon), ansible-core 2.21, коллекции из `ansible/requirements.yml` (ansible.posix 1.6.2, community.docker 4.5.0, community.general 9.5.0), sshpass для первичного запуска |
| Управляемый узел | `devops-vm`, Ubuntu Server 24.04 **arm64 (aarch64)**, Host-only `192.168.56.101` (`enp0s9`, закреплён в `/etc/netplan/60-hostonly.yaml`), SSH на порту 22 |
| Inventory | `inventory/bootstrap.ini` (`student`, пароль, только первый запуск); `inventory/prod.ini` (`devops`, ключ `~/.ssh/devops_vm`, по умолчанию в `ansible.cfg`) |
| Учётные записи | `student` — первичный доступ, после bootstrap вход по паролю отключён; `devops` — вход по ключу, группы `sudo` и `docker`, `NOPASSWD:ALL` в `/etc/sudoers.d/devops` |
| Роль `base` | проверка ОС (Ubuntu 24.04 aarch64), базовые пакеты, `python3-debian`, `python3-requests`, репозиторий download.docker.com (deb822, arm64), Docker Engine 29.10.0
1.24.0 |
| Роль `ssh` | пользователь `devops` и ключ, sudoers с `visudo -cf`, `/etc/ssh/sshd_config.d/00-ansible-hardening.conf` с `sshd -t` (пароли и root запрещены, `AllowUsers student devops`), UFW: deny incoming, allow 22/tcp и 80/tcp |
| Роль `nginx` | Nginx 29.10.0
(Ubuntu), сайт `/etc/nginx/sites-available/devops-ci-app` (порт 80, `proxy_pass http://127.0.0.1:5000`), default site отключён; handlers `nginx -t` → reload |
| Роль `app` | образ из GHCR и контейнер `devops-ci-app` с `restart_policy: unless-stopped` |
| Образ | `ghcr.io/vleypi/devops-ci-app` (мультиархитектурный linux/amd64 + linux/arm64) |
| Версия (полный SHA) | `1e3f7fc942f1f94687374a4c7034a25dd8392b7d`, переменная `app_tag` в `group_vars/all.yml`; `latest` не используется |
| Порт контейнера | `127.0.0.1:5000` → `5000`, снаружи доступен только через Nginx на 80 |
| Health endpoint | `GET /health` → `{"status":"ok"}` (`http://127.0.0.1:5000/health` на VM, `http://192.168.56.101/health` снаружи) |

Первичный запуск (чистая VM, пароль student):

    cd ansible
    ansible-galaxy collection install -r requirements.yml
    ansible-playbook -i inventory/bootstrap.ini site.yml --ask-pass --ask-become-pass

Повторный и все последующие запуски (неинтерактивно, ожидается `changed=0`):

    ansible-playbook site.yml

Read-only аудит (SSH без паролей, контейнер запущен, /health отвечает):

    ansible-playbook verify.yml

Особенности чистого снимка, найденные при первом запуске:

- после восстановления снимка часы VM отстают, и `apt` отклоняет Release-файлы как «not valid yet»; время синхронизируется командой `ansible -i inventory/bootstrap.ini vm -b -m shell -a "date -u -s '<UTC>' && timedatectl set-ntp true" --ask-pass --ask-become-pass`;
- сразу после этого запускается `unattended-upgrades` и занимает блокировку dpkg, поэтому перед bootstrap нужно дождаться её освобождения;
- в Ubuntu 24.04 sshd активируется через `ssh.socket`, и `/run/sshd` может отсутствовать, поэтому роль `ssh` создаёт этот каталог до проверки `sshd -t`.

Пароли, закрытый ключ и токены в репозитории не хранятся.
