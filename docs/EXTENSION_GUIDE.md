# How to Add Stuff to AsyncREPL

So you want to extend this REPL with your own commands and features. Cool! This guide will walk you through the different ways to do that and when to use each one.

## The Five Ways to Extend

There are basically five ways to add functionality:

1. **Commands** - The stuff users type
2. **Background Tasks** - Things that run without blocking the prompt
3. **Hooks** - Code that runs at specific times (startup, before commands, etc.)
4. **Modules** - Shared code that multiple commands use
5. **Plugins** - Self-contained optional features

Let's go through each one.

---

## 1. Commands

**Use when:** You want to add something users can type and interact with

**Where to put it:** `src/commands/`

Commands are the bread and butter of the REPL. They're what the user types to actually do things.

### Basic Command

```python
# src/commands/basic_commands.py
from src.core.repl_engine import CommandContext, CommandResult

def register_basic_commands(repl):
    @repl.command(
        name="greet",
        description="Say hi to someone",
        usage="greet <name>",
        aliases=["hi"]  # can also use 'hi' instead of 'greet'
    )
    def greet_cmd(ctx: CommandContext):
        # ctx.args is a list of arguments the user typed
        if not ctx.args:
            print("Usage: greet <name>")
            return CommandResult.ERROR
        
        name = ctx.args[0]
        print(f"Hey there, {name}!")
        return CommandResult.SUCCESS
```

### Async Command

If your command needs to do async stuff (like network requests), just make it async:

```python
@repl.command(name="fetch", description="Fetch a URL")
async def fetch_cmd(ctx: CommandContext):
    url = ctx.args[0] if ctx.args else "example.com"
    
    print(f"Fetching {url}...")
    await asyncio.sleep(2)  # pretend this is a real network call
    print(f"Got response from {url}")
    
    return CommandResult.SUCCESS
```

The decorator automatically figures out if your command is sync or async.

### Using Session Data

You can store data between commands using the context:

```python
@repl.command(name="remember")
def remember_cmd(ctx: CommandContext):
    thing = " ".join(ctx.args)
    ctx.set('memory', thing)  # save it
    print(f"I'll remember: {thing}")
    return CommandResult.SUCCESS

@repl.command(name="recall")
def recall_cmd(ctx: CommandContext):
    thing = ctx.get('memory', 'nothing')  # retrieve it (defaults to 'nothing')
    print(f"You told me: {thing}")
    return CommandResult.SUCCESS
```

### Command Checklist
- [ ] Return `CommandResult.SUCCESS` or `CommandResult.ERROR`
- [ ] Check if `ctx.args` is empty before using it
- [ ] Add description and usage text so `help` command works
- [ ] Handle errors gracefully (try/except)

---

## 2. Background Tasks

**Use when:** You need something to run continuously without blocking the prompt

**Where to put it:** Usually inside commands or modules

Background tasks are perfect for monitoring, periodic checks, or long-running operations. The user can keep typing commands while the task runs.

### Simple Background Task

```python
@repl.command(name="countdown", description="Start a countdown timer")
def countdown_cmd(ctx: CommandContext):
    count = int(ctx.args[0]) if ctx.args else 10
    
    async def countdown_task():
        for i in range(count, 0, -1):
            print(f"\r[Timer: {i}]", end="", flush=True)
            await asyncio.sleep(1)
        print("\r[Timer: Done!]      ")
    
    # Emit it as a background task
    ctx.emit_task(countdown_task(), f"countdown_{count}")
    print(f"Started {count} second countdown")
    return CommandResult.SUCCESS
```

Now the user can start a countdown and keep using other commands while it runs.

### File Watcher Example

```python
@repl.command(name="watch", description="Watch a file for changes")
def watch_cmd(ctx: CommandContext):
    if not ctx.args:
        print("Usage: watch <filename>")
        return CommandResult.ERROR
    
    filename = ctx.args[0]
    
    async def file_watcher():
        import os
        last_modified = 0
        
        while True:
            try:
                mtime = os.path.getmtime(filename)
                if mtime > last_modified:
                    print(f"\n[File changed: {filename}]")
                    last_modified = mtime
            except FileNotFoundError:
                print(f"\n[File not found: {filename}]")
                break
            
            await asyncio.sleep(1)  # check every second
    
    ctx.emit_task(file_watcher(), f"watch_{filename}")
    return CommandResult.SUCCESS
```

### Managing Tasks

Use the built-in commands:
- `tasks` - See what's running
- `cancel <task_name>` - Stop a task

### Background Task Tips
- Always use `await asyncio.sleep()` not `time.sleep()`
- Give your tasks descriptive names
- Handle exceptions inside the task (they won't crash the REPL)
- Remember tasks get cancelled when the REPL exits

---

## 3. Hooks

**Use when:** You need to run code at specific times or affect all commands

**Where to put it:** `src/core/hooks.py` usually

Hooks are like event handlers that run automatically. There are four types:

### Startup Hooks (runs once when REPL starts)

```python
# src/core/hooks.py

def setup_hooks(repl):
    @repl.add_startup_hook
    async def load_config(repl_instance):
        print("Loading config...")
        # Load your config file here
        config = {"theme": "dark", "timeout": 30}
        repl_instance.set_session_data('config', config)
    
    @repl.add_startup_hook
    def greet_user(repl_instance):
        print("Welcome! Type 'help' to get started.")
```

Good for: Database connections, loading configs, initializing services

### Pre-Execute Hooks (runs before every command)

```python
@repl.add_pre_execute_hook
def log_commands(ctx: CommandContext):
    """Log every command to a file"""
    with open('command.log', 'a') as f:
        f.write(f"{datetime.now()} - {ctx.raw_input}\n")

@repl.add_pre_execute_hook
async def check_rate_limit(ctx: CommandContext):
    """Prevent spam"""
    last_command_time = ctx.get('last_command', 0)
    now = time.time()
    
    if now - last_command_time < 0.5:  # 500ms cooldown
        print("Slow down!")
        return CommandResult.ERROR
    
    ctx.set('last_command', now)
```

Good for: Logging, authentication checks, rate limiting

### Post-Execute Hooks (runs after every command)

```python
@repl.add_post_execute_hook
def track_stats(ctx: CommandContext, result: CommandResult):
    """Track command success/failure"""
    stats = ctx.get('stats', {'success': 0, 'error': 0})
    
    if result == CommandResult.SUCCESS:
        stats['success'] += 1
    elif result == CommandResult.ERROR:
        stats['error'] += 1
    
    ctx.set('stats', stats)
```

Good for: Metrics, auto-saving state, cleanup

### Shutdown Hooks (runs once when REPL exits)

```python
@repl.add_shutdown_hook
async def cleanup(repl_instance):
    """Close connections and save data"""
    print("Cleaning up...")
    
    # Close database or network connections
    db = repl_instance.get_session_data('db')
    if db:
        await db.close()
    
    # Save stats
    stats = repl_instance.get_session_data('stats')
    if stats:
        with open('stats.json', 'w') as f:
            json.dump(stats, f)
```

Good for: Closing connections, saving data, cleanup

### Hook Tips
- Keep hooks fast - they run on every command (except startup/shutdown)
- Handle exceptions so one bad hook doesn't break everything
- Pre-execute hooks can return `CommandResult.ERROR` to cancel the command

---

## 4. Modules

**Use when:** Multiple commands need the same logic

**Where to put it:** `src/modules/`

Modules are just regular Python code that commands can use. If you find yourself copying code between commands, make it a module.

### Example Module

```python
# src/modules/api_client.py
import asyncio

class APIClient:
    def __init__(self, base_url):
        self.base_url = base_url
        self.cache = {}
    
    async def get(self, endpoint):
        """Fetch data from API"""
        # Check cache first
        if endpoint in self.cache:
            return self.cache[endpoint]
        
        # Pretend we're making a real request
        print(f"Fetching {self.base_url}/{endpoint}...")
        await asyncio.sleep(1)
        
        data = {"status": "ok", "data": "some data"}
        self.cache[endpoint] = data
        return data
    
    def clear_cache(self):
        self.cache.clear()
```

### Using the Module

```python
# src/commands/api_commands.py
from src.modules.api_client import APIClient

def register_api_commands(repl):
    # Create the client once
    client = APIClient("https://api.example.com")
    
    @repl.command(name="get")
    async def get_cmd(ctx: CommandContext):
        endpoint = ctx.args[0] if ctx.args else "/"
        data = await client.get(endpoint)
        print(f"Response: {data}")
        return CommandResult.SUCCESS
    
    @repl.command(name="clear-cache")
    def clear_cmd(ctx: CommandContext):
        client.clear_cache()
        print("Cache cleared")
        return CommandResult.SUCCESS
```

---

## 5. Plugins

**Use when:** You want a self-contained optional feature

**Where to put it:** `src/plugins/`

Plugins are like modules but they register themselves. They're good for features that can be toggled on/off.

### Example Plugin

```python
# src/plugins/git_plugin.py

class GitPlugin:
    def __init__(self, repl):
        self.repl = repl
        self.enabled = True
        self.register()
    
    def register(self):
        """Setup all the plugin's functionality"""
        self._add_commands()
        self._add_hooks()
    
    def _add_commands(self):
        @self.repl.command(
            name="git",
            description="Git operations",
            usage="git <status|add|commit>"
        )
        async def git_cmd(ctx: CommandContext):
            if not ctx.args:
                print("Usage: git <status|add|commit>")
                return CommandResult.ERROR
            
            subcommand = ctx.args[0]
            
            if subcommand == "status":
                # Run git status
                proc = await asyncio.create_subprocess_shell(
                    "git status",
                    stdout=asyncio.subprocess.PIPE
                )
                stdout, _ = await proc.communicate()
                print(stdout.decode())
            elif subcommand == "add":
                print("git add not implemented yet")
            
            return CommandResult.SUCCESS
    
    def _add_hooks(self):
        @self.repl.add_startup_hook
        def check_git(repl_instance):
            import os
            if not os.path.exists('.git'):
                print("Warning: Not in a git repository")
```

### Using Plugins

```python
# main.py
from src.plugins.git_plugin import GitPlugin

repl = AsyncREPLEngine(prompt="dev> ")

# Just instantiate it and it registers itself
GitPlugin(repl)

repl.run()
```

---

## Putting It All Together

Here's what a typical `main.py` might look like:

```python
# main.py
from src.core.repl_engine import AsyncREPLEngine
from src.commands.file_commands import register_file_commands
from src.commands.api_commands import register_api_commands
from src.core.hooks import setup_hooks
from src.plugins.git_plugin import GitPlugin

def main():
    # Create REPL
    repl = AsyncREPLEngine(
        prompt="myapp> ",
        history_file=".myapp_history"
    )
    
    # Setup hooks (logging, stats, etc.)
    setup_hooks(repl)
    
    # Register commands
    register_file_commands(repl)
    register_api_commands(repl)
    
    # Load plugins
    GitPlugin(repl)
    
    # Go!
    repl.run()

if __name__ == "__main__":
    main()
```

---

## Quick Decision Guide

**I want to add something the user can type** → Command

**I want something to run in the background** → Background Task

**I want to run code before/after every command** → Hook

**Multiple commands need the same logic** → Module

**I want an optional feature I can turn on/off** → Plugin

---

## Common Patterns

### Pattern: Command with Background Processing

```python
@repl.command(name="process-file")
def process_cmd(ctx: CommandContext):
    filename = ctx.args[0] if ctx.args else None
    if not filename:
        print("Usage: process-file <filename>")
        return CommandResult.ERROR
    
    async def process_task():
        print(f"Processing {filename}...")
        await asyncio.sleep(3)  # fake work
        print(f"Done processing {filename}")
    
    ctx.emit_task(process_task(), f"process_{filename}")
    print(f"Started processing (will run in background)")
    return CommandResult.SUCCESS
```

### Pattern: Authenticated Commands

```python
# In hooks.py
@repl.add_pre_execute_hook
def require_auth(ctx: CommandContext):
    """Require login for certain commands"""
    cmd_name = ctx.raw_input.split()[0] if ctx.raw_input else ""
    public_commands = ['login', 'help', 'exit']
    
    if cmd_name not in public_commands:
        if not ctx.get('logged_in', False):
            print("Please login first")
            return CommandResult.ERROR

# In commands
@repl.command(name="login")
def login_cmd(ctx: CommandContext):
    username = ctx.args[0] if ctx.args else None
    if username:
        ctx.set('logged_in', True)
        ctx.set('username', username)
        print(f"Logged in as {username}")
        return CommandResult.SUCCESS
    return CommandResult.ERROR
```

### Pattern: Progress Monitoring

```python
@repl.command(name="download")
def download_cmd(ctx: CommandContext):
    url = ctx.args[0] if ctx.args else None
    if not url:
        return CommandResult.ERROR
    
    async def download_task():
        total = 100
        for i in range(total + 1):
            print(f"\rDownloading: {i}%", end="", flush=True)
            await asyncio.sleep(0.1)
        print("\nDownload complete!")
    
    ctx.emit_task(download_task(), f"download")
    return CommandResult.SUCCESS
```

---

## Tips and Tricks

### Tip 1: Check Arguments Before Using Them

```python
# Bad
def bad_cmd(ctx):
    name = ctx.args[0]  # crashes if no args!

# Good
def good_cmd(ctx):
    if not ctx.args:
        print("Usage: command <name>")
        return CommandResult.ERROR
    name = ctx.args[0]
```

### Tip 2: Use Try/Except

```python
@repl.command(name="divide")
def divide_cmd(ctx: CommandContext):
    try:
        a = float(ctx.args[0])
        b = float(ctx.args[1])
        result = a / b
        print(f"Result: {result}")
        return CommandResult.SUCCESS
    except (IndexError, ValueError):
        print("Usage: divide <number> <number>")
        return CommandResult.ERROR
    except ZeroDivisionError:
        print("Can't divide by zero!")
        return CommandResult.ERROR
```

### Tip 3: Don't Block the Event Loop

```python
# Bad - blocks everything
def bad_cmd(ctx):
    time.sleep(5)  # nothing works during this

# Good - doesn't block
async def good_cmd(ctx):
    await asyncio.sleep(5)  # other commands still work
```

### Tip 4: Clean Up Your Background Tasks

```python
@repl.command(name="start-server")
def start_server_cmd(ctx: CommandContext):
    async def server_task():
        try:
            while True:
                # do server stuff
                await asyncio.sleep(1)
        except asyncio.CancelledError:
            # Task was cancelled, clean up
            print("\nServer stopped")
            # close sockets, files, etc.
    
    ctx.emit_task(server_task(), "server")
    return CommandResult.SUCCESS
```

---

## Troubleshooting

### "My async command isn't working"
- Make sure you're using `await asyncio.sleep()` not `time.sleep()`
- Check that you defined it with `async def`
- Make sure you're awaiting any async calls

### "My background task prints weird"
- Use `\r` to overwrite the current line
- Use `flush=True` when printing
- Add `\n` at the end to move to next line
```python
print(f"\rProgress: {i}%", end="", flush=True)  # overwrites line
print()  # move to next line when done
```

### "My hook isn't running"
- Did you remember to register it? (call the function that adds it)
- Is it returning something when it shouldn't? (only pre-execute hooks should return)
- Check for exceptions - they might be silently caught

### "Session data isn't persisting"
- Session data only lasts for the current REPL session
- Use a shutdown hook to save it to disk
- Load it back in a startup hook

---

## Examples You Can Copy

Check the `examples/` folder (if I make one) for complete working examples:
- Simple todo list app
- File processing tool  
- API client
- System monitor

---

That's pretty much it! Start with a few commands, add background tasks when you need them, use hooks sparingly, and you'll figure out the rest as you go.

If something's confusing or broken, open an issue. I'll try to help when I have time between classes.

Happy coding! 🚀
