# Память Claude Code между Mac, Windows и VPS: симлинк на git — и больше ничего не копирую руками

**TL;DR.** Claude Code ведёт память (`MEMORY.md` + заметки в `~/.claude/projects/<hash>/memory/`) локально на каждой машине — между устройствами ничего не синхронизируется. Я превратил эту папку в симлинк (Junction на Windows) на приватный git-репо, настроил auto-pull каждые 5 минут, добавил pre-commit hook для защиты от утечки токенов. Весь набор — [github.com/DevKitRU/claude-memory-sync](https://github.com/DevKitRU/claude-memory-sync), MIT. Установка на новой машине — 3 шага.

Под катом — проблема, архитектура, грабли по платформам (PowerShell 5.1 без BOM умеет ломать парсер на кириллице; cron без `$PATH` не видит `git`; Windows Task Scheduler не принимает `[TimeSpan]::MaxValue`) и отдельный крупный раздел про безопасность — потому что память Claude в git-репо это возможность слить ключ одной командой.

---

## Контекст — кто я и зачем

Я не профессиональный разработчик. Основная работа — инженер-котельщик (диагностика, ремонт котельного оборудования). С Claude Code начал разбираться потому что захотел сам править свой сайт, не нанимая подрядчика. За несколько месяцев к сайту добавились Telegram-боты, SaaS-набор для коллег-мастеров, блог. Подписка Claude Pro стала рабочим инструментом.

Работаю с трёх машин: MacBook дома, Windows-десктоп для «серьёзных задач», VPS для продакшн-сервисов. Claude Code на всех трёх. И вот здесь — разрыв.

Сценарий, который меня задолбал:

- **Понедельник, Mac:** рассказал Claude про нового клиента — фирма Х, у них проблемы с котлом марки Y, предпочитают WhatsApp а не Telegram. Claude прилежно записал в `memory/project_client_X.md` + добавил ссылку в `MEMORY.md`.
- **Вторник, Windows:** открываю сессию — Claude меня не узнаёт. *«Здравствуйте! Чем могу помочь?»* Спрашиваю про фирму X — «не имею информации».

Потому что `memory/` локальная. MacBook пишет в `/Users/sergey/.claude/projects/<hash>/memory/`, Windows читает из `C:\Users\Администратор\.claude\projects\<другой-hash>\memory\`, VPS — из своего. Три независимых мирка.

Решил один раз и навсегда собрать: симлинк на git-репо, auto-pull, pre-commit hook против утечек. Собрал — и заодно оформил в публичный open-source кит, чтобы другим не проходить то же самое.

---

## 1. Что уже есть в open-source

Перед тем как писать своё, искал готовые решения. Нашёл по сути три типа:

| Проект | Что делает | Покрывает ли задачу |
|---|---|---|
| [anthropics/claude-code](https://github.com/anthropics/claude-code) | Сам Claude Code CLI | Просто пишет память локально, про sync ничего не знает |
| [Awesome Claude Code](https://github.com/hesreallyhim/awesome-claude-code) | Подборка ресурсов | Раздел «Memory» — только доки Anthropic, без практики |
| Отдельные gist'ы и reddit-посты | «Я сделал rsync-скрипт» | Не работают на всех трёх платформах, без rollback, без защиты от утечек |

Anthropic официально эту задачу не решает — у них есть [Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills) как формат, но пока ни одного официального скила для memory-sync. В комьюнити куча хотелок на эту тему, конкретной «поставил-забыл» реализации не нашёл.

Эту нишу я и закрываю.

---

## 2. Архитектура

Идея — максимально простая, почти тривиальная: **локальная папка памяти Claude = симлинк на рабочую копию приватного git-репо.**

```
┌─────────────┐  git push     ┌──────────────┐  git pull (auto)  ┌─────────────┐
│    Mac      │ ────────────▶ │    GitHub    │ ─────────────────▶│     VPS     │
│  (симлинк)  │               │  (приватный  │                   │  (симлинк)  │
└─────────────┘               │   memory     │                   └─────────────┘
                              │    репо)     │
                              └──────────────┘
                                     ▲
                                     │ git pull (auto)
                                     │
                              ┌─────────────┐
                              │   Windows   │
                              │  (Junction) │
                              └─────────────┘
```

Что происходит:

1. Claude пишет в `~/.claude/projects/<hash>/memory/session_state.md`.
2. Потому что это симлинк — файл физически появляется в `~/Documents/claude-memory/memory/session_state.md`.
3. Я делаю `git push` (или это делает мой помощник сам через слово «сохрани»).
4. На других машинах cron / launchd / Task Scheduler каждые 5 минут тянут `git pull`.
5. Через 5-10 минут память видна везде.

Лаг 5 минут — норма. Это не чат-синхронизация, а документы. Если срочно — руками `git pull`.

### Почему симлинк, а не копирующий скрипт

Альтернатива «рсинк каждые 5 минут» падает на трёх вещах:

- **Гонки.** Claude пишет в момент копирования — часть файла уедет, часть останется локально. Редко, но бывает.
- **Потеря `mtime`.** При копировании время изменения сдвигается, конфликт-резолв становится угадайкой.
- **Сложность не меньше.** Всё равно нужен cron плюс отдельный процесс копирования — больше точек отказа.

Симлинк снимает все три: Claude пишет **прямо** в рабочую копию git.

### Почему Junction на Windows, не Symbolic Link

| Тип | Команда | Нужны admin-права |
|---|---|---|
| **Junction** | `mklink /J` или `New-Item -ItemType Junction` | **Нет** |
| Symbolic Link | `mklink /D` | Да (или Developer Mode) |

Junction — это reparse point уровня NTFS, не требует привилегии `SeCreateSymbolicLinkPrivilege`. Для Claude Code работает идентично обычной папке. Пользовательский скрипт, не требующий UAC — это правильно.

### Почему cron а не webhook

Webhook «GitHub → ping → git pull» работал бы быстрее. Но:

- Нужен публичный endpoint на каждой машине. Mac и Windows за NAT — геморрой.
- Секрет вебхука надо где-то хранить безопасно.
- Если машина офлайн — webhook потеряется; всё равно нужен fallback.

Cron `*/5 * * * *` покрывает 99% случаев с нулевой инфраструктурой. Офлайн-машина при включении первым же pull'ом догоняет всё.

---

## 3. Установка за 3 шага

Предполагаю что у тебя уже есть учётка GitHub и установлен Claude Code.

### Шаг 1. Заведи приватный репо для своей памяти

На GitHub → New repository → имя `claude-memory`, **private**, без README (мы положим свой). Склонируй:

```bash
# macOS
git clone git@github.com:<твой-ник>/claude-memory.git ~/Documents/claude-memory

# Linux (VPS)
git clone git@github.com:<твой-ник>/claude-memory.git ~/claude-memory

# Windows (PowerShell)
git clone https://github.com/<твой-ник>/claude-memory.git E:\projects\claude-memory
```

### Шаг 2. Склонируй кит рядом

```bash
# macOS / Linux
git clone https://github.com/DevKitRU/claude-memory-sync.git ~/Documents/claude-memory-sync

# Windows
git clone https://github.com/DevKitRU/claude-memory-sync.git E:\projects\claude-memory-sync
```

Кит — это просто коллекция setup-скриптов и шаблонов. Твой памятный репо и кит — **разные** проекты.

### Шаг 3. Запусти setup

```bash
# macOS
cd ~/Documents/claude-memory-sync && ./setup/mac.sh

# Linux
cd ~/claude-memory-sync && ./setup/linux.sh

# Windows (PowerShell)
cd E:\projects\claude-memory-sync
.\setup\windows.ps1
```

Скрипт:
1. **Находит** активную папку памяти Claude через `~/.claude/projects/*/memory`. Если их несколько — спросит какую использовать.
2. **Бэкапит** её в `~/claude-memory-backup-<timestamp>`.
3. **Мержит** локальные уникальные/новые файлы в твой git-репо с памятью.
4. **Заменяет** локальную папку на симлинк/Junction к git-репо.
5. **Настраивает** auto-pull: launchd на Mac, cron на Linux, Task Scheduler на Windows.
6. **Тестирует** запись через симлинк.

Идемпотентен — повторный запуск не ломает настроенную систему.

### Проверка

```bash
./setup/health-check.sh     # Mac/Linux
.\setup\health-check.ps1    # Windows
```

Пять пунктов должны быть зелёными: симлинк, git-чистота, sync с GitHub, auto-pull работает, файлы совпадают.

### Откат

Если что-то пошло не так:

```bash
./setup/rollback.sh         # Mac/Linux
.\setup\rollback.ps1        # Windows
```

Восстанавливает папку из бэкапа, удаляет симлинк, убирает auto-pull задачу. Git-репо остаётся на месте для следующей попытки.

---

## 4. Грабли по платформам

Самое интересное. Что я накосячил — и как обошёл.

### Windows PowerShell 5.1 + кириллица = сломанный парсер

**Симптом:** запускаешь `.\setup\health-check.ps1` — на выходе строки вроде `Р¤Р°Р№Р»С‹` и парсер ругается на `Missing closing '}'` в коде где синтаксически всё очевидно правильное.

**Причина:** Windows PowerShell 5.1 (default на Win 10/11) без **UTF-8 BOM** читает `.ps1` как ANSI (cp1251 на русской системе). Кириллица UTF-8 превращается в мусор, и в какой-то момент ломает токенизатор.

**`chcp 65001` не помогает** — это про вывод консоли, а не про чтение файла.

**Решение:** все `.ps1` должны быть **UTF-8 with BOM** (`EF BB BF` в начале). Проверка:

```bash
head -c 3 file.ps1 | xxd  # должно быть: 00000000: efbb bf
```

Добавить BOM:
```bash
printf '\xef\xbb\xbf' > tmp && cat file.ps1 >> tmp && mv tmp file.ps1
```

В VS Code: правый нижний угол → `UTF-8` → Save with Encoding → `UTF-8 with BOM`.

Все скрипты в ките уже с BOM. Это простая вещь, которая хоронит скрипт на многих машинах.

### Windows Task Scheduler не принимает `[TimeSpan]::MaxValue`

**Симптом:** `Register-ScheduledTask` падает с `XML-документ содержит значение в неправильном формате: Duration:P99999999DT23H59M59S`.

**Причина:** XML-схема Task Scheduler ограничивает поле Days девятью цифрами, а `[TimeSpan]::MaxValue` даёт больше. StackOverflow молчит, сама MS-документация показывает примеры с `MaxValue` как если бы он работал — не работает.

**Решение:** 10 лет хватит:
```powershell
-RepetitionDuration (New-TimeSpan -Days 3650)
```

### Cron на Linux не видит `git`

**Симптом:** cron-запись есть (`crontab -l` показывает), а в логе ничего. `git pull` руками работает, из cron — нет.

**Причина:** cron не наследует `$PATH` твоего shell. В `/bin:/usr/bin` git есть, но на VPS с manual-установкой (brew, asdf, nvm) он может быть в `/opt/...` или `/usr/local/bin`.

**Решение:** использовать абсолютный путь к git (я его вычисляю через `command -v git` на этапе setup и вшиваю в wrapper-скрипт):

```bash
GIT_BIN=$(command -v git || echo "/usr/bin/git")

# В wrapper-скрипте:
"$GIT_BIN" pull --quiet >> "$LOG_FILE" 2>&1
```

В cron-записи — просто путь до wrapper'а, без пайпов и inline-команды.

### macOS LaunchAgent не видит SSH-ключ

**Симптом:** `launchctl list | grep claudememory` показывает агент, в логе — `Permission denied (publickey)`.

**Причина:** LaunchAgent запускается без твоего SSH-ключа: `ssh-agent` у агента другой процесс.

**Решение:** перейти на HTTPS + токен в credential helper, или переключить remote URL. GitHub принимает fine-grained PAT с доступом только к одному репо:

```bash
git remote set-url origin https://oauth2:<TOKEN>@github.com/<USER>/claude-memory.git
```

Токен кладётся в `~/.claude/secrets/api-keys.env` (см. секцию про безопасность), в URL подставляется явно.

### Discovery активной папки: почему это важно

В первой версии скрипта я захардкодил путь как `~/.claude/projects/-Users-$(whoami)/memory` для Mac, `e--projects\memory` для Windows. Работало у меня, у всех остальных — нет.

Потому что Claude Code создаёт папку проекта из абсолютного пути рабочей директории, заменяя `/` на `-`. У меня на Windows Claude запускается из `E:\projects\`, получается `e--projects`. У Васи будет `C--work-ai-projects`. У Пети `-Users-petya-dev-ai`. Никакого универсального имени нет.

Пришлось переписать на **discovery**:

```bash
candidates=()
while IFS= read -r -d '' dir; do
    candidates+=("$dir")
done < <(find "$HOME/.claude/projects" -mindepth 1 -maxdepth 1 -type d -print0)

# Если один — используем. Если несколько — спрашиваем у пользователя.
```

Это типичная ошибка любого «пишу для себя» → «выкладываю в open-source»: захардкоженный путь от твоего конкретного окружения. Помог **code review**: я прогнал скрипты через агента `code-reviewer` в Claude Code, и он нашёл этот блокер одним из первых. Без ревью в первый релиз 99% пользователей получили бы сломанный симлинк в несуществующую ветку.

---

## 5. Безопасность: не слейте токены в git

**Самое важное.** Память Claude теперь в git. Если Claude «услужливо» запишет в memory-файл строку `ANTHROPIC_API_KEY=sk-ant-...` — она уедет на GitHub **навсегда**. Даже если потом удалишь файл, история коммитов остаётся. Приватный репо не страховка: GitHub индексирует публичные форки, любой коллаборатор видит, случайное переключение на public — катастрофа.

У меня уже был такой инцидент: на старой системе (до этого кита) два Telegram-бот-токена уехали через memory-файл, пришлось отзывать и ротировать.

### Правило

**В `memory/*.md` НЕ должно быть реальных секретов.** Хранить можно только:
- *Где* ключ лежит: `~/.claude/secrets/api-keys.env`, 1Password vault.
- *Зачем* он нужен: «Anthropic API, личный, лимит $5/день».
- *Какой scope*: «только read-only, только метрики».

**Нельзя:** сам ключ. Ни в шуточном примере, ни «временно, потом уберу», никогда.

### Где хранить ключи правильно

- **Основное:** `~/.claude/secrets/api-keys.env` (chmod 600 на Mac/Linux, NTFS ACL на Windows — только твой пользователь). Обычный `.env`-файл вне git-репо памяти.
- **Альтернативы:** 1Password CLI (`op read "op://Private/github-token/credential"`), macOS Keychain, Bitwarden CLI. Ключи on-demand, никогда в plain-text файлах.

### Pre-commit hook — автоматическая защита

В ките есть `hooks/pre-commit` — bash-скрипт, который перед коммитом сканирует staged diff на 15+ форматов токенов:

```
Anthropic:   sk-ant-[a-zA-Z0-9_-]{20,}
OpenAI:      sk-(proj-)?[a-zA-Z0-9_-]{20,}
GitHub PAT:  ghp_[a-zA-Z0-9]{30,}, github_pat_...
Slack:       xox[bpoas]-...
Telegram:    \d{8,10}:[A-Za-z0-9_-]{35}
AWS:         AKIA[0-9A-Z]{16}
JWT:         eyJ...eyJ....-...
SSH private: -----BEGIN (RSA |OPENSSH )?PRIVATE KEY-----
Generic:     (API_?KEY|SECRET|TOKEN|PASSWORD)=<длинная строка>
```

Если что-то нашёл — **откажет в коммите**, покажет маскированную строку:

```
========================================
  PRE-COMMIT: найдены возможные секреты
========================================

! memory/project_ai.md:12 → Anthropic API key
    12:+ANTHROPIC_API_KEY=sk-ant-******
```

Установка — копируешь в `.git/hooks/pre-commit` своего memory-репо и `chmod +x`. Всё.

Для намеренных примеров в доке (где нужна строка похожая на ключ) — маркер `SECRETS_OK_PLACEHOLDER` в той же строке, hook пропустит.

### Ротация: что делать если ключ уже уехал

1. **Отзови ключ** в соответствующем сервисе (Anthropic console, GitHub Settings → Tokens, Telegram BotFather). Немедленно.
2. **Создай новый**, сохрани в `~/.claude/secrets/api-keys.env`.
3. Удали упоминание старого из memory-файлов. В `MEMORY.md` добавь запись про инцидент — урок на будущее.
4. **История git остаётся скомпрометированной.** `git filter-branch` чистит локально, но GitHub кеширует даже удалённую историю. Не надейся на 100% удаление.

---

## 6. Что не получилось / не проверено

Для честности:

- **Чистая Windows 11 VM** — не проверил. Windows Sandbox у меня конфликтует с антивирусом. Все тесты на Windows 10 Pro + реальные коммиты в продакшен-памяти.
- **WSL1** — симлинки там не пересекают границу с Windows корректно, лучше не пытаться. WSL2 работает как обычный Linux.
- **Android-клиент Claude Code** — такого нет как продукта; для телефона делается отдельный Telegram-бот на VPS, который ходит в свою копию репо. Это следующая итерация.
- **Self-hosted Gitea / Gitlab** — теоретически работает (скрипты не привязаны к GitHub, просто `git pull`), но я не тестировал.
- **2+ одновременно редактирующих машины** — конфликты разруливаются через обычный `git pull --rebase`. Не автоматизировал, потому что сложно сделать общий интерфейс для кейса «что оставить, что выбросить» когда ты правил один и тот же файл на двух машинах.

### Что наоборот получилось лучше чем ждал

- **Idempotency.** Переустанавливаю сотый раз — никаких проблем. Проверяет флагами `SYMLINK_READY`/`JunctionReady`, делает только то что не сделано.
- **Discovery активной папки.** Сначала пытался захардкодить — code review показал что это **блокер на любой чужой машине**. Переписал через `find ~/.claude/projects/*/memory` — работает везде, плюс если у тебя несколько проектов Claude — скрипт интерактивно спросит.

---

## 7. Итого и ссылки

Сухо:

- **Репо:** [github.com/DevKitRU/claude-memory-sync](https://github.com/DevKitRU/claude-memory-sync), MIT. 13 файлов, скрипты + pre-commit hook + `/setup-memory-sync` Skill + документация.
- **Стоимость:** 0 ₽ сверху того, что у тебя уже есть. Работает на любом private GitHub-репо.
- **Платформы:** macOS, Linux (проверено на Ubuntu 22.04 и 25.04), Windows 10/11 (проверено на Win10 Pro).
- **Время установки на одной машине:** 5-10 минут с нуля.

### Что в репо посмотреть отдельно

- [setup/windows.ps1](https://github.com/DevKitRU/claude-memory-sync/blob/main/setup/windows.ps1) — discovery + Junction + Task Scheduler
- [hooks/pre-commit](https://github.com/DevKitRU/claude-memory-sync/blob/main/hooks/pre-commit) — детектор токенов
- [docs/secrets.md](https://github.com/DevKitRU/claude-memory-sync/blob/main/docs/secrets.md) — безопасность развёрнуто
- [docs/architecture.md](https://github.com/DevKitRU/claude-memory-sync/blob/main/docs/architecture.md) — почему симлинк, почему Junction, почему cron
- [docs/troubleshooting.md](https://github.com/DevKitRU/claude-memory-sync/blob/main/docs/troubleshooting.md) — FAQ по граблям

### Что просит сообщество

PR приветствуются особенно по трём направлениям:

1. **Тесты на чистых VM** — Windows 11 разных редакций, macOS разных версий, WSL2, Gitea.
2. **Новые regex-паттерны в pre-commit** — если знаешь формат токена какого-то сервиса который не учтён (Yandex, Mail.ru, DigitalOcean, и т.д.) — открой issue или PR.
3. **Android/iOS клиент** — сейчас только Claude Code на десктопе. Доступ к своей памяти с телефона — через свой Telegram-бот на VPS, в следующей статье напишу как.

---

## Зачем я это сделал

Потому что каждый раз когда я садился за другую машину и Claude меня «не узнавал», я терял 5-10 минут на то чтобы скопировать ему контекст руками. За месяц это часы выкинутого времени. Плюс обидное чувство что компьютер, за который ты платишь, не работает так, как должен.

Один раз потратил день — дальше работает само. Плюс: перед публикацией я прогнал весь кит через code review (прямо через агента Claude Code) — он нашёл 5 блокеров и 7 рисков. Исправил. Без этого ревью первый же пользователь получил бы сломанный симлинк в никуда: я захардкодил путь от своей конкретной машины. Рекомендую делать code review всего open-source который собираешься выкладывать — это почти бесплатно и сильно экономит лицо.

Проект — не для денег (MIT, поддержка через issues). Хочется чтобы у коллег по этой тропе был готовый ответ на «а почему Claude меня не помнит между устройствами». Если пригодится — звёздочка на репо приятна, но issues с багами ценнее.

Если интересно — в следующей статье расскажу как к этому прикрутить Telegram-бота, чтобы писать Claude со своим контекстом прямо с телефона, пока едешь в метро. Подписывайтесь / следите.
