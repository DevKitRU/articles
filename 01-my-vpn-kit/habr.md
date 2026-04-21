# Свой VPN для разработчика в РФ за день: sing-box, Claude Code и 4 скилла под каждую платформу

## Контекст — кто я и зачем это

Я не профессиональный разработчик. Основная работа — инженер-котельщик (техобслуживание, диагностика котлов). Начиналось всё с желания разобраться со своим сайтом — через Claude Code, не нанимая подрядчика. Затянуло: за несколько месяцев появились телеграм-боты под работу, SaaS-набор для коллег по профессии, блог. Подписка Claude Pro стала рабочим инструментом.

Параллельно вырос запрос на **стабильный интернет**. Публичные VPN периодически ломают логин в Anthropic — приходится перелогиниваться несколько раз в день. Hiddify с «Russia bypass» решает половину: большинство сайтов открывается, но Mini App газсервис-бота в Telegram не грузится, STALCRAFT ругается на регион, Discord зависает на «Starting». Смена режимов (proxy / system / VPN) помогает одним приложениям и ломает другим.

Потратил условный «один день» (по факту ~10 часов на Windows + часть ночи) на настройку своего VPN с нуля — разобрался со sing-box, Windows Service, routing rules под российскую специфику. Собрал всё в публичный open-source репо, чтобы другим такого же не проходить.

**TL;DR.** Репо: [github.com/DevKitRU/my-vpn-kit](https://github.com/DevKitRU/my-vpn-kit), MIT. Четыре Claude Skills (3 клиентских + 1 серверный от [AndyShaman](https://github.com/AndyShaman/3x-ui-skill)), устанавливают полный стек своего VPN за 1-2 часа. Стоимость — ~670 ₽/мес за VPS. Ниже — что работает, что нет, какие грабли и как их обойти.

---

## 1. Почему именно сейчас болит

Три конкретные проблемы накладываются:

**Anthropic блокирует датацентровые российские IP.** `claude auth login` из РФ даёт `403 Forbidden`. Решается прокси через свой non-RU VPS с sing-box + VLESS Reality (маскировка под Chrome TLS-fingerprint). Обычный OpenVPN/WireGuard не помогает — их фингерпринт палится. На самом ключе от сервера тоже OVH Канада, без дополнительной маскировки — тоже 403.

**Публичные платные VPN меняют exit-IP.** После переключения Anthropic снова ловит изменившийся fingerprint и выкидывает. Приходится перелогиниться. Для разработчика это ломает рабочий цикл.

**Российские сервисы на иностранных TLD банят не-RU IP.** STALCRAFT (`stalcraft.net`), игры Mail.ru (`my.games`, `my.com`), xsolla, часть банков (`sber.com`, `tinkoff.com` — международные страницы), некоторые госсервисы. Если гнать весь трафик через VPN — регион теряется, логин в эти сервисы не проходит. Hiddify с режимом «Russia bypass» закрывает часть через `.ru` суффикс и `geoip-ru`, но стандартный список не ловит домены на `.com/.net/.io`.

В результате типичный разработчик в РФ держит **два-три прокси одновременно** (для AI-сервисов, для игр, для Telegram) и переключается руками. Я устал и решил собрать один стек.

---

## 2. Что уже есть в open-source

| Проект | Что делает | Звёзд | Покрывает ли наш кейс |
|---|---|---|---|
| [SagerNet/sing-box](https://github.com/SagerNet/sing-box) | Ядро (универсальный proxy с TUN, VLESS Reality, gvisor) | 23k | Движок, нужен клиент-обёртка |
| [savely-krasovsky/antizapret-sing-box](https://github.com/savely-krasovsky/antizapret-sing-box) | Генератор rule-sets по спискам Роскомнадзора | 335 | Только `.srs` файлы, клиент нужен отдельно |
| [AndyShaman/3x-ui-skill](https://github.com/AndyShaman/3x-ui-skill) | Claude Skill: поднимает 3x-ui на VPS с нуля | 34 | **Серверная часть — берём её как зависимость** |
| [Nocturnal-ru/singbox-service_single](https://github.com/Nocturnal-ru/singbox-service_single) | sing-box как Windows Service, автообновление antizapret | 1 | Похоже на нашу задачу, но малоизвестно и заточено под один use-case |
| [runetfreedom/russia-blocked-geosite](https://github.com/runetfreedom) | Готовые geosite-списки заблокированного в РФ | — | Только данные |

Пустая ниша — **клиентский стек, который учитывает «банки на .com» и ориентирован на разработчика с Claude Code**. Именно это я и сделал.

---

## 3. Архитектура

```
┌────────────────────────────────────┐
│  VPS с non-RU IP (OVH, Hetzner…)   │
│  ┌──────────────────────────────┐  │
│  │ 3x-ui + VLESS Reality        │  │ ← ставится через
│  │ (AndyShaman/3x-ui-skill)     │  │   AndyShaman/3x-ui-skill
│  └──────────────────────────────┘  │
└─────────────┬──────────────────────┘
              │ VLESS на 443 с utls=chrome
              │
┌─────────────▼──────────────────────┐
│  sing-box клиент (my-vpn-kit)      │
│  ┌──────────────────────────────┐  │
│  │ routing rules:               │  │
│  │  .ru/.рф/.su       → direct  │  │
│  │  geoip-ru          → direct  │  │
│  │  stalcraft.net,              │  │
│  │  mail.ru, sber.com,          │  │
│  │  tinkoff.com, и т.д.→ direct │  │ ← та самая «банки на .com»
│  │  всё остальное     → VPN     │  │   боль которая закрывается
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
              │
   ┌──────────┼──────────┐
   ▼          ▼          ▼
 Windows    macOS      iOS
 Service  LaunchAgent Shadowrocket
 (WinSW)             (вручную)
```

На Windows sing-box запускается как Windows Service через WinSW с LocalSystem — стартует до логона, TUN-интерфейс поднимается автоматически, admin-права есть без UAC. На macOS — LaunchAgent (`KeepAlive=true`) плюс системный SOCKS через Network Preferences. На iOS — Shadowrocket с ручной настройкой (автоматизации в iOS SDK нет).

Стоимость: **~670 ₽/мес** за OVH Kimsufi. Плюс Shadowrocket $2.99 один раз в US App Store. Всё остальное бесплатно.

---

## 4. Матрица приложений: кто как читает прокси

Главная боль Windows — **разные приложения по-разному слышат прокси**. Это экспериментально проверено за 9 часов настройки:

| Приложение | Что читает | Если `HTTPS_PROXY=http://127.0.0.1:1080` | Если system proxy в реестре | Если TUN |
|---|---|---|---|---|
| Claude Code (CLI) | env `HTTPS_PROXY` | ✓ | ✗ (не читает реестр) | ✓ (через loopback) |
| curl, gh, pip, npm | env `HTTPS_PROXY` | ✓ | иногда | ✓ |
| Chrome / Edge | Windows system proxy | ✗ | ✓ | ✓ |
| Firefox | свои настройки `about:preferences` | ✗ | ✗ | ✓ |
| Telegram Desktop | system proxy или свои SOCKS5 в настройках | ✗ | ✓ | ✓ |
| Telegram Mini App (WebView2) | **только** system proxy | ✗ | ✓ | ✓ |
| Discord Desktop (Electron) | system proxy (с капризами) или флаги запуска | ✗ | иногда | ✓ |
| Steam клиент | игнорирует | ✗ | ✗ | ✓ |
| Игры через Steam | игнорируют | ✗ | ✗ | ✓ |

Вывод: **без TUN-режима половина приложений просто не видит прокси**. Это первая причина почему Hiddify в режиме «Proxy only» на Windows лично у меня не работал для Telegram Mini App — WebView2 не читал SOCKS-настройки Telegram клиента.

TUN-режим sing-box перехватывает весь IP-трафик ещё на уровне сетевой карты, приложениям вообще не надо знать что есть прокси. Дальше sing-box роутит по своим правилам: `.ru` → direct, иностранное → через VLESS.

---

## 5. Windows — самая больная часть

Я попробовал 4 способа собрать рабочий автозапуск. Три сломались, один взлетел.

### Способ 1: Hiddify Next

**Что получил.** Установщик, GUI, встроенный RU bypass по региону.

**Что пошло не так.** Режим «Systemproxy» ставит `ProxyServer=http://127.0.0.1:12334` в реестре Windows. Telegram клиент и браузеры подхватывают. **WebView2 в Telegram Mini App** — подхватывает тоже, но `bot.gazservis72.ru` улетает через `.ru → direct → RU IP`, а сайт стоит за Cloudflare, который на RU IP иногда вылавливает «подозрительное» поведение и не отдаёт JS → Mini App висит на «Загрузка». Discord через system proxy не взлетает вообще — Electron не любит HTTP CONNECT из реестра в режиме Windows proxy.

### Способ 2: `.bat` в Startup

Простой: батник `sing-box run -c config.json` в `Startup` папке. Стартует при логоне.

**Не пошло.** `sing-box` без admin не может создать TUN-интерфейс → падает с «Access denied». Через `Start-Process -Verb RunAs` внутри батника — UAC prompt при каждом логоне. Для хомяка ок, для нормальной работы нет.

### Способ 3: Windows Task Scheduler с `RunLevel=Highest` + `LocalSystem`

**Не пошло.** Scheduled Task падал с `LastTaskResult=1` на старте TUN-интерфейса. Несколько часов дебага — причина не гуглится. Подозрение — особенность DispatcherQueue/COM в TaskScheduler контексте. Плюс отдельный баг — имя пользователя в кириллице («Администратор») через PowerShell `New-ScheduledTaskPrincipal -UserId 'Администратор'` ломается кодировкой → нужен SID.

### Способ 4: Windows Service через WinSW (сработал)

Канонический путь для длинных процессов на Windows.

`WinSW` (Windows Service Wrapper) — один `exe` + XML-конфиг, оборачивает любой exe в Windows-службу.

```xml
<!-- sing-box-service.xml -->
<service>
  <id>sing-box</id>
  <name>sing-box VPN</name>
  <executable>C:\ProgramData\sing-box\sing-box.exe</executable>
  <arguments>run -c "C:\ProgramData\sing-box\singbox.json"</arguments>
  <workingdirectory>C:\ProgramData\sing-box</workingdirectory>
  <startmode>Automatic</startmode>
  <onfailure action="restart" delay="10 sec"/>
  <log mode="roll-by-size">
    <sizeThreshold>10240</sizeThreshold>
    <keepFiles>2</keepFiles>
  </log>
</service>
```

```powershell
.\sing-box-service.exe install
Start-Service sing-box
```

Что получаем:
- Служба стартует **при старте Windows, до логона** (в отличие от TaskScheduler с `AtLogOn`)
- `LocalSystem` имеет admin-права автоматически → TUN-интерфейс создаётся без UAC
- Автоматический рестарт через 10 секунд если упадёт
- Логи с ротацией по размеру в рабочей папке

Это всё, что нужно. Дальше ставим env-переменную для Claude Code:

```powershell
foreach ($v in "HTTPS_PROXY","HTTP_PROXY","https_proxy","http_proxy") {
    [Environment]::SetEnvironmentVariable($v, "http://127.0.0.1:1080", "User")
}
```

и перезапускаем VSCode. Весь остальной трафик идёт через TUN.

**Почему не Hiddify.** Он хорош как «поставил-забыл» для обычного пользователя, но: его встроенный роутинг не учитывает РФ-платформы на иностранных TLD, custom rules в GUI не редактируются (только через кастомный sing-box XML, что отрицает смысл иметь Hiddify), и Electron-клиенты (Discord) через его system proxy ведут себя нестабильно. Плюс Hiddify обновляется и может в один момент поменять поведение.

**Почему не v2rayN.** Его routing хранится в SQLite (`guiNDB.db`), правила задаются через UI, программно управлять этим неудобно. И он тоже Electron-приложение, что добавляет ненужного веса.

**sing-box как Service** — минимальная зависимость (один Go-бинарник + WinSW-обёртка, всего ~40 МБ), конфиг в JSON, всё прозрачно.

---
