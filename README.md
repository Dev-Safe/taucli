# TauCLI

A robust, chainable command-line argument parser for the Tauraro programming language.

TauCLI provides a fluent builder API to construct complex CLI applications with deep subcommand trees, robust flag parsing, automatic help generation, and type-safe value extraction.

## Capabilities

- **Fluent Builder API:** Construct your CLI tree instantly by chaining `.subcommand()`, `.str_arg()`, and `.toggle_flag()`.
- **Flexible Flags:** Full support for toggles, long names, and shorthands (e.g., `-r, --release`), as well as fallback default values.
- **Multiple/Repeating Flags:** Safely parses flags provided multiple times (e.g., `-I src -I include`) into a list of raw slices.
- **Positional Arguments:** Support for required, optional, and variadic arguments with customizable validation matching.
- **Automatic Help Generation:** Intelligently provides `-h` and `--help` menus for all commands and subcommands—unless you choose to override them.
- **Dual Execution Patterns:** Execute your CLI logic via integrated handler routing, or manually inspect the `ParseResult` for complete imperative control.
- **Type-Safe Extraction:** Retrieve parsed inputs dynamically using `get_arg_str()`, `get_flag_str()`, and their `_int()` / `_float()` variants.

---

## Building the Command Tree

Everything starts by initializing a root `Command` via `parser()`. You then attach your arguments, flags, and nested subcommands using the built-in DX (Developer Experience) helper methods.

```tauraro
from lib import parser, Command, ParseResult

mut my_cli = parser("taupkg", "Tauraro package manager")
    .subcommand(
        Command.init("build", "Compile the project")
            .opt_str_arg("target", "Target architecture")
            .toggle_flag("r,release", "Compile in release mode")
            .str_flag("o,output", "Output directory")
    )
```

---

## Execution Patterns

TauCLI supports two distinct paradigms for executing your command-line logic once parsing is complete.

### Pattern 1: Integrated Routing (Declarative)

Attach functions directly to your commands using `.handler()`. When parsing finishes, simply call `.handle()` on the result, and TauCLI will automatically route execution to the deepest matched subcommand.

```tauraro
# 1. Define the handler
def handle_build(cmd: ParseResult) -> Result[int, str]:
    if cmd.has_flag("release"):
        print("Building in release mode...")

    mut out_flag = cmd.get_flag_str("output")
    if out_flag.is_some():
        print("Outputting to: " + out_flag.unwrap())

    return Result.ok(0)

def main(args: List[str]) -> int:
    args.remove(0)

    # 2. Bind the handler to the command
    mut p = parser("taupkg", "Desc")
        .subcommand(
            Command.init("build", "Compile")
                .toggle_flag("r,release", "Release mode")
                .str_flag("o,output", "Output dir")
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

```tauraro
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
        # Access flags/args via result.get_flag_str() etc.

    elif result.matched_cmd.name == "clean":
        print("Cleaning workspace...")

    else:
        print("Usage: taupkg [build|clean]")

    return 0
```

---

## Data Extraction API

Once you have a `ParseResult`, you can safely extract the values the user provided. All extraction methods return `Option` wrappers to prevent null-reference panics.

### Arguments

- `get_arg_str(name: str) -> Option[Pointer[char]]`: Fetches a positional argument as a string.
- `get_arg_int(name: str) -> Option[int]`: Fetches a positional argument and casts it to an integer.
- `get_arg_float(name: str) -> Option[float]`: Fetches a positional argument and casts it to a float.

### Flags

- `has_flag(name: str) -> bool`: Quick boolean check to see if a flag (like a toggle) was provided.
- `get_flag_str(name: str) -> Option[Pointer[char]]`: Fetches the first string value of a flag.
- `get_flag_int(name: str) -> Option[int]`: Fetches the first value of a flag and casts it to an integer.
- `get_flag_float(name: str) -> Option[float]`: Fetches the first value of a flag and casts it to a float.

_(Note: To retrieve a list of all values passed to a repeating flag, you can still use the raw `get_flag(name)` method to access the `FlagValue` object directly)._
