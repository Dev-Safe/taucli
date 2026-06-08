# TauCLI

A robust, chainable command-line argument parser for the Tauraro programming language.

TauCLI provides a fluent builder API to construct complex CLI applications with deep subcommand trees, robust flag parsing, and type-safe value extraction.

## Capabilities

- **Fluent Builder API:** Construct your CLI tree by chaining `.subcommand()`, `.arg()`, and `.flag()`.
- **Flexible Flags:** Support for toggles, long names, and shorthands (e.g., `-r, --release`), as well as default values.
- **Multiple/Repeating Flags:** Safely parses flags provided multiple times (e.g., `-I src -I include`) into a list of raw slices.
- **Positional Arguments:** Support for required, optional, and variadic arguments with customizable validation matching.
- **Dual Execution Patterns:** Execute your CLI logic via integrated handler routing, or manually inspect the `ParseResult` for complete imperative control.
- **Type-Safe Extraction:** Retrieve parsed inputs dynamically using `get_arg()`, `get_flag()`, and their `_int()` variants.

---

## Building the Command Tree

Everything starts by initializing a root `Command` via `parser()`. You then attach your arguments, flags, and nested subcommands.

```python
from lib import parser, Flag, Arg, Command, ParseResult

mut my_cli = parser("taupkg", "Tauraro package manager")
    .subcommand(
        Command.init("build", "Compile the project")
            .arg(Arg.init("target", "Target architecture").optional(true))
            .flag(Flag.toggle("r,release", "Compile in release mode"))
            .flag(Flag.str("o,output", "Output directory"))
    )
```

---

## Execution Patterns

TauCLI supports two distinct paradigms for executing your command-line logic once parsing is complete.

### Pattern 1: Integrated Routing (Declarative)

Attach functions directly to your commands using `.handler()`. When parsing finishes, simply call `.handle()` on the result, and TauCLI will automatically route execution to the deepest matched subcommand.

```python
# 1. Define the handler
def handle_build(cmd: ParseResult) -> Result[int, str]:
    if cmd.get_flag("release").is_some():
        print("Building in release mode...")

    mut out_flag = cmd.get_flag("output")
    if out_flag.is_some():
        print("Outputting to: " + out_flag.unwrap().raw_slices.get(0))

    return Result.ok(0)

def main(args: List[str]) -> int:
    args.remove(0)

    # 2. Bind the handler to the command
    mut p = parser("taupkg", "Desc")
        .subcommand(
            Command.init("build", "Compile")
                .flag(Flag.toggle("r,release", "Release mode"))
                .flag(Flag.str("o,output", "Output dir"))
                .handler(handle_build)
        )
        .parse(args)

    if p.is_err():
        print(p.unwrap_err())
        return 1

    # 3. Automatically execute the correct handler
    return p.unwrap().handle().unwrap_or(-1)
```

### Pattern 2: Manual Evaluation (Imperative)

If you prefer not to bind functions to commands, you can omit the handlers, parse the arguments, and manually evaluate the `ParseResult` in your main loop.

```python
def main(args: List[str]) -> int:
    args.remove(0)

    # 1. Build and parse without handlers
    mut p = parser("taupkg", "Desc")
        .subcommand(Command.init("build", "Compile"))
        .subcommand(Command.init("clean", "Remove artifacts"))
        .parse(args)

    if p.is_err():
        print(p.unwrap_err())
        return 1

    mut result = p.unwrap()

    # 2. Inspect the matched command manually
    if result.matched_cmd.name == "build":
        print("Executing build logic...")
        # Access flags/args via result.get_flag() etc.

    elif result.matched_cmd.name == "clean":
        print("Cleaning workspace...")

    else:
        print("Usage: taupkg [build|clean]")

    return 0
```

---

## Data Extraction API

Once you have a `ParseResult`, you can safely extract the values the user provided. All extraction methods return `Option` wrappers to prevent null-reference panics.

- `get_arg(name: str) -> Option[Pointer[ArgValue]]`: Fetches a positional argument.
- `get_arg_int(name: str) -> Option[int]`: Fetches a positional argument and casts it to an integer.
- `get_flag(name: str) -> Option[Pointer[FlagValue]]`: Fetches a flag. Use `.unwrap().raw_slices` to access the values (handles repeating flags).
- `get_flag_int(name: str) -> Option[int]`: Fetches the first value of a flag and casts it to an integer.
- `has_flag(name: str) -> bool`: Quick boolean check to see if a flag (like a toggle) was provided.
