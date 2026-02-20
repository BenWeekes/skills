# TEN Framework — Operations

## Environment Setup

### Single .env File

**Only ONE .env file is used**: `ai_agents/.env`

This is loaded by `docker-compose.yml`. All other .env files have been removed.

### Required Environment Variables

```bash
# Log & Server & Worker
LOG_PATH=/tmp/ten_agent
LOG_STDOUT=true
SERVER_PORT=8080
WORKERS_MAX=100

# Agora RTC
AGORA_APP_ID=your_app_id
AGORA_APP_CERTIFICATE=  # Optional

# API Keys (depends on extensions used)
DEEPGRAM_API_KEY=your_key
OPENAI_API_KEY=your_key
ELEVENLABS_TTS_KEY=your_key
ANAM_API_KEY=your_key
```

### API Keys Best Practice

Store keys outside the git repository (e.g., `/home/ubuntu/PERSISTENT_KEYS_CONFIG.md`) to:
- Switch branches without losing keys
- Never accidentally commit secrets

## Docker

### Always Use sudo

```bash
# ❌ WRONG - permission denied
docker exec ten_agent_dev ...

# ✅ CORRECT
sudo docker exec ten_agent_dev ...
```

### First Time Setup

```bash
# Enter container
docker exec -it ten_agent_dev bash

# Build and install (5-8 minutes first time)
cd /app/agents/examples/voice-assistant-advanced
task install
```

### Codespaces Alternative

GitHub Codespaces can be used instead of local Docker — typically starts faster and requires no local setup. See https://theten.ai/docs/ten_agent/setup_development_env/setting_up_development_inside_codespace for details.

### After Container Restart

Python dependencies don't persist across container restarts:

```bash
sudo docker exec ten_agent_dev bash -c \
  "cd /app/agents/examples/voice-assistant-advanced/tenapp && \
   bash scripts/install_python_deps.sh"
```

## Starting the Server

**ALWAYS use `task run`** — never `./bin/api` or `./bin/main` directly:

```bash
sudo docker exec -d ten_agent_dev bash -c \
  "cd /app/agents/examples/voice-assistant-advanced && \
   task run > /tmp/task_run.log 2>&1"
```

### Why task run?

- Sets PYTHONPATH correctly for `ten_runtime` and `ten_ai_base`
- Direct `./bin/main` will fail with Python import errors
- Task commands are documented in `Taskfile.yaml`

### Other Task Commands

```bash
task build          # Rebuild after changing compiled code (TypeScript, Go). NOT needed for Python changes.
task test           # Run all tests
task test-extension EXTENSION=agents/ten_packages/extension/my_ext  # Test specific extension
task test-server    # Test server only
task use AGENT=agents/examples/demo  # Switch to a different agent example
task run-gd-server  # Run graph designer server separately
task lint           # Run linting
task format         # Format code (Black for Python)
task clean          # Clean build artifacts
```

**Note:** `task build` is only needed after changing compiled languages (TypeScript, Go). Python changes are picked up automatically on next session.

### What task run Starts

| Service | Port | Description |
|---------|------|-------------|
| API Server | 8080 | Backend REST API for session management |
| Playground | 3000 | Next.js frontend UI |
| TMAN Designer | 49483 | Graphical editor for extension properties |

### TMAN Designer (localhost:49483)

The TMAN Designer is a visual graph editor that lets you modify agent properties without editing JSON files directly:

1. Open http://localhost:49483
2. Right-click on STT, LLM, TTS, or other extension nodes
3. Open their properties and edit values (API keys, model names, prompts, etc.)
4. Submit changes — the updated agent is available at http://localhost:3000

Useful for quick property changes without editing `property.json` manually.

## Nuclear Restart

When in doubt, nuclear restart. **Required after adding/removing graphs.**

```bash
# Nuclear restart - COPY AND PASTE THIS ENTIRE BLOCK
sudo docker exec ten_agent_dev bash -c "pkill -9 -f 'bin/api'; pkill -9 node; pkill -9 bun"
sudo docker exec ten_agent_dev bash -c "rm -f /app/playground/.next/dev/lock"
sleep 2
sudo docker exec -d ten_agent_dev bash -c \
  "cd /app/agents/examples/voice-assistant-advanced && \
   task run > /tmp/task_run.log 2>&1"
sleep 12
curl -s http://localhost:8080/health && echo " API OK"
curl -s http://localhost:8080/graphs | jq -r '.data[].name'
```

### Playground Prod Mode (Optional)

After nuclear restart, optionally switch playground to prod mode for faster loading:

```bash
# Kill only playground (keep API and gd-server running)
sudo docker exec ten_agent_dev bash -c "kill -9 \$(pgrep -f 'next-server') 2>/dev/null"
sleep 2

# Start prod playground
sudo docker exec -d ten_agent_dev bash -c \
  "cd /app/playground && bun run start > /tmp/playground.log 2>&1"
```

**Note:** Prod mode requires gd-server (tman designer) on port 49483. If graphs don't appear, do full nuclear restart first.

## When to Restart What

| Changed | Container Restart? | Nuclear Restart? | Notes |
|---------|-------------------|------------------|-------|
| **property.json** (add/remove graphs) | No | **YES** | Frontend caches graph list |
| **property.json** (config only, prompts, params) | No | **No** | Loaded per NEW session — just reconnect |
| **rebuild_property.py** (then run script) | No | Only if graphs added/removed | Run script first to regenerate property.json |
| **Python extension code** (.py files) | No | **No** | Loaded per NEW session — just reconnect |
| **.env file** | **YES** | Then yes | Must restart container first |

**NO RESTART NEEDED**: Editing prompts, params in property.json, or Python extension code. Changes apply to new sessions automatically.

## Log Locations

| Log | Location |
|-----|----------|
| All extension logs | `/tmp/task_run.log` |
| Session configs | `/tmp/ten_agent/property-{channel}-{timestamp}.json` |
| Agora logs | `/tmp/agoraapi.log`, `/tmp/agorasdk.log` |

### Log Monitoring

```bash
# Real-time all logs
sudo docker exec ten_agent_dev tail -f /tmp/task_run.log

# Filter by channel
sudo docker exec ten_agent_dev tail -f /tmp/task_run.log | grep --line-buffered "channel_name"

# Filter errors
sudo docker exec ten_agent_dev tail -200 /tmp/task_run.log | grep -E "(ERROR|Traceback)"
```

### Log Configuration

Without this config in `property.json`, `ten_env.log_*()` calls are silent:

```json
{
  "ten": {
    "log": {
      "handlers": [{
        "matchers": [{"level": "debug"}],
        "formatter": {"type": "plain", "colored": false},
        "emitter": {"type": "console", "config": {"stream": "stdout"}}
      }]
    }
  }
}
```

### Quick Health Check

```bash
echo "=== Health ===" && curl -s http://localhost:8080/health && \
echo -e "\n=== Graphs ===" && curl -s http://localhost:8080/graphs | jq -r '.data[].name' && \
echo -e "\n=== Playground ===" && curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:3000
```

## Remote Access

### Cloudflare Tunnel (Quick HTTPS)

```bash
pkill cloudflared
nohup cloudflared tunnel --url http://localhost:3000 > /tmp/cloudflare_tunnel.log 2>&1 &
sleep 5
grep -o 'https://[^[:space:]]*\.trycloudflare\.com' /tmp/cloudflare_tunnel.log | head -1
```

### Nginx Reverse Proxy (Production)

```nginx
server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # API endpoints
    location ~ ^/(health|ping|token|start|stop|graphs|list)(/|$) {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Playground
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## Production Deployment

### Docker Image Release

Build a production Docker image (run **outside** any container):

```bash
cd ai_agents
docker build -f agents/examples/<example-name>/Dockerfile -t example-app .
docker run --rm -it --env-file .env -p 3000:3000 example-app
```

### Split Deployment (Vercel/Netlify + Backend)

For hosting frontend and backend separately:

1. **Backend**: Run on any container platform (VM with Docker, Fly.io, Render, ECS, Cloud Run). Use the Docker image above and expose port `8080`.
2. **Frontend**: Deploy `ai_agents/agents/examples/<example>/frontend` to Vercel or Netlify. Run `pnpm install && pnpm build` (or `bun install && bun run build`).
3. **Environment**: Set `AGENT_SERVER_URL` on the frontend to point to the backend URL. Add any `NEXT_PUBLIC_*` keys the UI needs.
4. **CORS**: Ensure the backend accepts requests from the frontend origin — via open CORS or the built-in proxy middleware.

## Common Errors — Quick Fixes

| Error | Fix |
|-------|-----|
| "No graphs available" in playground | Nuclear restart (graphs cached by frontend) |
| "502 Bad Gateway" | Start server with `task run` |
| Lock file error | `sudo docker exec ten_agent_dev bash -c "rm -f /app/playground/.next/dev/lock"` then nuclear restart |
| Port 3000/8080 in use | `sudo docker exec ten_agent_dev bash -c "pkill -9 -f 'bin/api'; pkill -9 node; pkill -9 bun"` |
| ModuleNotFoundError | Reinstall Python deps after container restart |
| "Environment variable X not found" | Container restart required after .env changes |

### Zombie Worker Cleanup

Workers run inside the container, not on host:

```bash
# Find zombie workers
sudo docker exec ten_agent_dev ps aux | grep 'bin/main' | grep -v grep

# Kill by PID (replace <PID> with actual PID from above)
sudo docker exec ten_agent_dev kill -9 <PID>
```

## Pre-Commit

### Format Python Code

Required to pass CI:

```bash
sudo docker exec ten_agent_dev bash -c \
  "cd /app/agents && black --line-length 80 \
  --exclude 'third_party/|http_server_python/|ten_packages/system' \
  ten_packages/extension"
```

### Pre-Commit Hook Setup

Create `.git/hooks/pre-commit` to auto-check formatting and prevent API key commits:

```bash
cat > /home/ubuntu/ten-framework/.git/hooks/pre-commit << 'HOOK'
#!/bin/bash
# Pre-commit hook: check for API keys and black formatting

# Check for API keys in staged files
if git diff --cached --name-only | xargs grep -l -E "(API_KEY|api_key).*=.*[A-Za-z0-9]{20,}" 2>/dev/null; then
    echo "ERROR: Potential API key found in staged files!"
    exit 1
fi

# Check black formatting for Python extension files
staged_py_files=$(git diff --cached --name-only --diff-filter=ACM | grep -E "^ai_agents/agents/ten_packages/extension/.*\.py$" | grep -v "third_party/\|http_server_python/\|ten_packages/system")

if [ -n "$staged_py_files" ]; then
    if command -v docker &> /dev/null && sudo docker ps -q -f name=ten_agent_dev &> /dev/null; then
        unformatted=$(echo "$staged_py_files" | xargs -I {} sudo docker exec ten_agent_dev bash -c "cd /app && black --check --line-length 80 {} 2>&1" 2>/dev/null | grep "would reformat" || true)
    elif command -v black &> /dev/null; then
        unformatted=$(echo "$staged_py_files" | xargs black --check --line-length 80 2>&1 | grep "would reformat" || true)
    else
        echo "WARNING: black not available, skipping format check"
        unformatted=""
    fi

    if [ -n "$unformatted" ]; then
        echo "ERROR: Python files need black formatting!"
        echo "$unformatted"
        echo "Run: sudo docker exec ten_agent_dev bash -c 'cd /app && black --line-length 80 agents/ten_packages/extension'"
        exit 1
    fi
fi
exit 0
HOOK
chmod +x /home/ubuntu/ten-framework/.git/hooks/pre-commit
```

### Commit Message Rules

- Subject must be lowercase
- Valid types: `build`, `chore`, `ci`, `docs`, `feat`, `fix`, `perf`, `refactor`, `revert`, `style`, `test`
- Body lines ≤100 characters

```bash
# Example
fix: correct import statements in heygen extension
```

## Official Documentation

- https://theten.ai/docs — TEN Framework docs
- https://docs.agora.io/en/ten-framework/overview/product-overview — Agora's TEN docs
