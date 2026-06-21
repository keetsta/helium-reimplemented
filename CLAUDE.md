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
- Платформенные репо склонированы для справки: `Z:\helium-windows`, `Z:\helium-linux`.

### Локальный цикл (Windows, два PS-скрипта в `Z:\helium-windows`)

Окружение у обоих скриптов одинаковое: `TMP/TEMP → Z:\tmp`, `vs2026_install=Z:\Visual Studio`,
`DEPOT_TOOLS_WIN_TOOLCHAIN=0`, затем `Enter-VsDevShell` (VS стоит в нестандартном `Z:\Visual Studio`).

- **`fork-build.ps1` — холодная/полная сборка.** Гоняет `python3 build.py` (скачивание
  Chromium → применение патчей → gn gen → ninja). Для быстрой итерации фич запускать с
  `--dev` (component-build, быстрые инкрементальные пересборки). Текущее дерево собрано в
  **official** (`is_official_build=true`, `is_component_build=false`) — годится для installer,
  но НЕ для быстрых пересборок.
- **`fork-rebuild.ps1` — инкрементальная пересборка.** Запускает ninja напрямую, минуя
  build.py (НЕ перепатчивает уже готовое дерево):
  `third_party\ninja\ninja.exe -C out\Default chrome chromedriver setup mini_installer`.
  Пересобирает только изменённые файлы + линковка → минуты. Использовать когда правишь
  исходник в `build\src` руками или после мелкой правки патча.
- **Installer из готового дерева**: `package.py` собирает раздаваемый
  `helium*-installer.exe` из текущего official-билда.
- **Где искать вывод**: `Z:\helium-windows\build\src\out\Default\chrome.exe` (бинарь),
  installer — рядом после `package.py`.

## Открытые следующие шаги

1. **Фича #2 (синк расширений + закладок)** — в работе. Разобрать контракт
   `helium-services` `/ext` (репо `imputnet/helium-services`), спроектировать синк закладок
   поверх того же канала.
2. Фича #3 (send-to-device): спроектировать поверх helium_services-канала.
3. Бэклог: иконка стокового Helium в списке импорта онбординга (Svelte); Вариант 2
   zoom bubble (бабл без иконки) — на `--dev` сборке.

> Этот CLAUDE.md — рабочий брифинг форка, не для коммита в upstream.
