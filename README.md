<div align="center">

<img src="https://raw.githubusercontent.com/bobivpn/assets/main/banner.png" alt="BobiVPN Checker" width="100%" />

<h1>🦊 BobiVPN Checker</h1>

<p><strong>Высокопроизводительный Go-инструмент для массовой проверки VPN-ключей в двухэтапном pipeline с параллельным исполнением</strong></p>

<p>
  <a href="https://golang.org/"><img src="https://img.shields.io/badge/Go-1.26-00ADD8?style=for-the-badge&logo=go&logoColor=white" alt="Go" /></a>
  <a href="https://github.com/xtls/xray-core"><img src="https://img.shields.io/badge/Engine-Xray--core-blueviolet?style=for-the-badge" alt="Xray-core" /></a>
  <a href="https://github.com/lilendian0x00/xray-knife"><img src="https://img.shields.io/badge/Knife-v10-red?style=for-the-badge" alt="xray-knife" /></a>
  <img src="https://img.shields.io/badge/Platform-Linux%20%7C%20macOS%20%7C%20Windows-lightgrey?style=for-the-badge" alt="Platform" />
  <img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge" alt="License" />
</p>

</div>

---

> [!CAUTION]
> **ДИСКЛЕЙМЕР / DISCLAIMER**
>
> Данный проект создан **исключительно в образовательных целях** — для изучения сетевых протоколов, архитектуры Go-приложений, конкурентного программирования и CI/CD автоматизации.
>
> Проект **не предназначен** для обхода государственных блокировок, нарушения законодательства или условий использования каких-либо сервисов. Автор не несёт ответственности за способы применения данного кода третьими лицами. Использование программного обеспечения осуществляется на ваш собственный риск и в соответствии с законодательством вашей страны.
>
> *This project is intended for educational purposes only. The author takes no responsibility for any misuse.*

---

## Содержание

- [О проекте](#-о-проекте)
- [Архитектура](#-архитектура)
- [Поддерживаемые протоколы](#-поддерживаемые-протоколы)
- [Быстрый старт](#-быстрый-старт)
- [Конфигурация](#-конфигурация)
- [Форматы вывода](#-форматы-вывода)
- [GitHub Actions](#-github-actions)
- [Структура проекта](#-структура-проекта)
- [Производительность](#-производительность)

---

## 🔬 О проекте

BobiVPN Checker — это production-grade CLI-инструмент для автоматической проверки работоспособности VPN-конфигураций. В отличие от примитивных TCP-пинг-чекеров, этот инструмент запускает **Xray-core in-process** (без fork внешних процессов) и выполняет реальное end-to-end подключение — так же, как это делает v2rayNG или NekoBox.

### Ключевые характеристики

- **Двухэтапный pipeline**: быстрый ProxyGet-фильтр + глубокая TUN-проверка
- **Высокая конкурентность**: 500+ параллельных проверок на Stage 1, 50 на Stage 2
- **Race-стратегия**: каждый ключ проверяется против 3 эндпоинтов одновременно — побеждает первый ответивший
- **IP-leak детекция**: автоматическое отсеивание ключей не меняющих выходной IP
- **Умный прогресс**: TTY-режим (живой прогресс-бар) и CI-режим (milestone-логирование без спама)
- **Атомарные счётчики**: `sync/atomic` вместо mutex для нулевой контенции при высоком параллелизме

---

## 🏗 Архитектура

```
subscriptions.txt / SUBSCRIPTION_URLS (env)
           │
           ▼
┌─────────────────────────────────────┐
│  subscription.Load()                │
│  • Параллельный fetch (8 горутин)  │
│  • Base64 / plain-text парсинг     │
│  • Глобальная дедупликация         │
└──────────────┬──────────────────────┘
               │  []string (уникальные ключи)
               ▼
┌─────────────────────────────────────┐
│  Stage 1: ProxyGet Race            │
│  concurrency = 500 (default)       │
│                                    │
│  для каждого ключа параллельно:    │
│  ┌─ endpoint 1 (Cloudflare 204) ─┐ │
│  ├─ endpoint 2 (Google 204)     ─┤ │ → первый ответил → победа
│  └─ endpoint 3 (Apple hotspot)  ─┘ │
│                                    │
│  Фильтрация: ~90% мёртвых ключей  │
└──────────────┬──────────────────────┘
               │  []CheckResult (выжившие)
               ▼
┌─────────────────────────────────────┐
│  Stage 2: Deep Check               │
│  concurrency = 50 (default)        │
│                                    │
│  • Xray-core in-process            │
│  • Cloudflare trace → exit IP      │
│  • IP-leak детекция                │
│  ┌──────────────┐                  │
│  │ Geo (ip-api) │ ← параллельно   │
│  │ Speed test   │   с singleflight │
│  └──────────────┘                  │
└──────────────┬──────────────────────┘
               │  []CheckResult (рабочие)
               ▼
┌─────────────────────────────────────┐
│  output.SaveResults()              │
│  Генерация файлов + статистика     │
└─────────────────────────────────────┘
```

---

## 📡 Поддерживаемые протоколы

| Протокол | Варианты |
|----------|----------|
| **VLESS** | Reality, TLS, WebSocket, gRPC, HTTP/2 |
| **VMess** | TLS, WebSocket, gRPC |
| **Trojan** | TLS, WebSocket |
| **Shadowsocks** | AEAD ciphers, SIP003 plugins |
| **Hysteria2** | QUIC-based |
| **Hysteria** | QUIC-based (legacy) |
| **TUIC** | v5 |

---

## 🚀 Быстрый старт

### Требования

- Go 1.21+ (рекомендуется 1.26)
- Linux/macOS (для Stage 2 TUN-режима необходимы права root или CAP_NET_ADMIN)

### Установка и сборка

```bash
# Клонируйте репозиторий
git clone https://github.com/bobivpn/vpn-checker.git
cd vpn-checker

# Установите зависимости
go mod tidy

# Сборка под текущую платформу
go build -ldflags="-s -w" -o vpn-checker .

# Кросс-компиляция под Linux (для CI)
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 \
  go build -ldflags="-s -w" -o vpn-checker-linux .
```

### Запуск

```bash
# Стандартный запуск (обе стадии)
./vpn-checker

# Только Stage 1 (быстрый фильтр, root не нужен)
./vpn-checker --skip-stage2

# Кастомная параллельность
./vpn-checker --concurrency-stage1=1000 --concurrency-stage2=30
```

### Подписки

Создайте файл `subscriptions.txt` в корне проекта:

```
# Поддерживаются HTTP и HTTPS ссылки
# Строки начинающиеся с # — комментарии

https://example.com/sub/your-subscription
https://another.provider/api/v1/sub
```

**Альтернатива** — переменная окружения (удобно для CI):

```bash
export SUBSCRIPTION_URLS="https://example.com/sub1
https://example.com/sub2"
./vpn-checker
```

---

## ⚙️ Конфигурация

Все параметры находятся в [`internal/config/config.go`](internal/config/config.go).

| Параметр | Значение по умолчанию | Описание |
|----------|-----------------------|----------|
| `TimeoutProxyGet` | `5s` | Таймаут Stage 1 ProxyGet |
| `TimeoutTUN` | `20s` | Таймаут Stage 2 TUN-подключения |
| `TimeoutSpeedTest` | `12s` | Таймаут теста скорости |
| `SpeedTestBytes` | `500 000 байт` | Объём данных для speed-теста |
| `DefaultConcurrencyStage1` | `500` | Параллельность Stage 1 |
| `DefaultConcurrencyStage2` | `50` | Параллельность Stage 2 |

### CLI флаги

```
--concurrency-stage1  int   Параллельных ProxyGet проверок (default: 500)
--concurrency-stage2  int   Параллельных TUN туннелей (default: 50)
--skip-stage2               Пропустить Stage 2, сохранить результаты Stage 1
```

---

## 📁 Форматы вывода

| Файл | Описание |
|------|----------|
| `vpn.txt` | Рабочие ключи, оригинальный формат |
| `vpn_base64.txt` | То же, в Base64 |
| `vpn_renamed.txt` | Ключи с автоименованием: `🇩🇪 Germany \| Hetzner 1` |
| `vpn_renamed_base64.txt` | То же, в Base64 |
| `bobi_vpn.txt` | Профиль для **Happ** с subscription-заголовками |
| `bobi_vpn_base64.txt` | То же, в Base64 |
| `bobi_vpn_lite.txt` | Lite: только RU/DE/FR/FI/EE/LV/LT |
| `bobi_vpn_lite_base64.txt` | То же, в Base64 |
| `vpn_report.json` | Детальная статистика: IP, страна, ISP, latency, speed |
| `countries/*.txt` | Отдельные профили по каждой стране |

---

## 🤖 GitHub Actions

```yaml
name: VPN Check

on:
  schedule:
    - cron: '0 */6 * * *'
  workflow_dispatch:

jobs:
  check:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.26'
          cache: true

      - name: Build
        run: |
          GOOS=linux GOARCH=amd64 CGO_ENABLED=0 \
          go build -ldflags="-s -w" -o vpn-checker .

      - name: Run
        env:
          SUBSCRIPTION_URLS: ${{ secrets.SUBSCRIPTION_URLS }}
        run: |
          chmod +x ./vpn-checker
          ./vpn-checker --concurrency-stage1=500 --concurrency-stage2=30

      - name: Commit results
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add vpn.txt vpn_base64.txt vpn_renamed.txt vpn_renamed_base64.txt \
                  bobi_vpn.txt bobi_vpn_base64.txt \
                  bobi_vpn_lite.txt bobi_vpn_lite_base64.txt \
                  vpn_report.json countries/
          git diff --staged --quiet || \
            git commit -m "chore: update keys [$(date -u '+%Y-%m-%d %H:%M UTC')]"
          git push
```

Добавьте в `Settings → Secrets and variables → Actions`:

| Secret | Описание |
|--------|----------|
| `SUBSCRIPTION_URLS` | URL подписок, каждый с новой строки |

---

## 📂 Структура проекта

```
vpn-checker/
├── main.go                      # Точка входа, CLI флаги, оркестрация
├── go.mod / go.sum
├── subscriptions.txt            # Список URL подписок
├── xray-knife-master/           # Форк xray-knife v10 (local replace)
└── internal/
    ├── checker/
    │   └── checker.go           # Stage 1 + Stage 2 pipeline, ProgressReporter
    ├── config/
    │   └── config.go            # Константы, таймауты, флаги стран
    ├── geo/
    │   └── geo.go               # Геолокация: ip-api.com + кеш + singleflight
    ├── output/
    │   └── output.go            # Генерация файлов, сортировка, статистика
    └── subscription/
        └── subscription.go      # Парсинг подписок: plain/base64/regex + дедупликация
```

---

## ⚡ Производительность

| Решение | Эффект |
|---------|--------|
| `sync/atomic` в `ProgressReporter` | Нулевая контенция при 500+ горутинах |
| Race-стратегия на Stage 1 | Latency = `min(endpoints)`, не `sum` |
| `singleflight.Group` в geo-кеше | 1 HTTP-запрос на уникальный IP, а не N |
| Geo + Speed параллельно | `max(geo, speed)` вместо `geo + speed` |
| Буферизованный канал в `checkKeyRace` | Нет goroutine leak при записи победителя |
| Пакетный `subscriptionClient` | Переиспользование TCP-соединений |

---

<div align="center">
  <sub>Создано в образовательных целях · Используйте ответственно и в рамках закона</sub>
</div>
