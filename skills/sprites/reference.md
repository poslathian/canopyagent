# Sprites.dev SDK Reference

## JavaScript / TypeScript SDK

**Install:** `npm install @fly/sprites`

### Creating a Client
```javascript
import { SpritesClient } from '@fly/sprites';
const client = new SpritesClient(process.env.SPRITE_TOKEN);
```

### Creating and Managing Sprites
```javascript
const sprite = await client.createSprite('my-sprite');
// Use existing sprite:
const sprite = client.getSprite('my-sprite');
await sprite.delete();
```

### Executing Commands
```javascript
// Run and capture output
const result = await sprite.execFile('python', ['-c', "print('hello')"]);
console.log(result.stdout);

// Stream long-running commands
const cmd = sprite.spawn('bash', ['-c', 'for i in {1..10}; do date +%T; sleep 0.5; done']);
for await (const line of cmd.stdout) {
  process.stdout.write(line);
}
```

### Full Example
```javascript
import { SpritesClient } from '@fly/sprites';

const client = new SpritesClient(process.env.SPRITE_TOKEN);
const sprite = await client.createSprite('my-sprite');

try {
  const result = await sprite.execFile('python', ['-c', "print('hello')"]);
  console.log(result.stdout);

  // Stream output
  const cmd = sprite.spawn('bash', ['-c', 'for i in {1..10}; do date +%T; sleep 0.5; done']);
  for await (const line of cmd.stdout) {
    process.stdout.write(line);
  }
} finally {
  await sprite.delete();
}
```

---

## Go SDK

**Install:** `go get github.com/superfly/sprites-go`

### Creating a Client
```go
import sprites "github.com/superfly/sprites-go"

client := sprites.New(os.Getenv("SPRITE_TOKEN"))
```

### Creating and Managing Sprites
```go
ctx := context.Background()
sprite, err := client.CreateSprite(ctx, "my-sprite", nil)
// Use existing:
sprite := client.Sprite("my-sprite")
// Cleanup:
client.DeleteSprite(ctx, "my-sprite")
```

### Executing Commands
The Go SDK mirrors the standard `exec.Cmd` API:

```go
// Capture output
cmd := sprite.Command("python", "-c", "print('hello')")
output, err := cmd.Output()
fmt.Println(string(output))

// Stream output
cmd = sprite.Command("bash", "-c", "for i in {1..10}; do date +%T; sleep 0.5; done")
stdout, _ := cmd.StdoutPipe()
cmd.Start()
io.Copy(os.Stdout, stdout)
cmd.Wait()
```

### Full Example
```go
package main

import (
    "context"
    "fmt"
    "io"
    "os"
    sprites "github.com/superfly/sprites-go"
)

func main() {
    ctx := context.Background()
    client := sprites.New(os.Getenv("SPRITE_TOKEN"))
    sprite, _ := client.CreateSprite(ctx, "my-sprite", nil)
    defer client.DeleteSprite(ctx, "my-sprite")

    cmd := sprite.Command("python", "-c", "print('hello')")
    output, _ := cmd.Output()
    fmt.Println(string(output))

    cmd = sprite.Command("bash", "-c", "for i in {1..10}; do date +%T; sleep 0.5; done")
    stdout, _ := cmd.StdoutPipe()
    cmd.Start()
    io.Copy(os.Stdout, stdout)
    cmd.Wait()
}
```

---

## Python SDK

**Install:** `pip install sprites`

**GitHub:** https://github.com/superfly/sprites-py

---

## Elixir SDK

```elixir
client = Sprites.new(System.get_env("SPRITE_TOKEN"))
{:ok, sprite} = Sprites.create(client, "my-sprite")

{output, 0} = Sprites.cmd(sprite, "python", ["-c", "print('hello')"])
IO.puts(output)

# Streaming
sprite
|> Sprites.stream("bash", ["-c", "for i in {1..10}; do date +%T; sleep 0.5; done"])
|> Stream.each(&IO.write/1)
|> Stream.run()

Sprites.destroy(sprite)
```

---

## REST API

**Base URL:** `https://api.sprites.dev`

**Auth:** `Authorization: Bearer $SPRITE_TOKEN`

### Endpoint Categories

| Category | Description |
|---|---|
| Sprites | Create, list, update, delete sprites |
| Checkpoints | Capture and restore filesystem snapshots |
| Exec | Run commands via WebSocket |
| Filesystem | Browse and manage files |
| Policy | Control outbound network (DNS filtering) |
| Proxy | Tunnel TCP connections |
| Services | Manage background services |

### Key Endpoints

```
PUT    /v1/sprites/{name}              # Create sprite
GET    /v1/sprites                     # List sprites
GET    /v1/sprites/{name}              # Get sprite
DELETE /v1/sprites/{name}              # Delete sprite
POST   /v1/sprites/{name}/exec        # Execute command (WebSocket)
POST   /v1/sprites/{name}/checkpoints  # Create checkpoint
GET    /v1/sprites/{name}/checkpoints  # List checkpoints
POST   /v1/sprites/{name}/restore     # Restore checkpoint
```

### Using `sprite api` from CLI
```bash
# With active sprite (-s flag or `sprite use`):
sprite api checkpoints              # GET /v1/sprites/<name>/checkpoints
sprite api exec -X POST -d '...'    # POST /v1/sprites/<name>/exec

# Without sprite context:
sprite api sprites                  # GET /v1/sprites
```

### Exec WebSocket Protocol

The exec endpoint uses WebSocket with a binary protocol:
- Commands continue running after disconnect (configurable via `max_run_after_disconnect`)
- Supports TTY and non-TTY modes
- In non-PTY mode: binary messages prefixed with stream identifier byte (stdin/stdout/stderr)
- In PTY mode: raw binary data without prefixes
- Sessions persist across disconnections

---

## Configuration Files

### Global: `~/.sprites/sprites.json`
Contains current selection, API URLs, organizations, stored configs.

### Local: `.sprite` (project directory)
Created by `sprite use <name>`. Stores active org and sprite context for the directory.

---

## Pricing

| Resource | Cost |
|---|---|
| CPU | $0.07/CPU-hour (min 6.25% utilization) |
| Memory | $0.04375/GB-hour (min 0.25 GB) |
| Persistent storage | $0.000027/GB-hour |
| Hot NVMe cache | $0.000683/GB-hour |

New accounts get $30 trial credits.
