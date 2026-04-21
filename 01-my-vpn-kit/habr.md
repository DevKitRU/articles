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

## 6. macOS — проще, но есть нюансы

На macOS всё собирается за три команды:

```bash
brew install sing-box
# сгенерить конфиг в ~/.config/sing-box/config.json
# создать ~/Library/LaunchAgents/com.singbox.vpn.plist
launchctl load ~/Library/LaunchAgents/com.singbox.vpn.plist
```

LaunchAgent с `KeepAlive=true` и `RunAtLoad=true` — это тот же паттерн что Windows Service, только нативный для macOS. Падения перезапускает сам, стартует при логоне.

### Gotcha №1: `ALL_PROXY=socks5://...` не работает с Node.js

Первая ошибка, на которую я наступил: поставил `export ALL_PROXY=socks5://127.0.0.1:1080` в `~/.zshrc`, думая что это универсальное. Curl с таким env работает, Python requests — работает. **Node.js fetch / https — игнорирует**. Claude Code (CLI на Node.js) продолжал лететь мимо прокси и ловить 403.

Правильно:

```bash
export http_proxy=http://127.0.0.1:1080
export https_proxy=http://127.0.0.1:1080
export HTTP_PROXY=http://127.0.0.1:1080
export HTTPS_PROXY=http://127.0.0.1:1080
```

Схема `http://` на mixed-порт, а не `socks5://`. Sing-box mixed inbound принимает HTTP CONNECT на том же 1080 где слушает SOCKS.

### Gotcha №2: нативный Telegram не читает env-переменные

Нативные macOS-приложения не читают env, только Terminal-инструменты. Для Telegram / Chrome / Safari нужен **системный SOCKS**:

```bash
networksetup -setsocksfirewallproxy "Wi-Fi" 127.0.0.1 1080
networksetup -setsocksfirewallproxystate "Wi-Fi" on
```

Или через GUI: Системные настройки → Сеть → Wi-Fi → Подробнее → Прокси → SOCKS-прокси. После этого все GUI-приложения идут через sing-box.

### Gotcha №3: `detour: direct` у DNS-сервера ломает sing-box

Неочевидная вещь. В конфиге sing-box 1.13+ у DNS-серверов нельзя ставить `detour: direct`:

```json
{
  "tag": "local-dns",
  "type": "udp",
  "server": "77.88.8.8",
  "detour": "direct"
}
```

Ошибка на старте:

```
FATAL start service: start dns/udp[local-dns]:
detour to an empty direct outbound makes no sense
```

Причём **`sing-box check -c config.json` эту ошибку не ловит** — валидатор говорит OK, но рантайм падает. Правильно — без `detour`, sing-box сам решит:

```json
{ "tag": "local-dns", "type": "udp", "server": "77.88.8.8" }
```

Час потерял пока нашёл.

---

## 7. iOS через Shadowrocket — только ручками

Никакого `install.sh` на iOS быть не может — Apple не разрешает программно настраивать VPN из скрипта. Только пошаговая инструкция.

Shadowrocket покупается в **US App Store** за $2.99 (в российском его нет). Дальше:

1. Скопировать VLESS-ссылку из 3x-ui в буфер iPhone
2. Shadowrocket → плюс → **Импортировать из буфера**
3. Главный экран → нижняя плашка с режимами → **Прокси** (Global)
4. Включить VPN

### Безопасные настройки Shadowrocket (можно включать)

- **По требованию → Всегда включено** — Kill Switch (если VPN упал, трафик блокируется)
- **Разрешения → Уведомления** — статус в iOS уведомлениях
- **Разрешения → Буфер обмена** — удобный импорт серверов

### Опасные настройки (категорически нельзя)

- **Туннель → Включить все сети** (Force All Networks). Перехватывает трафик на всех сетевых интерфейсах, включая внутренние iOS — iCloud Private Relay, AirDrop, Bonjour. Результат: SSL certificate verification failed на случайных приложениях, крэш всего сетевого стэка iOS.
- **Общее → Модули**. Модули расшифровывают HTTPS для модификации трафика (адблок, инъекции) — Shadowrocket подменяет SSL-сертификаты своим CA. iOS видит «сертификат не от Apple / банка / Google» и отказывается подключаться. SSL ломается везде.
- **Прокси → Цепь прокси** — цепочка из 2+ серверов, падает любой — падает вся цепь.

У меня был инцидент: включил Force All Networks по своей ошибке — Shadowrocket крэшнулся, пришлось делать полный сброс. После этого правило простое: **если настройка не в списке «безопасных» — не трогать**.

Полный список безопасных/опасных настроек — в [ios/README.md](https://github.com/DevKitRU/my-vpn-kit/blob/main/ios/README.md) репозитория.

---

## 8. Split-tunneling: «банки на .com» — главная боль, которой нет в других проектах

Стандартный Russia bypass в Hiddify / antizapret-sing-box / runetfreedom работает по двум правилам:

1. `domain_suffix: [".ru", ".рф", ".su"]` — direct
2. `geoip-ru` (IP-диапазоны РФ по GeoIP базам) — direct

Этого достаточно для сайтов вроде yandex.ru, vk.com (резолвится в RU IP), habr.com (аналогично). Не достаточно для:

| Сервис | Домен | Почему не ловится стандартными правилами |
|---|---|---|
| STALCRAFT | `stalcraft.net` | `.net` не в `.ru` суффиксах; Cloudflare IP не в geoip-ru |
| Mail.ru Games | `my.games`, `my.com` | то же |
| Xsolla (платежи для игр) | `xsolla.com` | Cloudflare IP, не RU |
| Gaijin (War Thunder) | `gaijin.net`, `warthunder.com` | CDN не в RU |
| Steam серверы | `steampowered.com`, `steamcommunity.com` | CDN глобально |
| Sber международный | `sber.com`, `sberbank.com` | Cloudflare US |
| Tinkoff | `tinkoff.com` | то же |
| Wildberries / Ozon (международные API) | `wildberries.com`, `ozon.com` | часть трафика через US CDN |

Эти сервисы **проверяют IP клиента**. Зашёл с канадского IP (через VPN) — получаешь бан региона, кикнут с игрового сервера, банк не даст авторизоваться.

Правильное решение — явно добавить эти TLD в direct-список sing-box:

```json
{
  "domain_suffix": [
    ".ru", ".рф", ".su",
    "stalcraft.net", "mail.ru", "my.games", "my.com",
    "vk.com", "vk.ru", "userapi.com",
    "sberbank.com", "sber.com", "tinkoff.com", "alfabank.com",
    "steampowered.com", "steamcommunity.com", "steam-chat.com",
    "xsolla.com", "gaijin.net", "warthunder.com",
    "wildberries.com", "ozon.com", "avito.com",
    "gosuslugi.ru", "nalog.gov.ru"
  ],
  "action": "route",
  "outbound": "direct"
}
```

В my-vpn-kit эти правила **включены по умолчанию** и распределены по четырём пресетам:

- **default** — базовый Russia bypass с расширенным списком выше
- **gaming** — с упором на игровые платформы
- **dev** — только AI/dev-сервисы через VPN, всё остальное direct (максимально приближено к обычному РФ-инету, с VPN-прикрытием только для Anthropic / OpenAI / GitHub)
- **minimalist** — только `anthropic.com` и `openai.com` через VPN, всё остальное direct

Этот список — то, ради чего стоило делать свой проект. В открытых решениях, которые я видел на момент разработки, такого не было.

---

## 9. Claude-на-VPS — решение проблемы курицы и яйца

Отдельная боль, которая всплыла на этапе написания документации. Если читатель вообще новичок, у него **нет работающего Claude Code** на локальной машине (Anthropic его не пустит из РФ). Ставить npm-пакет через заблокированный API тоже не получится. А мой скилл как раз и нужен, чтобы поставить VPN, — но чтобы запустить скилл, нужен Claude.

Выход, который я использовал сам:

**Claude Code ставится не на локальную машину, а на VPS.** У VPS non-RU IP, Anthropic пускает его без вопросов. Последовательность:

```bash
# SSH на VPS
ssh root@твой_VPS
# Установка Node + Claude Code
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs
npm install -g @anthropic-ai/claude-code
# Логин — ссылку OAuth открываешь в любом браузере
claude auth login
```

Дальше **Claude на VPS** выполняет скилл AndyShaman (поднимает 3x-ui на этом же VPS), получает VLESS-ссылку и **через SSH диктует команды тебе для локальной машины**.

Для SSH с iPhone отлично работает **Termius** — бесплатный SSH-клиент с хорошим GUI. Всю установку можно сделать с телефона: подключился к VPS, запустил Claude, получил команды, переключился на ноут, выполнил. Или через обратный SSH с VPS на локалку (если на Windows включён OpenSSH Server).

Экономит 2-3 часа новичку + избавляет от необходимости покупать временный VPN только чтобы поставить постоянный.

Полный гайд — в [docs/first-time-setup.md](https://github.com/DevKitRU/my-vpn-kit/blob/main/docs/first-time-setup.md) репо.

---

## 10. Что не получилось / не проверено

Для чистоты — список того, что я **не** сделал:

- **Тест на чистой Windows 11 VM** — не проверил. Windows Sandbox на моей машине не запускается (конфликт с Kaspersky — известная несовместимость). Windows 10 Pro — проверено ребутами, работает.
- **Discord Desktop через sing-box** — не стабилен. Electron капризничает с SOCKS-прокси, иногда зависает на «Starting». Запасной путь — Discord Web в браузере, работает всегда.
- **TUN через Scheduled Task от SYSTEM** — падал в `LastTaskResult=1`. Не нашёл причину. Обошёл через Windows Service (WinSW) — там от LocalSystem TUN создаётся нормально.
- **Android-клиент** — не делал. Можно использовать тот же sing-box через [SFA](https://sing-box.sagernet.org/clients/android/), но пресетов под my-vpn-kit нет.

Всё это — кандидаты на следующие итерации через issues.

---

## 11. Итого и ссылки

Сухо:

- **Репо:** [github.com/DevKitRU/my-vpn-kit](https://github.com/DevKitRU/my-vpn-kit), MIT. 23 файла, 3 клиентских скилла + пресеты + troubleshooting.
- **Серверная часть:** [AndyShaman/3x-ui-skill](https://github.com/AndyShaman/3x-ui-skill) — поднимает 3x-ui на чистой VPS за 10 минут.
- **Стоимость:** ~670 ₽/мес за VPS (OVH Kimsufi в Канаде). Один раз $2.99 за Shadowrocket на iPhone.
- **Платформы:** Windows 10/11 (проверено на 10 Pro), macOS Intel/Apple Silicon, iOS (Shadowrocket).
- **Время установки:** 1-2 часа с нуля (включая аренду VPS и настройку).

### Что в репо посмотреть отдельно

- [windows/install.ps1](https://github.com/DevKitRU/my-vpn-kit/blob/main/windows/install.ps1) — парсер VLESS-ссылки на PowerShell, WinSW-установщик, idempotent env-переменные
- [shared/presets/](https://github.com/DevKitRU/my-vpn-kit/tree/main/shared/presets) — четыре пресета routing-таблицы, JSON с плейсхолдерами
- [docs/troubleshooting.md](https://github.com/DevKitRU/my-vpn-kit/blob/main/docs/troubleshooting.md) — FAQ по симптомам (Telegram не грузит / Discord висит / Steam в NA регионе)
- [docs/first-time-setup.md](https://github.com/DevKitRU/my-vpn-kit/blob/main/docs/first-time-setup.md) — гайд для нулёвого пользователя через Claude-на-VPS + Termius

### Что просит сообщество

PR приветствуются особенно по трём направлениям:

1. **Новые РФ-платформы в direct-список** — если ваш банк / маркетплейс / игра на иностранном TLD и банит не-RU IP, открывайте issue (есть шаблон) с доменами.
2. **Windows 11 / Windows 10 Home баги** — моя машина Windows 10 Pro, на других версиях могут быть грабли.
3. **Android-клиент** — пока только iOS, Android не делал.

### Зачем я это сделал

Меня задолбало платить за публичные VPN и ловить реконнекты. Проект — не для денег (MIT, поддержка через issues). Хотелось, чтобы начинающему разработчику в РФ (каким я был ещё год назад) не приходилось тратить день на танцы с клиентами и антивирусами.

Если пригодится — звёздочка на репо приятна. Баги в issues ценнее чем звёздочка.
