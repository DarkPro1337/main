---
title: "WASM-моддинг для ваших игр на Godot .NET"
summary: "Откройте для себя безопасный, изолированный WASM-моддинг для вашей игры на Godot .NET с использованием Wasmtime и AssemblyScript. Создайте собственный API для модов, который исключает риски, связанные с внедрением DLL."
date: 2025-02-13T04:48:34+04:00
draft: false
tags: ["wasm", "godot", "modding", "devlog"]
thumbnail: "tn.png"
images:
- img/post.ru.png
---
![WASM-моддинг для ваших игр на Godot .NET](img/post.ru.png)

При разработке **Arcomage** с использованием **Godot .NET** я столкнулся с проблемой: как добавить поддержку моддинга на основе скриптов в игру? Проблема заключалась в том, что я не хотел разрешать прямую инъекцию DLL — это крайне небезопасно и подвержено злоупотреблениям, позволяющим внедрять вредоносный код, который может скомпрометировать системы пользователей или украсть конфиденциальные данные, такие как пароли.

Вместо этого я выбрал подход с использованием песочницы, при котором моды выполняются безопасно и не представляют угрозы для конечных пользователей. Моя цель состояла в создании единого API для всех модов, чтобы сделать их не только безопаснее, но и проще в написании. Именно тогда я наткнулся на видео от [безымянного разработчика](https://youtu.be/kcWVYeaFmqQ), в котором чётко и понятно объясняется, как реализовать поддержку модов на WASM. Огромное спасибо автору за его идеи!

Я уже знал о WebAssembly, но никогда не рассматривал его как песочницу для запуска скриптовых модов в игре. Было удивительно узнать, что Microsoft Flight Simulator 2020 использует WebAssembly для скриптов модов — и это действительно круто!

Итак, я решил попробовать нечто подобное для Arcomage. В этой статье я поделюсь своими находками и идеями по реализации WASM-моддинга в Godot .NET.

## Что такое WebAssembly и как им пользоваться?

**WASM** — это формат байткода, который выполняется в виртуальной машине. В отличие от CIL .NET, WASM предназначен для исполнения в браузерах или в любых приложениях, которые его поддерживают. В нашем случае мы используем версию Godot с поддержкой .NET. Для .NET существует несколько встроенных WASM-рантаймов. В моем проекте я использовал [Wasmtime](https://wasmtime.dev/) от Bytecode Alliance.

## Пример: Создание WASM-мода

### Настройка

Сначала добавьте пакет Wasmtime через NuGet в ваш проект Godot .NET:
```bash
dotnet add package wasmtime
```

Либо добавьте следующую ссылку на пакет в ваш файл `.csproj`:
```xml
<PackageReference Include="Wasmtime" Version="22.0.0" />
```

Не забудьте выполнить `dotnet restore` после внесения изменений в файл проекта.

Чтобы компилировать файлы **WASM** из **AssemblyScript**, установите **Node.js** (в комплекте с **npm**). Вы можете скачать его с [официального сайта Node.js](https://nodejs.org/en). Проверьте установку с помощью команд:
```bash
node -v
npm -v
```

В этом примере мы будем компилировать WASM с помощью **AssemblyScript**. Если вы предпочитаете другой язык (например, **Rust** с **wasm-pack**), ознакомьтесь с его документацией. Также настоятельно рекомендую изучить [документацию компилятора AssemblyScript](https://www.assemblyscript.org/compiler.html).

После установки Node.js выполните:
```bash
npm install -g assemblyscript
```

Эта команда устанавливает компилятор AssemblyScript глобально (либо локально, если предпочитаете).

После установки необходимых инструментов, пора писать код!

### WASM

Выберите рабочую директорию и создайте два файла: `env.ts` и `mod.ts`.

В файле `env.ts` определите экспортируемые функции, которые будут доступны вашему WASM-моду:
```typescript
@external("env", "host_log")
declare function host_log(ptr: i32, len: i32): void;

export function log(message: string): void {
  const encoded = String.UTF8.encode(message);
  host_log(changetype<i32>(encoded), encoded.byteLength);
}
```

В этом примере функция `log` отправляет сообщение хосту через `host_log`, которая принимает указатель на строку, закодированную в UTF-8, и её длину.

В файле `mod.ts` напишите код, который будет выполняться в WASM-моде:
```typescript
import { log } from "./env";

export function init(): void {
  log("Hello, World!");
}
```

Здесь экспортированная функция `init` будет вызвана при инициализации мода, что вызовет отправку лог-сообщения. На стороне Godot вы обработаете это (например, выведя сообщение в консоль с помощью `GD.Print`).

Эта простая настройка иллюстрирует концепцию. Как разработчик, поделитесь файлом `env.ts` с мододелами, чтобы они могли использовать определённые функции. В этой схеме `mod.ts` служит точкой входа для мода, где вы указываете функцию входа (в данном случае, `init`), которая автоматически вызывается игрой при загрузке WASM-файла.

Чтобы скомпилировать ваш WASM-мод, выполните:
```bash
asc .\mod.ts -o mod.wasm
```

Если AssemblyScript не установлен глобально, убедитесь, что вы указали корректный путь. Выполнение этой команды создаёт файл `mod.wasm` — точку входа для вашего мода.

### Интеграция с Godot .NET

Поместите скомпилированный WASM-файл в директорию `user` вашего проекта Godot (например, `user://mod.wasm`).

Рекомендую использовать пользовательскую директорию, включив её с помощью флага `application/config/use_custom_user_dir` в файле `project.godot`. Также задайте имя папки через `application/config/custom_user_dir_name`.

Создайте новый C# класс, наследующий от `Node` — в моем примере он называется `WasmLoader`:
```csharp
using Godot;
using System.Text;
using Wasmtime;
using Engine = Wasmtime.Engine;

public partial class WasmLoader : Node
{
   public override void _Ready()
   {
      byte[] wasmBytes;
      using (var file = FileAccess.Open("user://mod.wasm", FileAccess.ModeFlags.Read))
         wasmBytes = file.GetBuffer((int)file.GetLength());

      using var engine = new Engine();
      using var module = Module.FromBytes(engine, "mod", wasmBytes);
      using var store = new Store(engine);
      using var linker = new Linker(engine);

      linker.Define("env", "host_log", Function.FromCallback(store, (Caller caller, int ptr, int len) =>
      {
         var memory = caller.GetMemory("memory");
         if (memory is null)
            return;
         var span = memory.GetSpan<byte>(0);
         var message = Encoding.UTF8.GetString(span.Slice(ptr, len).ToArray());
         GD.Print(message);
      }));

      linker.Define("env", "abort", Function.FromCallback(store, (int msg, int file, int line, int column) =>
      {
         GD.Print($"Abort called at {file}:{line}:{column}");
      }));

      var instance = linker.Instantiate(store, module);
      instance.GetAction("init")?.Invoke();
   }
}
```

Разберем, что происходит в этом коде:
1. WASM-файл загружается из `user://mod.wasm` и его байты передаются в `Module.FromBytes`.
2. Создаются `Engine`, `Module`, `Store` и `Linker` для выполнения WASM-мода.
3. Определяются функции, экспортируемые модом. Здесь `host_log` принимает указатель и длину, декодирует строку из памяти и выводит её.
4. Также определяется функция `abort`; она вызывается при ошибках внутри WASM-мода. Хотя она может и не понадобиться, компилятор включает её по умолчанию. Если не определить её, вы получите исключение `Wasmtime.WasmtimeException` из-за неопределённого импорта.
5. Наконец, WASM-мод инициализируется, и вызывается его функция `init`, которая выводит "Hello, World!" в консоль.

После создания класса `WasmLoader` добавьте его в сцену и запустите игру. Если всё настроено правильно, вы увидите "Hello, World!" в консоли. Либо вы можете добавить загрузчик в `Autoload` вашего проекта, чтобы он запускался при старте.

## Следующие шаги

Вот несколько идей для расширения вашей системы моддинга:

1. **Передача обратных вызовов процесса:**  
   Передавайте обратные вызовы `_Process` и `_PhysicsProcess` в WASM-мод (не забудьте включить параметр `delta`), чтобы он мог взаимодействовать с игрой каждый кадр — например, обновлять позиции объектов:
   ```csharp
   public override void _Process(double delta)
   {
      foreach (var mod in _mods.Values)
         mod.Instance.GetFunction("process")?.Invoke(delta);
   }
   ```
   Здесь `_mods` — это `Dictionary`, хранящий загруженные моды, где ключом является имя мода, а значением — запись (содержащая экземпляр мода и его путь). Аналогично можно передавать события (например, событие выхода), когда игра закрывается или мод выгружается.

2. **Создание API для модов:**  
   Разработайте API для модов, которое будет предоставлять функции (например, `log`, `spawn`, `destroy`, `move`) для мододелов. Стремитесь к простоте и ясности, а также подробно задокументируйте API, чтобы мододелы знали, что доступно и как это использовать.

3. **Автоматическая загрузка модов:**  
   Реализуйте функциональность для загрузки WASM-модов из определенной папки, позволяя мододелам просто помещать свои файлы модов в папку для автоматической загрузки. Также рассмотрите возможность добавления поддержки выгрузки модов, чтобы избежать проблем с производительностью при слишком большом количестве активных модов. Полноценный интерфейс управления модами (например, интегрированный в меню игры) станет отличным улучшением.

Надеюсь, эта небольшая статья поможет вам реализовать WASM-моддинг в ваших играх на Godot .NET. Если у вас возникнут вопросы или потребуется дополнительная помощь, не стесняйтесь оставить комментарий — я постараюсь помочь!