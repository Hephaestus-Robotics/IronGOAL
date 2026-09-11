# IronGOAL

![Build](https://github.com/nathanpapke/ironGOAL/actions/workflows/dotnet-desktop.yml/badge.svg)
![License](https://img.shields.io/badge/license-MIT-blue)
![.NET](https://img.shields.io/badge/.NET-8.0-purple)

IronGOAL is a .NET class library to implement the GOAL scripting language as an
engine-agnostic package.  It uses IronScheme as the basis for running Scheme
R5RS.

## Overview

This library is directed towards developers who want Scheme as a scripting
language for 3D environments in their own engines, researchers studying GOAL
architecture, and game develpers who want to run script compatible with
Naughty Dog games in a .NET environment.  The artificial intelligence power of
Scheme can be used in a 3D environment to enhance characters in game worlds
and provide simulations for robotics.  GOAL can be used with this class library
in a pure .NET environment.

## Features

- Registers GOAL functions in an IronScheme environment
- Usable in any engine running .NET 8 or higher
- Compatible with Scheme R5RS

## Architecture / How It Works

The core concept of this class library is in the `Kernel` class that registers
the GOAL functions implemented as public static methods in a number of classes
in `IronGOAL.Backing`.  Higher level functions are exposed through an event
bus to be implemented by the end developer.

## Getting Started

The best way to get started is by cloning the repo with the IDE of your
choice.  IronGOAL has been used successfully in JetBrains Rider.  After
building the solution, IronGOAL can be linked to from another project.

## Usage

IronGOAL is a library, not an application. The host owns the frame loop, the
renderer, and the assets; IronGOAL owns the Scheme environment and the script
processes. They meet at two places: the host calls `Tick`, and the host drains
channels.

### Minimal Host

```csharp
using IronGOAL;

var result = Host.Create(new GoalRuntimeConfig
{
    LogHandler      = (sev, code, msg) => Console.WriteLine($"[{sev}] {msg}"),
    ScriptDirectory = "scripts/",
});

if (result.IsFailure)
    throw new InvalidOperationException(result.ErrorMessage);

using Host host = result.Value!;

host.LoadScript("boot.gc");

while (running)
{
    host.Tick(1f / 60f);
    DrainChannels(host);
}
```

Every `Host` method returns a `GoalResult` rather than throwing. Check
`IsSuccess` before reading `Value`; reading `Value` on a failed result is a
programming error and logs at `Fatal`.

### Loading Scripts

```csharp
host.LoadScript("action-walk.gc");   // relative to ScriptDirectory
host.LoadScript(@"C:\dev\boot.gc");  // absolute paths work with no config
```

**Load order matters.** Scheme resolves references at evaluation time, so a
file that calls into another must be loaded after it:

```csharp
foreach (var file in new[]
{
    "action-idle.gc",
    "action-walk.gc",
    "action-punch.gc",
    "boot.gc",          // last - references the above
})
{
    var load = host.LoadScript(file);
    if (load.IsFailure)
        Log.Error(load.ErrorMessage);
}
```

`LoadScript` evaluates one top-level form at a time. A form that fails does
not abort the file - the remaining forms still run, and the failure is
reported per form. Two error codes distinguish the cases:
`ScriptSyntaxError` means the reader could not parse the source at all (an
unmatched bracket, an unterminated string) and **no** forms ran;
`ScriptEvalFailed` means every form parsed but at least one raised.

Do not use Scheme's own `(load "other.gc")` from inside a `.gc` file.
IronScheme treats loaded files as R6RS top-level programs and rejects
anything without an `(import ...)` clause - silently, returning
`#<unspecified>` rather than raising. Load from C# instead.

### Evaluating Expressions

```csharp
object? value = host.Evaluate("(+ 1 2)");        // raw result, null if empty
FormResult r  = host.EvaluateForm("(car '())");  // never throws

if (!r.Success)
    Log.Error(r.ErrorMessage);
```

Use `EvaluateForm` for anything user-supplied - a REPL line, a hot-reload
snippet - since it captures the Scheme condition instead of raising it.
`Evaluate` is the terser choice when you control the input.

Bindings persist. The environment is a single shared
`interaction-environment`, so a `define` from one call is visible to every
later call and to every loaded script.

Numeric results come back boxed the way IronScheme boxes them: integers as
`long`, flonums as `double` - not `float`. Convert at the boundary.

### The Frame Tick

```csharp
host.Tick(deltaTime);
```

One call per simulation step. `Tick` advances the game clock and runs every
ready script process until it suspends or returns. Processes that called
`suspend` resume here; processes waiting on a host answer stay parked until
the answer arrives.

A delta of zero is valid - useful for stepping the scheduler without
advancing time. Negative deltas are rejected.

### Draining Channels

IronGOAL never touches a GPU, an audio device, or a file. It publishes typed
structs and the host decides what they mean. Drain each channel at whatever
cadence suits it - transforms once per frame, audio when convenient:

```csharp
void DrainChannels(Host host)
{
    while (host.TransformCommands.TryRead(out TransformCommand cmd))
        _scene.SetWorldTransform(cmd.EntityId, cmd.Transform);

    while (host.AudioCommands.TryRead(out AudioCommand cmd))
        _audio.Handle(cmd);

    while (host.GameEvents.TryRead(out GameEvent evt))
        HandleGameEvent(host, evt);

    while (host.DebugCommands.TryRead(out var stamped))
        Log.Debug($"[frame {stamped.FrameId}] {stamped.Command}");
}
```

Handles are plain integers. IronGOAL mints them; you map them to your own
meshes, clips, and entities. It never sees your engine's types.

### Answering Queries

Scripts can ask the host questions - `(entity-get-pos handle)`,
`(entity-exists? handle)`, `(heap-bytes-used "global")`. These suspend the
calling process, publish a `GameEventType.EntityQuery`, and wait.

The query arrives with the operation in `Param0` and the requesting process
handle in `Param3`. Answer it with `AnswerEntityQuery` and the process wakes
on the next tick:

```csharp
void HandleGameEvent(Host host, GameEvent evt)
{
    switch (evt.Type)
    {
        case GameEventType.EntitySpawn:
            _scene.Spawn(evt.Param0);       // type-name hash, not an opcode
            break;

        case GameEventType.EntityKill:
            _scene.Destroy(evt.EntityId);
            break;

        case GameEventType.EntityQuery:
            AnswerQuery(host, evt);
            break;
    }
}

void AnswerQuery(Host host, GameEvent evt)
{
    long process = evt.Param3;

    switch ((Opcode)evt.Param0)
    {
        case Opcode.Exists:
            host.AnswerEntityQuery(process, _scene.Exists(evt.EntityId));
            break;

        case Opcode.GetPosition:
            host.AnswerEntityQuery(process, _scene.Position(evt.EntityId));
            break;

        default:
            host.AnswerEntityQuery(process, null);   // script receives #f
            break;
    }
}
```

Three rules for answering:

- **Always answer.** A query with no response leaves that process suspended
  forever. Deposit `null` to decline - the script receives `#f`.
- **Match the expected type.** `bool` for predicates, `long` for handles,
  `Vector3` / `Quaternion` for transform components, `long[]` for
  multi-entity results.
- **`Param3` is reserved.** It carries the process handle on every query;
  never read it as data.

`Param0` is only an opcode on `EntityQuery` and `EntitySetState`. On
lifecycle events it carries data - `EntitySpawn` puts a type-name hash there.
Check `Type` before casting.

### Shutdown

```csharp
host.Dispose();
```

`Dispose` never throws. Calls to a disposed host return
`GoalErrorCode.RuntimeDisposed` rather than raising.

A script calling `(kernel-shutdown)` does not stop the loop - it publishes
`GameEventType.KernelShutdown` and the host decides when to actually tear
down.

## Configuration

IronGOAL is configured entirely through `GoalRuntimeConfig`, an init-only
record passed to `Host.Create`. There are no config files, environment
variables, or global settings - the host owns every decision.

```csharp
var config = new GoalRuntimeConfig
{
    LogHandler      = (severity, code, message) =>
                          Console.WriteLine($"[{severity}] {code}: {message}"),
    ScriptDirectory = "scripts/",
    GlobalHeapSize  = 64 * 1024 * 1024,
    StackHeapSize   =  8 * 1024 * 1024,
};

var result = Host.Create(config);
if (result.IsFailure)
    throw new InvalidOperationException(result.ErrorMessage);

using Host host = result.Value!;
```

`Host.Create` returns a `GoalResult<Host>` rather than throwing. Check
`IsSuccess` before reading `Value`.

### Logging

| Property | Type | Default |
|---|---|---|
| `LogHandler` | `GoalLogHandler` | **required** |

IronGOAL never decides where its output goes. It calls your delegate and
nothing else - no `Console.WriteLine`, no file, no trace listener.

```csharp
public delegate void GoalLogHandler(
    GoalLogSeverity severity,  // Info, Warning, Error, Fatal
    GoalErrorCode   code,
    string          message);
```

**The handler must never throw.** It is invoked from inside catch blocks and
from the guard path on failed results, where an exception has nowhere to go.

### Heaps

| Property | Type | Default |
|---|---|---|
| `GlobalHeapSize` | `int` | 64 MB |
| `StackHeapSize` | `int` | 8 MB |

The global heap holds long-lived data - entities, level resources, type
tables. The stack heap is per-process and is released automatically when a
`ScriptProcess` exits.

Both must be greater than zero, and the stack heap must be smaller than the
global heap; `Host.Create` rejects anything else with
`GoalErrorCode.InvalidConfig`.

### Script Loading

| Property | Type | Default |
|---|---|---|
| `ScriptDirectory` | `string?` | `null` |

The root that `LoadScript` resolves relative paths against. Leave it `null`
and every path passed to `LoadScript` must be absolute.

### Channel Capacities

| Property | Default | Full-channel behavior |
|---|---|-----------------------|
| `TransformChannelCapacity` | 4096 | Wait - backpressure   |
| `AudioChannelCapacity` | 1024 | Drop                  |
| `GameEventChannelCapacity` | 512 | Drop                  |
| `DebugChannelCapacity` | 256 | Drop                  |
| `MemoryChannelCapacity` | 128 | Wait - backpressure   |

Sized so a host draining once per frame never fills them under normal load.
Raise them if your engine drains less often than once per simulation tick.

Wait-mode channels apply real backpressure: a full channel suspends the
publishing script process until the host drains it, without blocking a .NET
thread. If a publish happens with no process context to suspend, the command
is dropped and counted. Poll `EventBus.TransformDropCount` and
`EventBus.MemoryDropCount` to detect it - both are monotonic and never reset.

### Development Flags

| Property | Type | Default |
|---|---|---|
| `EnableMemoryTracking` | `bool` | `true` |
| `EnableDebugChannel` | `bool` | `false`-worthy in ship builds |

`EnableMemoryTracking` activates the memory event channel and per-allocation
publishing. `EnableDebugChannel` activates `print`, `inspect`, and `_format`
publishing.

Turn both off for production. With the debug channel disabled, the backing
methods still compute and return their values, so `(_format #f ...)`
string-building continues to work - only the publishing stops.

### Sharing a Scheme Environment

| Property | Type | Default |
|---|---|---|
| `SchemeEnvironment` | `object?` | `null` |

An existing IronScheme top-level environment - the value from
`"(interaction-environment)".Eval()`. Leave it `null` and IronGOAL obtains one
for itself, exposed afterward as `Host.SchemeEnvironment`.

This is how a second kernel joins the first kernel's symbol table. The core
GOAL-faithful kernel and the extensions kernel share one namespace by passing
the first host's environment into the second's config:

```csharp
var extensions = Host.Create(new GoalRuntimeConfig
{
    LogHandler        = logHandler,
    SchemeEnvironment = core.SchemeEnvironment,
});
```

### Database

| Property | Type | Default |
|---|---|---|
| `SqlQueryHandler` | `SqlQueryDelegate?` | `null` |

Backs `(sql-query ...)` from scripts. The delegate receives the raw SQL string
and returns a `string[]` where element 0 is the content-type name and the rest
are flat field values — mirroring the layout of GOAL's `sql-result` type from
Jak X. Return `null` to signal failure and `sql-query` returns `#f`.

Left `null`, every `sql-query` call returns `#f` immediately, matching what
`sqlpipe-query` did when the named pipes were absent.

## Running the Tests

The test suite is headless - no engine, no graphics context, no audio device,
and no file I/O beyond script fixtures. Every test boots a real `Host` and
exercises the kernel through the same public surface a game engine would use.

### Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)

No other dependencies. `dotnet restore` pulls IronScheme and xUnit
automatically.

### Run Everything

```bash
dotnet test
```

From the repository root this builds the solution and runs all tests in
`Tests/`. To run the test project alone:

```bash
dotnet test Tests/Tests.csproj
```

### Run a Subset

Filter by fully qualified name to target one backing system:

```bash
# One test class
dotnet test --filter FullyQualifiedName~GameMathTests

# Everything touching the script loader
dotnet test --filter FullyQualifiedName~ScriptLoader

# One test method
dotnet test --filter FullyQualifiedName=Tests.KernelTests.Tick_Succeeds
```

### Verbose Output

Test failures print the Scheme condition that caused them. If you need to see
passing output as well:

```bash
dotnet test --logger "console;verbosity=detailed"
```

### Code Coverage

The project includes `coverlet.collector`:

```bash
dotnet test --collect:"XPlat Code Coverage"
```

Results land in `Tests/TestResults/<guid>/coverage.cobertura.xml`.

### From an IDE

The suite runs unmodified in the JetBrains Rider and Visual Studio test
runners via `xunit.runner.visualstudio`. No launch configuration is needed.

---

### How the Tests are Organized

One test class per backing system - `GameMathTests`, `EntitySystemTests`,
`TypeSystemTests`, `GameMemoryTests`, `AssetSystemTests`, `FileSystemTests`,
`DatabaseSystemTests` - plus `KernelTests` for lifecycle and `ScriptLoaderTests`
for the form-chunked evaluator.

Each class covers three concerns:

- **Symbol registration** - the GOAL symbol is bound and callable from Scheme
  (`(procedure? kmalloc)`)
- **Behavior** - the C# backing method produces the expected result
- **Guards** - wrong argument counts and wrong types return `#f` rather than
  raising a Scheme condition

### Writing New Tests

A few constraints are worth knowing before adding tests.

**Boot through `Host.Create`, not by touching IronScheme directly.** Test
classes create a `Host` with small heaps and logging suppressed:

```csharp
static readonly GoalRuntimeConfig Config = new GoalRuntimeConfig
{
    GlobalHeapSize           = 16 * 1024 * 1024,
    StackHeapSize            =  2 * 1024 * 1024,
    TransformChannelCapacity = 64,
    EnableMemoryTracking     = false,
    EnableDebugChannel       = false,
    LogHandler               = (_, _, _) => { }
};

static MySystemTests() => Host.Create(Config);
```

**The Scheme environment is global.** IronScheme's `interaction-environment`
is a single static environment shared across every `Evaluate` call in the
process, so bindings defined by one test are visible to all the others. Prefix
any global you define with something unique to the test - `ScriptLoaderTests`
uses `sl-` - or a name collision will surface as a baffling failure in an
unrelated class.

**Pass strings to type-guard tests, not CLR primitives.** Handing a raw `int`
or `float` to a backing method where the guard expects a rejection can crash
inside IronScheme's boxing layer rather than returning `#f`. A `string`
argument exercises the same branch safely.

**Expect `double`, not `float`.** IronScheme boxes flonum literals as
`System.Double`. Use the `AsFloat` / `AsInt` helpers at the interop boundary.

**Query-suspending methods are guard-tested only.** A full
suspend-deliver-resume round trip needs a live `ScriptProcess` and scheduler
harness, which is out of scope for unit tests. These methods are covered for
their wrong-argument and no-process-context paths.

**The test project name matters.** `IronGOAL.csproj` grants
`InternalsVisibleTo` to an assembly named exactly `Tests`. Renaming the project
breaks access to every internal type the suite reaches for.

**Fix the source, not the test.** If a test fails because the backing code is
wrong, the backing code is what changes.

## Roadmap

IronGOAL is developed in milestones. Checked items are complete; unchecked
items are planned or in progress.

### Milestone 1 - Backing Kernel *(complete)*

- [x] Single shared IronScheme environment hosting a GOAL-faithful core
      kernel alongside a separate extensions kernel
- [x] Eleven backing classes covering math, clock, processes, entities,
      animation, audio, input, graphics, physics, memory, types, debug,
      assets, files, and database
- [x] 89 GOAL-origin symbols registered under their original names, each
      verified against the original kernel source (`kscheme.cpp`,
      `kmachine.cpp`, `kmemcard.cpp`, and the `.gc` libraries)
- [x] Cooperative process scheduler — `run-process-and-function`,
      `run-function-in-process`, `set-to-run-function`, `suspend`,
      `defstate`, `go`, `send-event`, `kernel-shutdown`
- [x] Type system — `deftype`, `method-set!`, `method-id`, `type-type?`,
      `type-of`, with sequential vtable slot assignment
- [x] Silent stub registrations for PS2-specific symbols (linker and object
      file, C-kernel internals, IOP RPC, `scf-get-*`, memory card language
      setters) so genuine `.gc` files load without unrecognized-symbol errors
- [x] Headless xUnit test suite

### Milestone 2 - Event Bus & Host Boundary *(complete)*

- [x] Channel-based publish/subscribe bus with typed command structs —
      transform, audio, game event, debug, and memory channels
- [x] Engine-agnostic contract: the kernel emits plain handles and value
      structs, never engine types, and touches no GPU or audio resource
- [x] Process-suspend backpressure on Wait-mode channels, chosen after a
      written comparison against spin-wait and silent-drop alternatives
- [x] Observable drop counters for publishes originating outside a process
      context
- [x] Frame-stamped debug commands (`FrameId`, `GameTime`) for correlating
      output to a simulation step

### Milestone 3 - Opcode Registry *(complete)*

- [x] Structured bit-layout encoding — direction in bit 0, category in bits
      1–3, ordinal in bits 4–7 — replacing the earlier per-system numeric
      ranges
- [x] 39 opcodes assigned across five categories: GameMemory, Entity, Asset,
      File, and Audio, with three category slots held in reserve
- [x] `Invalid = 0xFF` sentinel and `OpcodeBits` decode helpers so the host
      can branch on direction or category without a lookup table

### Milestone 4 - Script Loading *(complete)*

- [x] Form-chunked evaluator (`ScriptLoader`) — reads one top-level form at
      a time with IronScheme's own reader and evaluates each into the shared
      environment, bypassing the R6RS top-level-program validator that causes
      `load` to silently reject plain R5RS `.gc` files
- [x] Per-form error reporting via `ScriptLoadReport`, so one bad form no
      longer discards an entire file
- [x] Distinct error codes for syntax failures and evaluation failures
- [x] Line-ending normalization for embedded hosts

## Contributing

Contributors can provide help via testing and writing interfaces to run in
various game engines.

## Acknowledgements

### Foundational Works

**[Naughty Dog's GOAL](https://en.wikipedia.org/wiki/Game_Oriented_Assembly_Lisp)**
IronGOAL is an implementation of Naughty Dog's GOAL scripting language
originally created to write Jak & Daxter.  GOAL provides functions in Scheme
for a 3D game environment, allowing a set of classical artificial intelligence
algorithms to be used in a manner that can simulate real-world environments.

**[IronScheme](https://github.com/leppie/IronScheme)**
IronGOAL uses IronScheme to run a Scheme environment in a purely .NET runtime.

**[OpenGOAL / jak-project](https://github.com/open-goal/jak-project)**
The GOAL functions were made known thanks in effort to the team creating
OpenGOAL and the jak-project.  

### Individuals

**Andy Gavin** [@agavin42](@agavin42) - Creator of GOAL.  One of the
co-founders of Naughty Dog, Gavin created Game Oriented Assembly Lisp for the
Jak & Daxter series after his education at MIT, backed by a rich history of
computer science dating back to the coining of the term artificial
intelligence and the creation of the Lisp programming language.

**Llewllyn Pritchard** [@leppie](@leppie) - Creator of IronScheme.  Inspired
by the DLR, Pritchard created IronScheme to fill a niche in .NET programming
and has continued to provide inspiration and assistance to fellow software
developers and even provided direct assistance with the `DefineFunction`
method that is used in the `Kernel` of this project.

## License

Distributed under the MIT License. See [LICENSE](LICENSE) for details.
