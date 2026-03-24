# Тестовое задание

Документация по взаимодействию с Windows API через PowerShell, собранная с помощью [Hugo](https://gohugo.io/) и темы [hugo-book](https://github.com/alex-shpak/hugo-book).

В процессе написания документации я обращался к следующим публичным источникам:

- Книга [C# in a Nutshell, Second Edition](https://www.oreilly.com/library/view/c-in-a/0596005261/ch17s08.html).
- Документация .NET: [Marshal.GetLastWin32Error](https://learn.microsoft.com/ru-ru/dotnet/api/system.runtime.interopservices.marshal.getlastwin32error?view=net-10.0), [CallingConvention Enum](https://learn.microsoft.com/ru-ru/dotnet/api/system.runtime.interopservices.callingconvention?view=net-10.0), [Interop Marshaling](https://learn.microsoft.com/en-us/dotnet/framework/interop/interop-marshalling).
- Документация Windows API: [Add-Type](https://learn.microsoft.com/ru-ru/powershell/module/microsoft.powershell.utility/add-type?view=powershell-7.5), [MessageBoxW](https://learn.microsoft.com/ru-ru/windows/win32/api/winuser/nf-winuser-messageboxw).

При форматировании текста и систематизации информации из источников я использовал [AI-агент Kiro](https://kiro.dev/).

Деплой автоматически выполняется на [GitHub Pages](https://pages.github.com/) при пуше в ветку `main`.

## Проверки качества документации

Перед деплоем в CI запускаются три проверки. Их можно выполнить локально:

### markdownlint

Проверяет стиль и форматирование Markdown-файлов согласно правилам [markdownlint](https://github.com/DavidAnson/markdownlint).

```bash
markdownlint "content/**/*.md" --config .markdownlint.json
```

Конфигурация `.markdownlint.json`:

```json
{
  "default": true,
  "MD013": false,
  "MD029": false,
  "MD060": false
}
```

Отключённые правила:

- `MD013` — ограничение длины строки (неприменимо для таблиц)
- `MD029` — строгая нумерация упорядоченных списков (конфликтует с блоками кода внутри списка)
- `MD060` — выравнивание колонок таблиц (избыточно для широких таблиц)

### textlint

Проверяет текстовое содержимое файлов: запрещает незакрытые TODO и следит за корректным написанием технических терминов.

```bash
textlint "content/**/*.md"
```

Конфигурация `.textlintrc`:

```json
{
  "rules": {
    "no-todo": true
  }
}
```

### markdown-link-check

Проверяет все внешние ссылки на доступность.

```bash
find content -name "*.md" | xargs -I {} markdown-link-check {} --config .markdown-link-check.json
```

Конфигурация `.markdown-link-check.json`:

```json
{
  "ignorePatterns": [
    { "pattern": "^#" }
  ],
  "retryOn429": true,
  "retryCount": 3,
  "fallbackRetryDelay": "30s",
  "aliveStatusCodes": [200, 206]
}
```

Якорные ссылки вида `[текст](#anchor)` исключены из проверки паттерном `^#`.

## CI/CD

Пайплайн `.github/workflows/deploy.yml` состоит из трёх последовательных джобов:

```
lint → build → deploy
```

- `lint` — запускает все три проверки выше
- `build` — собирает сайт командой `hugo --minify`
- `deploy` — публикует на GitHub Pages (только при пуше в `main`, не на PR)

Для работы деплоя необходимо в настройках репозитория выбрать:
**Settings → Pages → Source → GitHub Actions**

## Структура проекта

```
.
├── .github/workflows/deploy.yml  # CI/CD пайплайн
├── content/                      # Markdown-контент
├── themes/hugo-book/             # Тема (git submodule)
├── hugo.toml                     # Конфигурация Hugo
├── .markdownlint.json            # Правила markdownlint
├── .textlintrc                   # Правила textlint
└── .markdown-link-check.json     # Настройки проверки ссылок
```

