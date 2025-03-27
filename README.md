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
