# Установка и настройка 3X-UI панели на VPS

Руководство по установке **3X-UI** с протоколом **VLESS + REALITY** и маршрутизацией выбранных доменов через **Cloudflare WARP** (в режиме SOCKS5-прокси).

## 1. Подготовка сервера

### Обновление системы
```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### Проверка скорости интернета, CPU и RAM
```bash
wget -qO- bench.sh | bash
```

### Проверка GEO IP сервера

```bash
# Вариант 1
bash <(wget -qO- https://github.com/Davoyan/ipregion/raw/main/ipregion.sh)

# Вариант 2
bash <(wget -qO- https://github.com/vernette/ipregion/raw/master/ipregion.sh)
```

## 2. Установка 3X-UI панели

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

После установки зайдите в веб-панель (обычно `http://IP_сервера:2053` или порт, который укажет скрипт).  
Логин и пароль по умолчанию **рекомендуется сразу сменить**.

## 3. Установка Cloudflare WARP (в режиме прокси)

Официальный источник: [https://pkg.cloudflareclient.com/#ubuntu](https://pkg.cloudflareclient.com/#ubuntu)

```bash
# 1. Добавляем ключ репозитория
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | sudo gpg --yes --dearmor --output /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg

# 2. Добавляем репозиторий
echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflare-client.list

# 3. Устанавливаем WARP
sudo apt-get update && sudo apt-get install cloudflare-warp -y
```

### Настройка и запуск WARP

```bash
# Регистрация нового аккаунта WARP
warp-cli registration new

# Переключаем в режим прокси (SOCKS5)
warp-cli mode proxy

# Подключаемся
warp-cli connect

# Проверяем статус
warp-cli status
```

WARP будет слушать **SOCKS5** на `127.0.0.1:40000`.

## 4. Настройка Xray в 3X-UI
### Добавление Outbound (исходящее подключение через WARP)

1. Зайдите в панель → **Xray Settings** → **Outbound**
2. Нажмите **Add Outbound**
3. Заполните поля:
    - Protocol: `socks`
    - Tag: `warp-cli` (или любое удобное название)
    - Address: `127.0.0.1`
    - Port: `40000`
4. Сохраните.

### Настройка маршрутизации (Routing Rules)

1. Перейдите в **Xray Settings** → **Routing Rules**
2. Нажмите **Add Rule**
3. В поле **Outbound Tag** выберите `warp-cli`
4. В условиях укажите домены, которые хотите маршрутизировать через WARP.

**Примеры правил** (можно добавлять через запятую или отдельными правилами):

- `geosite:openai` — ChatGPT
- `geosite:google`, `geosite:youtube`
- `geosite:netflix`, `geosite:spotify`
- Конкретные домены: `example.com,another.com`
- Все российские сайты: `geosite:category-gov-ru,regexp:.*\.ru$`

Важно: После добавления правил нажмите **Save Settings** вверху и **Restart Xray**.

## 5. Создание Inbound VLESS + REALITY (рекомендуется)

1. **Inbounds** → **Add Inbound**
2. Основные настройки:
    - **Protocol:** `vless`
    - **Port:** `443` (рекомендуется)
    - **Transmission:** `tcp`, `xhttp` или `ws` (на выбор)
3. В разделе **Security** выберите **Reality**
4. Нажмите **Get New Cert**
5. Настройте:
    - **uTLS:** `chrome` (или `firefox`)
    - **Dest:** `dl.google.com:443` (или другой маскировочный сайт)
    - **SNI:** `dl.google.com`
    - **ShortIds:** сгенерируйте
6. Создайте клиента (укажите Email, UUID и т.д.).
7. Сохраните.

Готово! Теперь у Вас есть:

- Быстрый и хорошо замаскированный **VLESS + REALITY**
- Часть трафика (указанные домены) идёт через **Cloudflare WARP**

### Полезные советы

- После любых изменений в настройках Xray всегда выполняйте **Restart Xray**.
- Если WARP не подключается — проверьте статус командой `warp-cli status` и попробуйте `warp-cli disconnect && warp-cli connect`.
- Для автоматического запуска WARP после перезагрузки сервера можно добавить его в **crontab** или **systemd**.
