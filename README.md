# Amnezia WARP Host Routing (с Routing Watchdog)

> **Это форк репозитория [LostGit77/amnezia-wg-warp-host-routing](https://github.com/LostGit77/amnezia-wg-warp-host-routing)**,
> который в свою очередь является форком [isultanov99/amnezia-warp-host-routing](https://github.com/isultanov99/amnezia-warp-host-routing).
>
> Основное дополнение этого форка — встроенный **Routing Watchdog**: автоматическое восстановление маршрутизации при потере правил iptables или policy routing.

---

## Что делает скрипт

Маршрутизирует исходящий трафик VPN-клиентов через Cloudflare WARP на уровне хоста:

```
Клиент → amn0 (AmneziaWG) → wg0 (WARP/3x-ui) → Cloudflare → Интернет
```

- Контейнер `amnezia-awg` или `amnezia-awg2` продолжает принимать входящий трафик на публичный IP VPS
- Исходящий трафик клиентов уходит через WARP-интерфейс на хосте
- Внешние сервисы видят Cloudflare IP вместо IP вашего VPS
- Дефолтный маршрут хоста **не изменяется**

---

## Что добавлено в этом форке

### Routing Watchdog

Отдельный пункт меню **"Install routing watchdog"**, который:

- Каждые **60 секунд** проверяет наличие правил `fwmark` и маршрутов через WARP-интерфейс
- При обнаружении потери маршрутизации автоматически перезапускает `amnezia-warp-routing@.service`
- Пишет события в системный лог (`journalctl -t amnezia-watchdog`)
- Работает как `systemd` timer — не потребляет ресурсов в ожидании

Это решает проблему, когда WARP-интерфейс (`wg0`) остаётся активным, но правила iptables / policy routing слетают (например, после перезагрузки Docker или обновления сети).

---

## Требования

- Linux с `systemd`
- Docker (запущен)
- Один из контейнеров: `amnezia-awg` или `amnezia-awg2`
- Root-доступ
- `iptables`, `python3`

Для автоматической установки WARP через `wgcf`:
- Ubuntu/Debian или RHEL-совместимый дистрибутив
- Доступ к GitHub и Cloudflare

---

## Установка

### Быстрая установка

```bash
curl -fsSL https://raw.githubusercontent.com/sn1341/amnezia-warp-autofix.sh/master/deploy_amnezia_warp_host.sh | sudo bash
```

### Или через клонирование репозитория

```bash
git clone https://github.com/sn1341/amnezia-warp-autofix.sh.git
cd amnezia-warp-autofix.sh
sudo bash deploy_amnezia_warp_host.sh
```

---

## Меню

```
Amnezia WARP Host Routing

Environment
  WAN interface    : enp0s3
  WAN IP           : 203.0.113.10
  WAN subnet       : 203.0.113.0/24
  WARP interface   : wg0
  Amnezia bridge   : amn0
  Routing watchdog : not installed

Containers
  AmneziaWG Legacy: not found
  AmneziaWG v2: found
    container IP: 172.29.172.2
    routing service: active
  Host WARP: found (wg0)

1) Install WARP and route all detected containers
2) Install or refresh routing for AWG v2 only
3) Install routing watchdog          ← новый пункт
4) Remove everything configured by this script
5) Show status
6) Exit
```

---

## Использование watchdog

После установки маршрутизации выбери в меню **"Install routing watchdog"**.

Полезные команды:

```bash
# Логи watchdog в реальном времени
journalctl -t amnezia-watchdog -f

# Статус таймера
systemctl status amnezia-warp-watchdog.timer

# Запустить вручную (для теста)
sudo bash /usr/local/sbin/amnezia-warp-watchdog.sh
```

---

## Неинтерактивная установка

```bash
# Установить маршрутизацию для всех найденных контейнеров
sudo AUTO_YES=1 bash deploy_amnezia_warp_host.sh

# Проверить статус
sudo bash deploy_amnezia_warp_host.sh status

# Удалить всё
sudo bash deploy_amnezia_warp_host.sh uninstall
```

---

## Переменные окружения

| Переменная | Описание | По умолчанию |
|---|---|---|
| `WARP_IF` | Имя WARP-интерфейса | автоопределение |
| `WARP_PROFILE_NAME` | Имя профиля wg-quick | `wgcf` |
| `WAN_IF` | WAN-интерфейс хоста | автоопределение |
| `AUTO_YES` | Пропустить меню, установить всё | `0` |

Пример:
```bash
sudo WARP_IF=wg0 WAN_IF=enp0s3 bash deploy_amnezia_warp_host.sh
```

---

## Что устанавливается

Скрипт записывает:

- `/usr/local/sbin/amnezia-warp-routing.sh` — основной скрипт маршрутизации
- `/etc/systemd/system/amnezia-warp-routing@.service` — systemd-сервис
- `/etc/amnezia-warp/*.env` — конфигурация для каждого контейнера
- `/etc/sysctl.d/99-amnezia-warp.conf` — настройки ядра

Если установлен watchdog:

- `/usr/local/sbin/amnezia-warp-watchdog.sh` — скрипт watchdog
- `/etc/systemd/system/amnezia-warp-watchdog.service`
- `/etc/systemd/system/amnezia-warp-watchdog.timer`

---

## Проверка после установки

Подключись через VPN и открой любой сервис проверки IP:

- [myip.com](https://www.myip.com/)
- [2ip.io](https://2ip.io/)
- [whatismyipaddress.com](https://whatismyipaddress.com/)

Ты должен увидеть IP из диапазона Cloudflare вместо IP своего VPS.

---

## Лицензия

MIT — см. [LICENSE](LICENSE)

---

## Благодарности

- [LostGit77](https://github.com/LostGit77) — непосредственный upstream этого форка
- [isultanov99](https://github.com/isultanov99) — оригинальный автор концепции
