# Helium fork — рабочий контекст

Это **личный форк** helium (владелец — я, делаю что хочу). Этот репо `Z:\helium` —
**core**: кросс-платформенный патч-сет поверх Chromium. Платформенная упаковка/сборка
живёт в отдельных тонких репо (`helium-macos`/`-windows`/`-linux`), которые тянут этот
core, качают исходник Chromium и применяют патчи.

Chromium версия: **149.0.7827.155** (см. `chromium_version.txt`).

> Апстримный `AGENTS.md` (запрет на ИИ-контрибьюции) удалён, `CLAUDE.md` больше не
> симлинк на него, а обычный файл. Это мой форк — правила апстрима про ИИ тут не действуют.

## Цель

Реимплементировать вырезанные/недостающие фичи. Собираю под **все три десктопные
платформы: Windows, macOS, Linux** (НЕ только под одну).

Три фичи (план):
1. **Zoom bubble (окошко скейлинга Ctrl +/−)** — вырезано helium из-за «инвазивной
   структуры». НЕ скейлинг окна, а всплывашка с % и кнопкой сброса.
2. **Синк расширений + закладок на селфхосте** — синк списка расширений И закладок между
   устройствами через свой сервер (helium_services).
3. **Send-to-device** — отправка вкладки между своими устройствами через helium_services.

## Сделано в этой сессии

- **Zoom bubble (фича #1) — ГОТОВО, проверено билдом (2026-06-21).** Бабл оказался склеен
  с page-action иконкой `ZoomView` — показ бабла триггерится из самой иконки, поэтому
  «бабл без иконки» одной строкой не делается. Выбран **Вариант 1: полное удаление**
  `patches/helium/ui/remove-zoom-action.patch` (снят с диска + из `patches/series`). Зум-бабл
  (% + кнопка reset) снова работает на Ctrl +/− и Ctrl+колёсике. Компромисс: иконка зума
  снова появляется в адресной строке при зуме ≠ 100% (Вариант 2 «бабл без иконки» отложен —
  требует расцепления склеенной логики ZoomView, делать на `--dev` сборке).
- **Разведение идентификаторов форка** — чтобы стоял рядом со стоковым Helium:
  - Display-имя → **"Helium Reimplemented"**. ВАЖНО: имя в UI берётся НЕ из BRANDING
    (это только метаданные installer/.exe), а из build-time подстановки
    `utils/name_substitution_utils.py` (строка `REPLACEMENT_REGEXES_STR`), которая в build.py
    переписывает `Chrome`/`Chromium` → имя во ВСЕХ `.grd`/`.xtb`. Изменено на
    `Helium Reimplemented` (+ синхронизированы sanity-asserts в `name_substitution.py`).
    Подстановка односторонняя и без бэкапа → новое имя видно ТОЛЬКО после чистой
    распаковки Chromium + полного `build.py` (ninja-only пересборка имя не меняет).
    Также `change-chromium-branding.patch`: `PRODUCT_FULLNAME`/`PRODUCT_SHORTNAME`/installer.
  - Windows (`helium-windows/patches/helium/windows/change-branding.patch`):
    `kProductPathName="Helium Reimplemented"` (профиль уходит в
    `%LOCALAPPDATA%\imput\Helium Reimplemented\User Data`), новые `app_guid`/`active_setup`
    `{298763AD-...}`, command_execute `{F6145005-...}`, toast `0xA1B73292...`,
    elevator_clsid `0x2B6B017D...`, elevator_iid `0xDE2DE0BD...`, ProgID `HeliumReimpl*`.
- **Импорт из стокового Helium в онбординг (фича-доп)** — добавлен Helium как источник
  импорта в `patches/brave/chrome-importer-files.patch` для **win+mac+linux**
  (`GetHeliumUserDataFolder()` + `kHeliumBrowser` + запись в `DetectChromeProfiles`,
  тип `TYPE_CHROME`). ⚠️ ОГРАНИЧЕНИЕ штатного импортёра: только история + закладки +
  расширения. Пароли/настройки/cookies НЕ переносятся (это предел `ChromeImporter`).
  Пути стокового: win `%LOCALAPPDATA%\imput\Helium\User Data`,
  mac `~/Library/Application Support/Helium`, linux `~/.config/net.imput.helium`.

## Что уже выяснено по патчам (НЕ перепроверять с нуля)

- **`components/helium_services`** — встроенный селфхостед-фреймворк. Pref
  `kHeliumServicesOrigin` задаёт СВОЙ инстанс сервера (UI: "Use your own instance of
  Helium services"). `kHeliumServicesEnabled` + `kHeliumServicesConsented` — тумблер/
  consent. `GetServicesBaseURL(prefs)` резолвит базу.
- **`patches/helium/core/proxy-extension-downloads.patch`** — проксирование расширений
  уже реализовано: webstore update URL → `{origin}/ext`, сниппеты →
  `{origin}/ext/cws_snippet?id=<id>`, тумблер `kHeliumExtProxyEnabled`. `chrome.management`
  API НЕ тронут.
- **Chrome Sync НЕ вырезан**, заглушена только GAIA
  (`patches/ungoogled-chromium/disable-gaia.patch`). `//components/sync` живой. Родной синк
  не нужен — есть helium_services.
- **Импортёр данных** портирован из Brave (`patches/brave/chrome-importer-files.patch`),
  подключён в онбординг через `BraveImportDataHandler`. Источник = 3 правки (путь+имя+тип).

## Сборка

- **Все три платформы.** Кросс-компиляция между ОС невозможна (mac нужен mac SDK и т.д.).
  Патчи — просто diff-текст, пишутся/правятся на Windows.
- **Windows CI готов**: `imputnet/helium-windows` workflow `main.yml`, ~2–2.5 ч на прогон,
  до 15 стадий через артефакты, на выходе подписанный `helium*-installer.exe` как
  artifact прогона (скачивается даже при `do-release: false`).
- ВСЁ держать на диске `Z:` (исходник, билд, кэши, temp) — ничего не лить на C:.
- Платформенные репо склонированы для справки: `Z:\helium-windows`,
  `Z:\helium-reimplemented-linux`, `Z:\helium-reimplemented-macos`.

### Гочи сборки (проверено в бою)

- **Правка любого `.gn`/`.gni` → `fork-rebuild.ps1` падает на `gn gen`**: x86-тулчейн
  (`vcvarsall amd64_x86`, error 255) не поднимается из-под `Enter-VsDevShell` с `-arch=x64`.
  Фикс: один раз прогнать `gn gen` в ЧИСТОМ bash/cmd (без DevShell):
  `cd build/src/out/Default && DEPOT_TOOLS_WIN_TOOLCHAIN=0 ./gn.exe gen . --fail-on-unused-args`.
  После успешного gen `fork-rebuild` снова инкрементальный.
- **Перед пересборкой закрыть форк-браузер** (`out\Default\chrome.exe`) — он лочит
  `chrome.dll`, линковка падает `permission denied`. Стоковый Helium (`%LOCALAPPDATA%\imput\Helium`)
  можно не трогать, он держит другой файл.
- **HTML5 `pattern=` в settings-страницах теряет бэкслеши** (HTML оборачивается в JS
  template literal: `\d`→`d`, `\.`→`.`). Использовать `[0-9]` вместо `\d`, `[.]` вместо `\.`.
  Проверка в живом DOM через DevTools-консоль (рекурсивный обход shadowRoot) — быстрее, чем
  пересборка.
- **WebUI C++↔TS**: образец — `chrome/browser/ui/webui/settings/services_schema_handler.cc`
  (`RegisterMessageCallback` / `sendWithPromise` / `FireWebUIListener`). На TS-стороне
  `sendWithPromise<T>(...)` обязательно с дженериком, иначе результат `unknown` (TS2345).
  Handler уже зарегистрирован в `settings_ui.cc`; deps на factory из `//chrome/browser` в
  `//chrome/browser/ui` НЕ нужны (ui и browser слиты в один `chrome.dll`) — не добавлять, иначе
  лишний `gn gen`.
- **Онбординг (`components/helium_onboarding`) собирается vite ТОЛЬКО при полном `build.py`
  (release).** `fork-rebuild` (ninja chrome) и `--dev` его НЕ пересобирают → правки
  Svelte/strings НЕ видны до полной пересборки (типичная ловушка «добавил тумблер, а его
  нет»). `src/lib/strings.ts` генерится из `helium_onboarding_strings.grdp` скриптом
  `i18n` на `prebuild` → править `.grdp` (и `.ts` перегенерится) либо оба синхронно.
- **Перегенерация патча новых файлов (`--- /dev/null`)**: секцию файла заменять ЦЕЛИКОМ
  (содержимое с диска, каждая строка с `+`), хедер хунка `@@ -0,0 +1,N @@`, N = число строк.
  При склейке секций НЕ потерять строку `--- /dev/null` перед `+++ b/...`, иначе git ругнётся
  `corrupt patch`. Достоверная проверка патча — НЕ `git apply --reverse` против живого дерева
  (врёт из-за сдвигов соседних патчей), а прогон всей серии до проверяемого патча в отдельном
  дереве каталогов (`mkdir t && cd t; for p in <series до патча>; do patch -p1 --forward -i $p`),
  затем forward + сравнение с диском (нормализуй CRLF: `diff <(tr -d '\r' a) <(tr -d '\r' b)`).
- **Тосты (`chrome/browser/ui/toasts`)**: `ToastId` — закрытый enum, `GetToastName` в
  `toast_id.cc` — switch с `NOTREACHED()` в конце. Свой тост = правки `toast_id.{h,cc}` +
  регистрация в `toast_service.cc`; ОБЯЗАТЕЛЬНО добавить `case` в `GetToastName`, иначе крэш
  при показе. Action-toast ОБЯЗАН иметь close button (`CHECK` в `ValidateSpecification`).
  Размер/отступы кнопок — общий `toast_view.cc` для всех тостов, точечно под один тост не
  меняется без риска для остальных.

### Локальный цикл (Windows, fork-скрипты закоммичены в helium-windows)

Скрипты теперь в репо `helium-windows` (на ремоуте), без хардкода путей — корень берётся из
расположения скрипта, машинно-специфика через env (см. README «Fork dev scripts»):
`FORK_VS_PATH` (путь VS, иначе автодетект через `vswhere`), `FORK_VS_YEAR` (год VS, иначе
автодетект), `FORK_TMP` (scratch/temp, иначе системный `%TEMP%`). На МОЕЙ машине задаю
`FORK_VS_PATH=Z:\Visual Studio` (VS в нестандартном пути) и `FORK_TMP=Z:\tmp` (правило «всё на Z:»).

Общее окружение вынесено в **`fork-env.ps1`** (temp + детект VS + `Enter-VsDevShell` с
`DEPOT_TOOLS_WIN_TOOLCHAIN=0`); `fork-build.ps1`/`fork-rebuild.ps1` его дот-сорсят. Нестандартный
путь VS прокидывается в gn через `vs{YEAR}_install` (ставится автоматически из `FORK_VS_PATH`+год).

**Release и Dev разведены по разным out-папкам — переключение режима БОЛЬШЕ НЕ затирает
другую сборку.** `build.py` параметризован флагом `--out-dir` (дефолт `out/Default`, чтобы
CI/штатное поведение не менялось). Оба PS-скрипта принимают `-Dev`:
- **без `-Dev`** → RELEASE: `out\Default`, `is_official_build=true` (текущая раздаваемая сборка).
- **с `-Dev`** → DEV: `out\Dev`, `is_component_build=true` (быстрые инкрементальные пересборки).

- **`fork-build.ps1 [-Dev]` — холодная/полная сборка.** Гоняет `python3 build.py`
  (скачивание Chromium → патчи → gn gen → ninja). `-Dev` добавляет `--dev --out-dir out/Dev`.
  ВАЖНО: в `--dev` build.py патчи НЕ накладывает сам — встаёт на `Apply patches using quilt,
  then press Enter`, патчи накатываешь quilt'ом вручную (два прохода: сначала
  `helium-chromium/patches`, потом `patches`). Также в `--dev` НЕ прогоняется
  name/domain-substitution и i18n → в браузере имя остаётся `Chromium`, строки английские.
  В RELEASE (official) — всё применяется, имя `Helium Reimplemented`, переводы работают.
- **`fork-rebuild.ps1 [-Dev]` — инкрементальная пересборка.** Запускает ninja напрямую,
  минуя build.py (НЕ перепатчивает дерево): `ninja -C <out> chrome chromedriver setup
  mini_installer`. `-Dev` → `out\Dev`, иначе `out\Default`. Пересобирает только изменённые
  файлы + линковка → минуты. Использовать когда правишь исходник в `build\src` руками или
  после мелкой правки патча.
- **`fork-sync.sh [--rebuild|-r] [--dev] [--dry-run|-n] [--no-pull]` (git-bash)** — синк дельты
  с ремоута БЕЗ полного ребилда. Тянет новые коммиты core+platform, накатывает на уже
  распакованное `build/src` ТОЛЬКО дельту изменённых `.patch` (реверс старой версии патча →
  форвард новой), дальше быстрый `fork-rebuild.ps1` (или сразу `-r`). Консервативен: бэкап
  всех затронутых файлов, при любом нечистом apply — restore + «гони полный fork-build.ps1».
  Отказывается от дельты, если изменился `series`/`.gn`/`downloads.ini`/version-файлы (нужен
  полный билд). `--dry-run`/`-n` делает `fetch` (НЕ pull, дерево/HEAD не трогает) и показывает
  входящую дельту. Baseline — `build/.fork-sync-marker` (gitignored, машинный), пишется
  автоматически успешным `fork-build.ps1` → ручной init НЕ нужен.
- **Installer из готового дерева**: `package.py` собирает раздаваемый
  `helium*-installer.exe` из official-билда (`out\Default`).
- **`fork-wipe-profile.bat`** — сносит ТОЛЬКО профиль форка
  (`%LOCALAPPDATA%\imput\Helium Reimplemented\User Data`) для чистого теста онбординга.
  Стоковый `imput\Helium` не трогает; отказывается работать, если форк-браузер запущен.
- **Где искать вывод**: release — `build\src\out\Default\chrome.exe`, dev —
  `build\src\out\Dev\chrome.exe`. Installer — рядом после `package.py`.

## Статус фич

Все три фичи ГОТОВЫ и в проде (свой инстанс helium-services). Детали реализации — в патчах,
здесь только карта.

1. **Фича #1 (zoom bubble)** — готово (см. выше в «Сделано»).
2. **Фича #2 (синк расширений + закладок)** — готово. Сервер `svc/sync` в `Z:\helium-services`
   (Deno KV, `/sync/{extensions,bookmarks}`, Bearer-токен, optimistic version). Клиент — патчи
   `sync-prefs-and-url`, `sync-client-engine`, `sync-settings-ui`, `services-prefs`,
   `sync-missing-extensions`. `HeliumSyncService` (KeyedService, eager). Tombstone-3-way-merge
   с baseline-pref, корзина удалённых закладок (восстановление/очистка в Settings). Триггеры
   синка: смена токена/тумблеров + live (debounce 4с по observer закладок/расширений) +
   **стартовый pull (~10с после запуска) + периодический (5 мин)** — без них устройство B не
   подтягивало чужие изменения. Закладки additive со структурой; расширения НЕ ставятся
   автоматом — отсутствующие показываются списком в Settings со ссылкой на webstore
   (`missing_extensions_` → `getHeliumSyncStatus`). Кнопка Reset server data убрана из UI
   (метод в движке оставлен).
   - **http://localhost разрешён в любом билде**: UI-паттерн origin = `https://.*` ИЛИ
     `http://localhost|127.0.0.1`; C++ (`helium_services_helpers.cc`) пускает http только на
     localhost (`SchemeIs(https) || IsLocalhost`).
3. **Фича #3 (send-to-device)** — готово. Long-poll (НЕ minipush): per-device inbox
   `{origin}/sync/sendtab/<id>`, сервер держит GET открытым через Deno `kv.watch`, **дренит
   инбокс на чтение** (каждый таб доставляется ровно раз — иначе был луп/пачка уведомлений).
   Патчи `sendtab-engine`, `sendtab-context-menu` (ПКМ по вкладке → «Send tab to my devices»),
   `sendtab-settings`, `sendtab-toast`. Приём: in-window toast (`ToastId::kHeliumSendTabReceived`)
   с fallback на OS-уведомление, если нет окна; или авто-открытие (pref `kHeliumSendTabReceiveMode`).
   Send-tab включается по дефолту при включении синка (one-way nudge).

Онбординг-тумблеры sync/sendtab + иконка стокового Helium в импорте — есть в коде/патчах,
видны ТОЛЬКО после полного `build.py` (онбординг собирается vite, см. гочу выше).

Бэклог: zoom bubble Вариант 2 (бабл без иконки) — на `--dev` сборке.

> Этот CLAUDE.md — рабочий брифинг форка, не для коммита в upstream.
