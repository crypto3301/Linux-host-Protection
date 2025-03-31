# Задание взято https://github.com/AmaHacka/secure-linux?tab=readme-ov-file
# Настройка окружения и интеграция Fail2Ban с Telegram

## Установка необходимых пакетов
```sh
sudo apt update
sudo apt install -y docker.io openssh-server nginx apache2-utils fail2ban apparmor apparmor-utils
```

## Регистрация Telegram-бота
1. Открываем Telegram и ищем @BotFather.
2. Отправляем команду `/newbot` и следуем инструкциям для создания бота.
3. Получаем токен для API бота.

## Получение ID чата
1. Открываем Telegram и пишем сообщение боту (например, отправляем `/start`).
2. Выполняем запрос:
```sh
curl -s "https://api.telegram.org/bot<YOUR_TELEGRAM_BOT_TOKEN>/getUpdates" | jq
```
3. Находим свой `chat_id` в ответе API.

## Установка дополнительных инструментов
```sh
sudo apt install -y curl jq
```

## Создание скрипта для уведомлений
```sh
sudo mkdir -p /etc/fail2ban/telegram
sudo tee /etc/fail2ban/telegram/notify.sh <<'EOF'
#!/bin/bash

# Конфигурация
TELEGRAM_TOKEN="YOUR_TELEGRAM_BOT_TOKEN"
CHAT_ID="YOUR_CHAT_ID"
BAN_ACTION="$1"
IP="$2"

# Формируем сообщение
if [ "$BAN_ACTION" == "ban" ]; then
    MESSAGE="🚨 IP $IP был заблокирован в Fail2Ban за подозрительную активность"
elif [ "$BAN_ACTION" == "unban" ]; then
    MESSAGE="🔓 IP $IP был разблокирован в Fail2Ban"
else
    exit 0
fi

# Отправляем сообщение
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage" \
    -d chat_id="${CHAT_ID}" \
    -d text="${MESSAGE}" \
    -d disable_notification="false" > /dev/null
EOF
```

### Даем права на выполнение
```sh
sudo chmod +x /etc/fail2ban/telegram/notify.sh
```

## Настройка Fail2Ban для использования скрипта

### Создание конфигурации действия для Telegram
```sh
sudo tee /etc/fail2ban/action.d/telegram.conf <<EOF
[Definition]
actionban = /etc/fail2ban/telegram/notify.sh ban <ip>
actionunban = /etc/fail2ban/telegram/notify.sh unban <ip>
EOF
```

### Добавление действия в правила Fail2Ban
```sh
sudo tee -a /etc/fail2ban/jail.d/ssh.local <<EOF
action = %(action_)s
         telegram
EOF

sudo tee -a /etc/fail2ban/jail.d/nginx.local <<EOF
action = %(action_)s
         telegram
EOF
```

# Тестирование

### 1. Проверка брутфорса HTTP-аутентификации
```bash
for i in {1..6}; do
    curl -s -u wronguser:wrongpass http://192.168.110.138;
    echo "Попытка $i: $(date)";
done
```

### 2. Запуск контейнера с ограничениями AppArmor
```bash
sudo docker run --security-opt "apparmor=docker-no-internet" --rm -it alpine ping -c 3 localhost
```

---

# ОГРАНИЧЕНИЕ Firefox с помощью AppArmor

# Определение версии Firefox

### Проверка установленной версии
```bash
snap list | grep firefox   # Для Snap-версии
apt list --installed firefox  # Для .deb-версии
```

---

# Настройка профиля AppArmor для Firefox

## 1. Настройка для .deb-версии

### A. Создание кастомного профиля
```bash
sudo nano /etc/apparmor.d/firefox-downloads-only
```
**Содержимое файла:**
```apparmor
#include <tunables/global>

profile firefox-downloads-only flags=(attach_disconnected) {
  # Базовые разрешения
  #include <abstractions/base>
  #include <abstractions/gnome>
  
  # Исполняемый файл Firefox
  /usr/lib/firefox/firefox mr,
  /usr/lib/firefox/** mr,
  
  # Профиль и кэш Firefox
  owner @{HOME}/.mozilla/firefox/** rw,
  owner @{HOME}/.cache/mozilla/** rw,
  
  ### === Разрешить запись ТОЛЬКО в Downloads ===
  owner @{HOME}/Downloads/ rw,
  owner @{HOME}/Downloads/** rw,
  
  ### === Явные запреты ===
  deny @{HOME}/Pictures/** w,
  deny @{HOME}/Documents/** w,
  deny @{HOME}/Videos/** w,
  deny @{HOME}/Music/** w,
  deny @{HOME}/Desktop/** w,
  deny @{HOME}/.* w,           # Скрытые файлы
  deny @{HOME}/* w,            # Корень домашней директории
  
  # Системные разрешения (только чтение)
  /etc/firefox/** r,
  /usr/share/firefox/** r,
}
```

### B. Применение профиля
```bash
sudo apparmor_parser -r /etc/apparmor.d/firefox-downloads-only
sudo aa-enforce firefox-downloads-only
```

### C. Запуск Firefox с ограничениями
```bash
firefox --profile /etc/apparmor.d/firefox-downloads-only
```

---

## 2. Настройка для Snap-версии

### A. Редактирование Snap-профиля
```bash
sudo nano /var/lib/snapd/apparmor/profiles/snap.firefox.firefox
```
Добавьте перед последней `}`:
```apparmor
  ### === Кастомные правила ===
  # Разрешить только Downloads
  owner @{HOME}/Downloads/ rw,
  owner @{HOME}/Downloads/** rw,
  
  # Явные запреты
  deny @{HOME}/Pictures/** w,
  deny @{HOME}/Documents/** w,
  deny @{HOME}/Videos/** w,
  deny @{HOME}/Music/** w,
  deny @{HOME}/Desktop/** w,
```

### B. Применение изменений
```bash
sudo apparmor_parser -r /var/lib/snapd/apparmor/profiles/snap.firefox.firefox
sudo systemctl restart apparmor
```

---

# Проверка работы

### Тест записи
```bash
# Должно работать:
touch ~/Downloads/test.txt

# Должно быть запрещено:
touch ~/Documents/test.txt
```

### Проверка логов
```bash
sudo tail -f /var/log/syslog | grep "apparmor.*DENIED.*firefox"
```


