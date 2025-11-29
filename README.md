# GeoIP Timezone & Locale Patch для Chromium

## Описание
Патч добавляет функциональность автоматического определения timezone и locale на основе IP адреса через MaxMind GeoIP2.

## Возможности
- Флаг `--geoip=true` для автоматической настройки timezone и locale
- Автоматическое получение IP с http://ipinfo.io/ip
- Поддержка SOCKS5 прокси (автоматически определяет `--proxy-server` от Playwright)
- MaxMind GeoLite2-City база данных для определения местоположения
- Timeout 5 секунд - НЕТ fallback, браузер не запустится при ошибке

## Применение патча

### 1. Скопируйте файл `timezone-lang.patch` в папку Chromium src:
```bash
cd C:\Users\Vsevolod32\chromium\src
```

### 2. Примените патч:
```bash
git apply timezone-lang.patch
```

### 3. Скачайте MaxMind GeoLite2-City базу данных:
- Зарегистрируйтесь на https://dev.maxmind.com/geoip/geolite2-free-geolocation-data
- Скачайте GeoLite2-City.mmdb
- Поместите в: `C:\Users\Vsevolod32\chromium\src\chrome\browser\resources\geoip\GeoLite2-City.mmdb`

### 4. Соберите Chromium:
```bash
autoninja -C out\Default chrome
```

## Использование

### Без прокси (использует ваш реальный IP):
```bash
chrome.exe --geoip=true
```

### С SOCKS5 прокси через Playwright:
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(
        executable_path="out/Default/chrome.exe",
        proxy={"server": "socks5://127.0.0.1:12000"},
        args=['--geoip=true']
    )
    page = browser.new_page()
    # Timezone и locale автоматически установлены по IP прокси!
```

### Множество браузеров с разными прокси:
```python
proxies = open('proxy.txt').readlines()  # 127.0.0.1:12000, 127.0.0.1:12001, ...

for proxy in proxies:
    browser = playwright.chromium.launch(
        proxy={"server": f"socks5://{proxy.strip()}"},
        args=['--geoip=true']
    )
    # Каждый браузер получает свой timezone и locale!
```

## Проверка работы

Откройте DevTools Console и выполните:
```javascript
// Проверить timezone
Intl.DateTimeFormat().resolvedOptions().timeZone

// Проверить locale
navigator.language

// Пример для US proxy должен вернуть:
// timeZone: "America/New_York" (или другой US timezone)
// language: "en-US"
```

## Текущий статус

⚠️ **ВАЖНО**: Locale пока работает только частично. `icu::Locale::setDefault()` установлен, но `navigator.language` может не меняться. Требуется дополнительная работа для полной интеграции с Chrome locale system.

**Что работает:**
- ✅ Автоматическое получение IP через ipinfo.io
- ✅ SOCKS5 proxy support
- ✅ MaxMind GeoIP2 lookup
- ✅ Timezone установка (Intl.DateTimeFormat работает)
- ✅ 5 секунд timeout, нет fallback

**Что требует доработки:**
- ⚠️ navigator.language и полная интеграция locale

## Структура изменений

### Новые файлы:
- `content/browser/geoip/BUILD.gn` - Build конфигурация
- `content/browser/geoip/geoip_manager.h/cc` - MaxMind DB операции
- `content/browser/geoip/geoip_initializer.h/cc` - Инициализация GeoIP
- `content/browser/geoip/simple_http_client.h/cc` - HTTP клиент с SOCKS5
- `third_party/libmaxminddb/` - MaxMind C библиотека

### Измененные файлы:
- `content/public/common/content_switches.h/cc` - Добавлен флаг `--geoip`
- `content/browser/browser_main_loop.cc` - Вызов инициализации GeoIP
- `content/browser/BUILD.gn` - Зависимость на geoip модуль

## Следующие шаги

Для завершения функциональности locale необходимо:
1. Найти правильный способ установки `navigator.language` в content module
2. Возможно потребуется hook в renderer process для установки locale перед JavaScript инициализацией
3. Рассмотреть использование Mojo IPC для передачи locale из browser process в renderer

## Контакты

Если нужна помощь с применением патча или доработкой - создавайте issue.
