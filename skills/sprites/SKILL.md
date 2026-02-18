---
name: sprites
description: Use sprites.dev to create, manage, and execute code in persistent cloud sandbox environments. Invoke when the user wants to spin up a sprite, run commands remotely, manage checkpoints, or work with the sprites CLI/SDK.
user-invocable: true
argument-hint: [command-or-task]
allowed-tools: Bash, Read, Write, Edit
---

# Sprites.dev — Persistent Cloud Sandbox Environments

Sprites are hardware-isolated, persistent Linux VMs powered by Firecracker. Unlike containers that reset, sprites retain your entire filesystem between sessions. They sleep when idle and wake instantly on demand.

## Key Concepts

- **Persistent storage**: Files, packages, git repos, databases — everything on disk survives sleep/wake cycles
- **Non-persistent**: Running processes and in-memory data are lost during hibernation
- **Always preserved**: Network config, open ports, URL settings
- **Each sprite gets a unique URL** at `https://<name>.sprites.app` (routes to port 8080)
- **Checkpoints**: Snapshot and restore entire filesystem state (~300ms to create)
- **Resources**: Up to 8 CPUs, 16 GB RAM, 100 GB persistent storage per sprite
- **Pre-installed**: Node.js, Python, Go, Ruby, Rust, Elixir, Java, Bun, Deno, Git, curl, vim

## CLI Quick Reference

### Installation
```bash
curl -fsSL https://sprites.dev/install.sh | sh
```

### Authentication
```bash
sprite org auth          # Opens browser for Fly.io auth
sprite auth setup --token <token>  # Use pre-generated token
```

### Sprite Lifecycle
```bash
sprite create <name>              # Create a new sprite
sprite use <name>                 # Set active sprite (creates .sprite file)
sprite list                       # List all sprites
sprite list --prefix "dev-"       # Filter by prefix
sprite destroy -s <name>          # Destroy (irreversible!)
sprite destroy -s <name> -f       # Force destroy, skip confirmation
```

### Running Commands
```bash
sprite exec <command> [args...]   # Run a command (alias: sprite x)
sprite exec -d /path <cmd>        # Set working directory
sprite exec -e KEY=val <cmd>      # Set environment variable
sprite exec -t <cmd>              # Allocate pseudo-TTY
sprite exec -f src:dest <cmd>     # Upload file before exec
sprite console                    # Interactive shell (alias: sprite c)
```

### Sessions (Persistent Background Processes)
```bash
sprite exec -tty npm run dev      # Start detachable session
# Press Ctrl+\ to detach
sprite sessions list              # View running sessions (alias: sprite s ls)
sprite sessions attach <id>       # Reconnect to session
sprite sessions kill <id>         # Terminate session
```

### Checkpoints
```bash
sprite checkpoint create                    # Snapshot current state
sprite checkpoint create --comment "msg"    # With description
sprite checkpoint list                      # List checkpoints
sprite checkpoint info <version-id>         # Show details
sprite checkpoint delete <version-id>       # Soft delete
sprite restore <version-id>                 # Restore (replaces entire filesystem!)
```

### Networking
```bash
sprite url                          # Show sprite URL
sprite url update --auth public     # Make URL public
sprite url update --auth default    # Revert to private (requires Bearer token)
sprite proxy 5432                   # Forward localhost:5432 -> remote 5432
sprite proxy 3001:3000              # Forward localhost:3001 -> remote 3000
sprite proxy 3000 8080 5432         # Forward multiple ports
```

### Services (Auto-restart on Wake)
```bash
sprite-env services create <name> --cmd <binary> --args <args>
# Example:
sprite-env services create my-server --cmd node --args server.js
```

### API Calls
```bash
sprite api <path> [curl-options]    # Authenticated API call
# With -s: path is relative to /v1/sprites/<sprite-name>/
# Without -s: path is relative to /v1/
```

### Utility
```bash
sprite upgrade                       # Upgrade CLI
sprite upgrade --check               # Check for updates
sprite upgrade --channel dev         # Use dev channel
sprite upgrade --version <ver>       # Specific version
```

## Environment Variables

| Variable | Purpose |
|---|---|
| `SPRITE_TOKEN` | API token override |
| `SPRITE_URL` | Direct sprite URL (local/dev) |
| `SPRITES_API_URL` | API URL override (default: `https://api.sprites.dev`) |

## Exit Codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | General error |
| 2 | Command not found |
| 126 | Command cannot execute |
| 127 | Command not found in sprite |
| 128+ | Terminated by signal |

## SDK Usage

For complete SDK reference, see [reference.md](reference.md).

### JavaScript (`@fly/sprites`)
```javascript
import { SpritesClient } from '@fly/sprites';
const client = new SpritesClient(process.env.SPRITE_TOKEN);
const sprite = await client.createSprite('my-sprite');
const result = await sprite.execFile('python', ['-c', "print('hello')"]);
console.log(result.stdout);
await sprite.delete();
```

### Go (`github.com/superfly/sprites-go`)
```go
client := sprites.New(os.Getenv("SPRITE_TOKEN"))
sprite, _ := client.CreateSprite(ctx, "my-sprite", nil)
cmd := sprite.Command("python", "-c", "print('hello')")
output, _ := cmd.Output()
fmt.Println(string(output))
```

### Python (`sprites`)
```bash
pip install sprites
```

## API Base URL

`https://api.sprites.dev` — All requests require `Authorization: Bearer $SPRITE_TOKEN`.

API categories: Sprites, Checkpoints, Exec (WebSocket), Filesystem, Policy, Proxy, Services.

## Common Patterns

### Set up a dev server that survives sleep
```bash
sprite create my-app
sprite use my-app
sprite exec npm install
sprite-env services create web --cmd node --args server.js
sprite url update --auth public
sprite url  # Get the public URL
```

### Checkpoint before risky changes
```bash
sprite checkpoint create --comment "before migration"
sprite exec python migrate.py
# If something goes wrong:
sprite checkpoint list
sprite restore <version-id>
```

### Mount sprite filesystem locally (SSHFS)
```bash
sprite exec sudo apt install -y openssh-server
sprite-env services create sshd --cmd /usr/sbin/sshd
sprite exec mkdir -p .ssh
sprite exec bash -c "echo '$(cat ~/.ssh/id_*.pub)' >> .ssh/authorized_keys"
sprite proxy 2000:22 &
sshfs sprite@localhost: -p 2000 /tmp/sprite-mount
```

## Documentation Links

- Docs: https://docs.sprites.dev/
- Quickstart: https://docs.sprites.dev/quickstart/
- CLI Reference: https://docs.sprites.dev/cli/commands/
- Working with Sprites: https://docs.sprites.dev/working-with-sprites/
- API Reference: https://docs.sprites.dev/api/v001-rc30/
- Go SDK: https://github.com/superfly/sprites-go
- Python SDK: https://github.com/superfly/sprites-py

When the user asks to work with sprites, use the CLI commands via Bash. Always confirm sprite names before destructive operations like `destroy` or `restore`.

If $ARGUMENTS is provided, interpret it as the task to perform (e.g., "create a sprite called my-app", "run tests in the sprite", "checkpoint before deploy").
