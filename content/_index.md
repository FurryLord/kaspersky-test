# Взаимодействие с Windows API с помощью PowerShell

## 1. Введение {#introduction}

[Содержание раздела]

---

## 2. Архитектура взаимодействия {#architecture}

### 2.1. Принцип работы P/Invoke {#pinvoke-principle}

[Содержание раздела]

### 2.2. Жизненный цикл P/Invoke вызова {#pinvoke-lifecycle}

[Содержание раздела]

### 2.3. Командлет `Add-Type` {#add-type}

[Содержание раздела]

### 2.4. Альтернативные способы взаимодействия {#alternatives}

[Содержание раздела]

### 2.5. Ограничения {#limitations}

[Содержание раздела]

---

## 3. Предварительные требования {#prerequisites}

[Содержание раздела]

---

## 4. Базовый синтаксис импорта API {#import-syntax}

Для вызова функций из нативных библиотек Windows (`kernel32.dll`, `user32.dll` и других) в PowerShell используется механизм [P/Invoke](#pinvoke-principle).
Это стандартный способ взаимодействия управляемого кода .NET с неуправляемыми функциями Windows API.  
Подробнее о принципах работы и отличиях в разных версиях PowerShell см. в разделе [Архитектура взаимодействия](#architecture).

Для реализации P/Invoke применяется атрибут `DllImport`.
Он указывает среде выполнения, из какой библиотеки и какую функцию следует вызвать.  
Описание параметров этого атрибута см. в разделе [Атрибут DllImport](#dllimport-attribute).

Командлет `Add-Type` предназначен для использования `DllImport` в PowerShell.
Он компилирует фрагмент кода на C#, содержащий этот атрибут, и загружает полученный тип в текущую сессию.  
Типовой сценарий применения см. в разделе [Шаблон импорта с помощью Add-Type](#add-type-template).

### 4.1. Атрибут `DllImport` {#dllimport-attribute}

Атрибут `DllImport` — ключевой элемент механизма [Platform Invoke](#pinvoke-principle).
Он указывает среде CLR (Common Language Runtime), какую неуправляемую библиотеку (DLL) загрузить и какую функцию в ней вызвать.
Атрибут применяется к статическому внешнему методу и содержит всю информацию, необходимую для [маршаллинга данных](#marshalling).

Атрибут имеет следующие параметры:

| Параметр                  | Описание                                                                                                                                                                                                         | Значение по умолчанию      |
| ---------------------------| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| ----------------------------|
| `dll-name` (обязательный) | Имя DLL-файла, из которого импортируется функция                                                                                                                                                                 | —                          |
| `EntryPoint`              | Имя функции в библиотеке. Указывается, если имя метода в C# отличается от имени экспортируемой функции                                                                                                           | —                          |
| `CharSet`                 | Определяет кодировку строковых параметров. Доступные значения: `Auto`, `Unicode`, `Ansi`                                                                                                                         | `CharSet.Auto`             |
| `SetLastError`            | Указывает, сохраняет ли вызываемая функция код ошибки. При значении `true` код ошибки можно получить через `Marshal.GetLastWin32Error()`. См. [Получение кода ошибок](#error-codes)                              | `false`                    |
| `ExactSpelling`           | Определяет, должно ли имя из `EntryPoint` точно совпадать с именем функции в библиотеке. При `false` среда выполняет поиск с учётом вариантов написания (например, `MessageBoxA` / `MessageBoxW`)                | `false`                    |
| `PreserveSig`             | Управляет обработкой `HRESULT`. При `true` возвращаемое значение передаётся как `int`. При `false` неудачные `HRESULT` автоматически преобразуются в исключения. См. [Обработка исключений](#exception-handling) | `true`                     |
| `CallingConvention`       | Определяет порядок передачи аргументов и очистки стека                                                                                                                                                           | `CallingConvention.Winapi` |

Значения `CallingConvention`:

| Значение   | Описание                                                                                                                                                                                                         |
| ------------| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Winapi`   | Фактически не определяет конкретное соглашение вызова, а использует соглашение, принятое на платформе по умолчанию                                                                                               |
| `Cdecl`    | Вызывающий код очищает стек. Это позволяет вызывать функции с переменным числом аргументов (varargs), что делает его подходящим для методов, принимающих переменное количество параметров (например, `printf`)   |
| `StdCall`  | Вызываемая функция очищает стек                                                                                                                                                                                  |
| `ThisCall` | Первый параметр — указатель `this`, который передается через регистр `ECX`. Остальные параметры помещаются в стек. Это соглашение используется для вызова методов классов, экспортированных из неуправляемой DLL |
| `FastCall` | Это соглашение вызова не поддерживается                                                                                                                                                                          |

Пример атрибута c заданными параметрами:

```csharp
[ ("user32.dll",
    EntryPoint = "MessageBoxW",
    CharSet = CharSet.Unicode,
    SetLastError = true,
    ExactSpelling = true,
    PreserveSig = true,
    CallingConvention = CallingConvention.Winapi)]
```

### 4.2. Шаблон импорта c помощью Add-Type {#add-type-template}

In PowerShell, Windows API functions are imported using the `Add-Type` cmdlet.
This cmdlet compiles C# code at runtime and loads the resulting type into the current session.

The example below demonstrates how to import the `MessageBoxW` function using the `DllImport` attribute with the parameters described in the [previous subsection](#dllimport-attribute).

`MessageBoxW` is a Windows API function that displays a modal dialog box with a message and buttons, returning a value that indicates which button the user selected.

Step-by-step example:

1. Define the function signature for `MessageBoxW` according to the [Windows API documentation](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-messageboxw):

  ```powershell
  $signature = @'
  [DllImport("user32.dll", EntryPoint = "MessageBoxW", CharSet = CharSet.Unicode, SetLastError = true, ExactSpelling = true, PreserveSig = true, CallingConvention = CallingConvention.Winapi)]
  // Parameters:
  //   hWnd      - handle to the parent window (0 = no parent)
  //   lpText    - message text
  //   lpCaption - window title
  //   uType     - type of buttons and icons
  public static extern int MessageBox(IntPtr hWnd, string lpText, string lpCaption, uint uType);
  '@
  ```
  
  **Important**: The modifiers `public`, `static`, and `extern` are required for P/Invoke to work correctly in PowerShell:
  
- `public` — makes the method accessible from PowerShell;
- `static` — allows calling the method without instantiating the class;
- `extern` — indicates that the method implementation resides in a native DLL.
  
  Omitting any of these modifiers prevents the Windows API function from being properly imported and invoked.

2. Import the API using `Add-Type`:

  ```powershell
  $type = Add-Type -MemberDefinition $signature -Name "User32API" -Namespace "Win32" -PassThru
  ```
  
  For the list of `Add-Type` parameters, refer to the subsection [Add-Type Parameters](#add-type-parameters).

3. Invoke the imported function:

  ```powershell
  $result = $type::MessageBox([IntPtr]::Zero, "Hello World!", "Windows API", 0)
  ```

4. If the import was successful, a dialog box appears:

  ![Click the "OK" button](./hello_world.png)

  Click **OK**. The function returns a value corresponding to the pressed button.

5. Display the result:

  ```powershell
  Write-Host "Function call result: $result"
  ```

### 4.3. Параметры Add-Type {#add-type-parameters}

When importing Windows API functions, the most commonly used `Add-Type` parameters are listed below:

| Parameter                      | Purpose                                                                                | Example Value                      |
| --------------------------------| ----------------------------------------------------------------------------------------| ------------------------------------|
| `-MemberDefinition` (required) | Contains the C# code with the `[DllImport]` attribute and method signature             | `$signature` (here-string)         |
| `-Name` (required)             | Specifies the name of the generated type or class                                      | `"Win32Utils"`, `"NativeMethods"`  |
| `-Namespace`                   | Organizes the type within a namespace to avoid naming conflicts                        | `"Win32"`, `"PInvoke"`, `"Native"` |
| `-PassThru`                    | Returns the created type object, allowing immediate assignment to a variable           | `$type = Add-Type ... -PassThru`   |
| `-ReferencedAssemblies`        | Lists additional .NET assemblies required by the code (for example, Windows Forms)     | `"System.Windows.Forms"`           |
| `-IgnoreLastError`             | Disables automatic capturing of `GetLastError()` — not recommended for Win32 API calls | —                                  |

Examples of their usage are listed below:

#### `MemberDefinition`

This parameter accepts the actual C# declaration of the imported function(s).
It is typically provided as a here-string for better readability:

```powershell
# Define the function signature
$signature = @'
[DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
public static extern int MessageBoxW(IntPtr hWnd, string lpText, string lpCaption, uint uType);
'@

# Pass the signature using -MemberDefinition
$type = Add-Type -MemberDefinition $signature -Name "User32Methods" -PassThru

# Call the imported function
$result = $type::MessageBoxW([IntPtr]::Zero, "Test", "PowerShell", 0)
```

#### `Name`

....

---

## 5. Маршаллинг данных {#marshalling}

### 5.1. Общие принципы {#marshalling-principles}

[Содержание раздела]

### 5.2. Простые типы {#simple-types}

[Содержание раздела]

### 5.3. Строковые типы {#string-types}

[Содержание раздела]

### 5.4. Указатели {#pointers}

[Содержание раздела]

### 5.5. Структуры {#structures}

[Содержание раздела]

### 5.6. Массивы {#arrays}

[Содержание раздела]

### 5.7. Callback-функции (делегаты) {#callbacks}

[Содержание раздела]

### 5.8. Оптимизация маршаллинга {#marshalling-optimization}

[Содержание раздела]

---

## 6. Отладка и диагностика {#debugging}

### 6.1. Программная отладка {#software-debugging}

#### 6.1.1. Получение кода ошибок {#error-codes}

[Содержание раздела]

#### 6.1.2. Обработка исключений {#exception-handling}

[Содержание раздела]

### 6.2. Инструментальная отладка {#instrumental-debugging}

#### 6.2.1. Проверка экспортов DLL {#dll-exports}

[Содержание раздела]

#### 6.2.2. Проверка размеров структур {#structure-size}

[Содержание раздела]

---

## 7. Практические примеры {#examples}

### 7.1. Окна {#windows-examples}

[Содержание раздела]

### 7.2. Процессы и память {#process-memory-examples}

[Содержание раздела]

### 7.3. Системная информация {#system-info-examples}

[Содержание раздела]

### 7.4. Управление питанием {#power-management-examples}

[Содержание раздела]

---

## 8. Лучшие практики {#best-practices}

### 8.1. Создание модуля-обертки {#wrapper-module}

[Содержание раздела]

### 8.2. Кэширование типов {#type-caching}

[Содержание раздела]

### 8.3. Улучшение производительности {#performance}

[Содержание раздела]

---

## 9. Безопасность {#security}

### 9.1. Политики выполнения скриптов {#execution-policies}

[Содержание раздела]

### 9.2. Конфиденциальность данных {#data-privacy}

[Содержание раздела]

---

## 10. Распространённые ошибки {#common-errors}

### 10.1. EntryPointNotFound {#entrypointnotfound}

[Содержание раздела]

### 10.2. DllNotFound {#dllnotfound}

[Содержание раздела]

### 10.3. AccessViolation {#accessviolation}

[Содержание раздела]

---

## 11. Полезные ссылки {#links}

[Содержание раздела]
