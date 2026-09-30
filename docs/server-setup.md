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
| `02-keys-configured` | 29.09.2026, после настройки входа по ключу (задание 1) |
| `03-ssh-hardened` | 29.09.2026, после усиления защиты SSH (задание 2) |
