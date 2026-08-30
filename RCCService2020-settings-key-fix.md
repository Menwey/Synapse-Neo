# RCCService 2020 — «Settings key must be defined»

Разбор и починка ошибки при запуске `RCCService2020\RCCService.exe`.

- **Бинарь:** `D:\important\RCCService\RCCService2020\RCCService.exe` (PE32, x86, ImageBase `0x400000`)
- **Симптом:** при старте процесс падает с сообщением `Settings key must be defined`
- **Итог:** в реестре отсутствует value `Settings202` (сборка пропатчена, оригинальное имя было `SettingsKey`)

---

## 1. Причина

Бинарь — пропатченная сборка RCC. В оригинале ключ настроек читается из реестра под именем `SettingsKey`, а здесь имя строки заменено на `Settings202`. Оба имени ровно 11 байт, поэтому патч влез in-place, без пересборки экзешника:

```
0139a474  41 63 63 65 73 73 4b 65 79 00 00 00 53 65 74 74  AccessKey...Sett
0139a484  69 6e 67 73 32 30 32 00 52 65 61 64 20 73 65 74  ings202.Read set
0139a494  74 69 6e 67 73 20 6b 65 79 3a 20 25 73 00 00 00  tings key: %s...
```

Именно поэтому гайды, где написано создать `SettingsKey`, для этой сборки не работают.

Для сравнения: в `RCCService2017\RCCService.exe` и `RCCService2018\RCCService.exe` строка `Settings202` отсутствует, там штатный `SettingsKey`, и строки `Settings key must be defined` в них нет вообще.

### Чтение ключа — `sub_53EF60`

```asm
0053EFA7  push 0x179b850    ; "Software\ROBLOX Corporation\Roblox\"
0053EFB0  push 0x80000002   ; HKEY_LOCAL_MACHINE
0053EFB4  call RegOpenKeyExA         ; 0x20019 = KEY_READ, без флагов KEY_WOW64_*
0053EFCF  push 0x179b880    ; "Settings202"   <-- имя value
0053EFD7  call RegQueryValueExA
0053F020  push "Read settings key: %s"
```

Флаги `KEY_WOW64_*` не выставляются, процесс 32-битный → Windows перенаправляет чтение в `HKLM\SOFTWARE\WOW6432Node\...`.

### Проверка и бросок исключения — `sub_53F530`

Функция инициализации настроек, вызывается из main по `0x5408B9`:

```asm
0053F707  call 0x53ef60                ; чтение ключа из реестра
0053F80D  call 0x53eed0                ; парсинг аргумента -SettingsFile
...
0053F827  lea  ecx, [ebp - 0x1b0]      ; settingsKey (std::string)
0053F82D  call 0x53dc40                ; return (size == 0)
0053F837  je   0x53f863                ; не пусто -> продолжаем
0053F839  movzx edx, byte [ebp-0x11c]  ; флаг "-SettingsFile передан"
0053F842  jne  0x53f863                ; передан -> продолжаем
0053F844  push 0x179bae8               ; "Settings key must be defined"
0053F85E  call 0x15f85e5               ; _CxxThrowException -> процесс умирает
```

В C-подобном виде:

```c
if (settingsKey.empty() && !settingsFileProvided)
    throw "Settings key must be defined";
```

### Что было в реестре

```
HKLM\SOFTWARE\WOW6432Node\ROBLOX Corporation\Roblox
    AccessKey   = KoroneRcc-dev-secret
    SettingsKey = RCCService2020        <-- читается? нет
```

`Settings202` отсутствует → `settingsKey` пустая → исключение.

`AccessKey` при этом читается штатно, отдельной функцией `sub_53ED70` — её не патчили.

---

## 2. Починка

### Шаг 1 — добавить value в реестр

cmd **от администратора**:

```cmd
reg add "HKLM\SOFTWARE\WOW6432Node\ROBLOX Corporation\Roblox" /v Settings202 /t REG_SZ /d RCCService2020 /f
```

- ветка именно `WOW6432Node` — см. разбор `sub_53EF60` выше;
- значение `RCCService2020` — это applicationName, под которым RCC потом запрашивает свои флаги;
- `SettingsKey` и `AccessKey` не удалять: `AccessKey` реально используется, `SettingsKey` просто останется мёртвым.

**Альтернатива без админа.** Парсер `sub_53EED0` понимает аргумент `-SettingsFile <путь>` (регистронезависимо) и выставляет флаг, который тоже гасит проверку. Дописать в каждый `.bat`:

```cmd
RCCService.exe -console -verbose -port 1621 -SettingsFile settings.json
```

### Шаг 2 — локальные ClientSettings (необязательно)

После прохождения проверки RCC идёт за флагами (`sub_86CE40`):

```
https://clientsettingscdn.<домен>/v2/settings/application/RCCService2020
```

`BaseUrl` в `AppSettings.xml` = `http://www.pekora.zip`, этот CDN недоступен. Дальше `sub_8FC210` → `sub_8FD0E0` пробуют локальный файл `<ключ>.json` с секцией `ClientSettings` внутри. Если и его нет — в логе будет:

```
LoadClientSettingsFromLocal: Couldn't fetch any data from local
```

Это **не** блокер: крэш был бы только при включённом флаге `DebugCrashOnFailToLoadClientSettings`. Но чтобы лог был чище и флаги не оставались дефолтными, создать рядом с `RCCService.exe` файл `RCCService2020.json` (имя = значение ключа + `.json`):

```json
{ "ClientSettings": {} }
```

### Шаг 3 — запуск

```cmd
cd /d <путь>\RCCService2020
RCCService.exe -console -verbose -port 1621
```

Признак успеха — строка `Service started on port 1621` (из `sub_573BE0`, печатается после успешного `bind`). Если порт занят, там же вылетит исключение.

Когда один инстанс взлетел, поднимать остальные через `start.bat` — он стартует четыре процесса (Player / Image / Game / Catalog Render), каждый в цикле с авто-рестартом.

---

## 3. Диагностика, если снова упадёт

Порядок инициализации в `sub_53F530`:

1. базовый URL из `AppSettings.xml` (`sub_53EA40`, ищет тег `BaseUrl`, логирует `Got Base url: %s`)
2. machine address / hostname (`sub_53ED20`, `sub_53EE60`)
3. `AccessKey` из реестра (`sub_53ED70`)
4. settings key из реестра (`sub_53EF60`, логирует `Read settings key: %s`)
5. проверка `Roblox.Thumbnails.Relay`
6. старт SOAP-сервиса (`sub_573BE0`, `Service starting` → `Service started on port %d`)

По тому, на какой строке обрывается вывод `-console -verbose`, видно, какой шаг сломался.

---

## Приложение: методика реверса

IDA Professional 9.0 установлена в `C:\Program Files\IDA Professional 9.0`, но headless-режим не запускается:

```
Could not acquire license: File not found: ida*.hexlic
```

Лицензия лежит в `C:\Users\user\Desktop\ida-pro-9.0-crack-windows\licence\ida.hexlic` и `D:\1df\idapro.hexlic`, но не скопирована в каталог IDA. Чтобы пользоваться `idat64.exe -A -S<script>`, файл нужно положить рядом с `ida64.exe`.

Вместо IDA использованы `capstone` + `pefile` (Python 3.14). Схема работы:

1. вытащить строки из `.rdata`/`.data` с их виртуальными адресами;
2. найти ссылки на строку поиском 4-байтового little-endian VA внутри `.text` (x86 без RIP-relative — адрес лежит в `push` как immediate);
3. найти начало функции сканированием назад до пролога `55 8B EC`, отсечённого `CC`/`C3`, с проверкой линейным дизассемблированием;
4. найти вызывающих — брутфорс `E8 rel32` по всей `.text` с пересчётом цели.
