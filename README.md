# AsyncREPL Engine

A Python REPL (Read-Eval-Print Loop) engine I built to learn more about asyncio and command-line interfaces. It uses decorators for command registration and supports running background tasks while keeping the command prompt interactive.

## Why I Made This

I kept writing similar command-line tools for different projects and got tired of reimplementing the same basic REPL logic every time. So I decided to build something reusable that I could drop into future projects. Plus, I wanted to get better at working with async/await in Python since that's something I didn't fully understand before.

## What It Does

- **Decorator-based commands** - Just add `@repl.command()` above a function and it becomes a command
- **Async support** - Commands can be sync or async, it figures it out automatically
- **Background tasks** - Start long-running tasks that won't block the prompt
- **Hooks** - Run code at startup, shutdown, or before/after every command
- **Tab completion** - Built-in command completion (thanks to readline)
- **Command history** - Saves your command history between sessions

## Quick Example

```python
from src.core.repl_engine import AsyncREPLEngine, CommandContext, CommandResult

# Create the REPL
repl = AsyncREPLEngine(prompt="myapp> ")

# Add a simple command
@repl.command(name="hello", description="Say hello")
def hello_cmd(ctx: CommandContext):
    name = ctx.args[0] if ctx.args else "World"
    print(f"Hello, {name}!")
    return CommandResult.SUCCESS

# Add an async command that does background work
@repl.command(name="countdown", description="Start a countdown")
async def countdown_cmd(ctx: CommandContext):
    count = int(ctx.args[0]) if ctx.args else 10
    
    async def do_countdown():
        for i in range(count, 0, -1):
            print(f"[{i}]")
            await asyncio.sleep(1)
        print("[Done!]")
    
    # This runs in the background - you can still type commands
    ctx.emit_task(do_countdown(), f"countdown_{count}")
    return CommandResult.SUCCESS

# Run it
repl.run()
```

## Project Structure

```
src/
├── commands/      # Your command definitions
├── core/          # The main REPL engine
├── modules/       # Shared logic and utilities
├── plugins/       # Optional features you can toggle
└── utils/         # Helper functions
```

Check out `docs/EXTENSION_GUIDE.md` for more details on how to add stuff.

## Installation

Just clone it and make sure you have Python 3.10+:

```bash
git clone <your-repo-url>
cd repl_testing
python -m pip install -r requirements.txt  # if you add any dependencies
```

## Usage

The basic pattern is:
1. Create a `AsyncREPLEngine` instance
2. Register commands with decorators
3. Optionally add hooks for initialization/cleanup
4. Call `repl.run()`

See the [Extension Guide](docs/EXTENSION_GUIDE.md) for examples of:
- Adding commands
- Running background tasks
- Setting up hooks
- Creating plugins

## Built-in Commands

- `help` - Show all commands (or details about a specific one)
- `exit` - Exit the REPL (also `quit` or `q`)
- `tasks` - List running background tasks
- `cancel <name>` - Cancel a background task
- `history` - Show command history

## Things I Learned

- How to properly use asyncio without blocking the event loop
- The threading model differences between sync and async Python
- Why `asyncio.sleep()` exists (don't use `time.sleep()` in async!)
- How Python decorators can build DSLs (the command registration system)
- Managing background tasks and their lifecycle

## Known Issues / TODO

- [ ] Error messages could be more helpful
- [ ] No unit tests yet (I know, I know...)
- [ ] Documentation is sparse in some places
- [ ] Command argument parsing is super basic (just splits on spaces)
- [ ] No built-in way to pipe commands or handle redirects

## Why It's Not "Production Ready"

This is a learning project. I haven't tested it extensively, the error handling could be better, and I'm sure there are edge cases I haven't thought of. If you want to use it for something serious, you should probably go with something more established like [cmd](https://docs.python.org/3/library/cmd.html) or [click](https://click.palletsprojects.com/).

That said, it works well enough for my personal projects and I've learned a ton building it.

## Contributing

Feel free to fork it and do whatever you want with it. If you find bugs or have suggestions, open an issue. I can't promise I'll get to everything quickly since this is a side project and I'm in school, but I'll try to respond when I can.

## License

Do whatever you want with it. Just don't blame me if something breaks. See LICENSE for the boring legal stuff.

---

Last updated: November 2025
