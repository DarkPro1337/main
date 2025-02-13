---
title: "WASM Modding for Your Godot .NET Games"
summary: "While developing Arcomage with Godot .NET, I encountered the challenge of adding support for script-based modding to the game."
date: 2025-02-13T04:48:34+04:00
draft: false
tags: ["wasm", "godot", "modding", "devlog"]
---

While developing **Arcomage** using **Godot .NET**, I faced a challenge: how can I add support for script-based modding to the game? The issue was that I didn't want to allow direct DLL injection—highly insecure and prone to abuse by injecting malicious code that could compromise user systems or steal sensitive data like passwords.

Instead, I opted for a sandboxed approach where mods execute safely without posing risks to end users. My goal was to create a common API for all mods to make them both safer and easier to write. That's when I came across a video by [the nameless dev](https://youtu.be/kcWVYeaFmqQ) that clearly and concisely explained how to implement WASM mod support. Huge thanks to the author for sharing his insights!

I was aware of WebAssembly before, but I hadn’t considered it as a sandbox for running script-based mods in a game. It was a revelation to learn that Microsoft Flight Simulator 2020 uses WebAssembly for mod scripts—and that's pretty awesome!

So, I decided to try something similar for Arcomage. In this article, I'll share my findings and ideas on implementing WASM modding in Godot .NET.

## What Is WebAssembly and How Do You Use It?

**WASM** is a bytecode format that runs in a virtual machine. Unlike .NET's CIL, WASM is designed to execute in browsers or any application that supports it. In our case, we're using Godot's .NET version. For .NET, several embedded WASM runtimes exist. In my project, I used [Wasmtime](https://wasmtime.dev/) from the Bytecode Alliance.

## Example: Creating a WASM Mod

### Setup

First, add the Wasmtime NuGet package to your Godot .NET project:
```bash
dotnet add package wasmtime
```
Alternatively, add the following Package Reference to your `.csproj` file:
```xml
<PackageReference Include="Wasmtime" Version="22.0.0" />
```
Don't forget to run `dotnet restore` after modifying your project file.

To compile **WASM** files from **AssemblyScript**, install **Node.js** (which comes with **npm**). You can download it from [Node.js's official website](https://nodejs.org/en). Verify the installations with:
```bash
node -v
npm -v
```

In this example, we'll compile WASM using **AssemblyScript**. If you prefer another language (like **Rust** with **wasm-pack**), refer to its documentation. I also highly recommend reviewing the [AssemblyScript compiler docs](https://www.assemblyscript.org/compiler.html).

After installing Node.js, run:
```bash
npm install -g assemblyscript
```
This installs the AssemblyScript compiler globally (or install it locally if preferred).

With the necessary tools in place, it's time to write some code!

### WASM

Choose a working directory and create two files: `env.ts` and `mod.ts`.

In `env.ts`, define the exported functions you want available to your WASM mod:
```typescript
@external("env", "host_log")
declare function host_log(ptr: i32, len: i32): void;

export function log(message: string): void {
  const encoded = String.UTF8.encode(message);
  host_log(changetype<i32>(encoded), encoded.byteLength);
}
```
In this example, the `log` function sends a message to the host via `host_log`, which receives a pointer to the UTF-8–encoded string and its length.

In `mod.ts`, write the code to execute in the WASM mod:
```typescript
import { log } from "./env";

export function init(): void {
  log("Hello, World!");
}
```
Here, the exported `init` function will be called upon mod initialization, triggering a log message. On the Godot side, you'll handle this (for example, by printing it to the console with `GD.Print`).

This simple setup illustrates the concept. As a developer, share the `env.ts` file with your modders so they can use the defined functions. In this scheme, `mod.ts` serves as the mod's entry point, where you specify an entry function (in this case, `init`) that the game calls automatically when loading the WASM file.

To compile your WASM mod, run:
```bash
asc .\mod.ts -o mod.wasm
```
If AssemblyScript isn’t installed globally, ensure you provide the correct path. Running this command produces a `mod.wasm` file—the entry point for your mod.

### Godot .NET Integration

Place the compiled WASM file in your Godot project's `user` directory (e.g., `user://mod.wasm`).

I recommend using a custom user directory, enabled via the `application/config/use_custom_user_dir` flag in your `project.godot` file. Also, set a folder name using `application/config/custom_user_dir_name`.

Create a new C# class inheriting from `Node`—in my example, it's named `WasmLoader`:
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

Let's break down what happens in this code:

1. The WASM file is loaded from `user://mod.wasm` and its bytes are passed to `Module.FromBytes`.
2. An `Engine`, `Module`, `Store`, and `Linker` are created to execute the WASM mod.
3. The functions exported by the mod are defined. Here, `host_log` takes a pointer and length, decodes the string from memory, and prints it.
4. The `abort` function is also defined; it’s called on errors within the WASM mod. Although you might not need it, the compiler includes it by default. Without defining it, you’d get a `Wasmtime.WasmtimeException` for an undefined import.
5. Finally, the WASM mod is instantiated and its `init` function is invoked, printing "Hello, World!" to the console.

After creating the `WasmLoader` class, add it to your scene and run the game. If everything is set up correctly, you should see "Hello, World!" in the console. Alternatively, you can add the loader to your project's `Autoload` to run it at startup.

## Next Steps

Here are a few ideas to expand your modding system:

1. **Forwarding Process Callbacks:**  
   Pass the `_Process` and `_PhysicsProcess` callbacks to the WASM mod (remember to include `delta` as a parameter) so it can interact with the game each frame—for example, updating object positions:
   ```csharp
   public override void _Process(double delta)
   {
      foreach (var mod in _mods.Values)
         mod.Instance.GetFunction("process")?.Invoke(delta);
   }
   ```
   Here, `_mods` is a `Dictionary` storing loaded mods, with the mod name as the key and a record (containing the mod instance and its path) as the value. You can similarly forward events (like an `exit` event) when the game closes or a mod is unloaded.

2. **Creating a Mod API:**  
   Develop a Mod API that exposes functions (e.g., `log`, `spawn`, `destroy`, `move`) for modders. Aim for simplicity and clarity, and document the API thoroughly so modders know what’s available and how to use it.

3. **Automatic Mod Loading:**  
   Implement functionality to load WASM mods from a designated folder, allowing modders to simply drop their mod files into the folder for automatic loading. Also, consider adding support for unloading mods to avoid performance issues with too many active mods. A full mod management interface (such as one integrated into the game menu) would be a great enhancement.

I hope this small article helps you implement WASM modding in your Godot .NET games. If you have any questions or need further assistance, feel free to leave a comment—I'll do my best to help!