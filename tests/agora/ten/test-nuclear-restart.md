# Test: Nuclear Restart TEN
## Prompt
I added a new graph to my TEN agent but it's not showing up in the playground. How do I fix this?

## Skill Files Loaded
- `~/.claude/skills/ten/SKILL.md` -- Top-level skill definition with key concepts, critical rules, and restart decision table
- `~/.claude/skills/ten/references/operations.md` -- Docker ops, nuclear restart procedure, log locations, health checks, common errors
- `~/.claude/skills/ten/references/graphs.md` -- Graph structure in property.json, connection types, rebuild_property.py sync, API endpoints, testing workflow
- `~/.claude/skills/ten/references/extensions.md` -- Extension directory structure, manifest.json, addon.py registration (relevant for verifying extension exists)

## Plan

This is a well-documented scenario in the skill. The playground frontend caches the graph list, so adding a new graph to `property.json` requires a "nuclear restart" to make it visible. Here is the full diagnosis and fix procedure:

### Step 1: Verify the graph was actually added to property.json

Confirm your new graph exists in the `ten.predefined_graphs` array inside `tenapp/property.json`:

```bash
sudo docker exec ten_agent_dev bash -c \
  "cat /app/agents/examples/voice-assistant-advanced/tenapp/property.json" \
  | jq '.ten.predefined_graphs[].name'
```

If your graph name does not appear in the output, the graph was not added correctly. Check the JSON structure -- each graph entry must follow this format:

```json
{
  "name": "my_new_graph",
  "auto_start": false,
  "graph": {
    "nodes": [...],
    "connections": [...]
  }
}
```

### Step 2: Check if rebuild_property.py is in sync

If the project uses `rebuild_property.py` to generate `property.json`, your direct edit to `property.json` may get overwritten on the next script run. To avoid this:

1. Update `rebuild_property.py` first to include your new graph definition
2. Run the script to regenerate `property.json`:
   ```bash
   sudo docker exec ten_agent_dev bash -c \
     "cd /app/agents/examples/voice-assistant-advanced/tenapp && \
      python3 rebuild_property.py"
   ```
3. Verify the graph appears after regeneration:
   ```bash
   sudo docker exec ten_agent_dev bash -c \
     "cat /app/agents/examples/voice-assistant-advanced/tenapp/property.json" \
     | jq '.ten.predefined_graphs[].name'
   ```

### Step 3: Verify the extension(s) referenced by the graph exist

Each node's `addon` field must match an actual extension directory under `ten_packages/extension/`. Check that all extensions referenced in your graph are present:

```bash
sudo docker exec ten_agent_dev ls /app/agents/ten_packages/extension/
```

Also verify the extension has the required files (`addon.py`, `extension.py`, `manifest.json`, `property.json`):

```bash
sudo docker exec ten_agent_dev ls /app/agents/ten_packages/extension/<your_extension_name>/
```

### Step 4: Perform the nuclear restart

This is the critical step. The playground frontend caches the graph list, so simply saving `property.json` is not enough -- you must kill all processes and restart. Copy and paste this entire block:

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

What this does:
1. Kills the API server (`bin/api`), the playground (`node`/`bun`), and any stale processes
2. Removes the Next.js dev lock file (prevents lock file errors on restart)
3. Waits 2 seconds for processes to die
4. Starts everything fresh with `task run` (which sets PYTHONPATH correctly -- never run `./bin/api` or `./bin/main` directly)
5. Waits 12 seconds for all services to start (API on port 8080, playground on port 3000, TMAN Designer on port 49483)
6. Runs a health check and lists all available graphs

### Step 5: Verify the graph now appears

The final command in the nuclear restart block (`curl ... /graphs | jq ...`) should output your new graph name. You can also run a full health check:

```bash
echo "=== Health ===" && curl -s http://localhost:8080/health && \
echo -e "\n=== Graphs ===" && curl -s http://localhost:8080/graphs | jq -r '.data[].name' && \
echo -e "\n=== Playground ===" && curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:3000
```

### Step 6: Test the graph via the API (optional)

If you want to verify the graph works before using the playground:

```bash
# Start a session with your new graph
curl -X POST http://localhost:8080/start \
  -H "Content-Type: application/json" \
  -d '{"graph_name": "my_new_graph", "channel_name": "test_channel", "remote_stream_id": 123}'

# Check it is running
curl -s http://localhost:8080/list | jq '.'

# Stop when done
curl -X POST http://localhost:8080/stop \
  -H "Content-Type: application/json" \
  -d '{"channel_name": "test_channel"}'
```

### Step 7: Check logs if the graph still does not appear

If the graph still does not show up after nuclear restart, check the logs for errors:

```bash
# Check for startup errors
sudo docker exec ten_agent_dev tail -200 /tmp/task_run.log | grep -E "(ERROR|Traceback|failed)"

# Check for extension loading errors
sudo docker exec ten_agent_dev tail -200 /tmp/task_run.log | grep -i "extension"
```

Common issues at this stage:
- **"No graphs available"**: The nuclear restart did not complete -- wait longer or re-run
- **Lock file error**: Already handled by the `rm -f` in step 4
- **ModuleNotFoundError**: Python deps need reinstalling (happens after container restarts):
  ```bash
  sudo docker exec ten_agent_dev bash -c \
    "cd /app/agents/examples/voice-assistant-advanced/tenapp && \
     bash scripts/install_python_deps.sh"
  ```
  Then re-run the nuclear restart.
- **"Environment variable X not found"**: If your new graph references env vars with `${env:VAR_NAME}` (required form), ensure they are set in `ai_agents/.env`. After editing `.env`, you must restart the Docker container first, then do nuclear restart.

### Summary of the restart decision table

| What you changed | Nuclear restart needed? |
|---|---|
| Added/removed graphs in property.json | **YES** (this is your case) |
| Changed config/prompts/params in existing graph | No -- just start a new session |
| Changed Python extension code | No -- just start a new session |
| Changed .env file | Container restart first, THEN nuclear restart |

## Assessment
**Rating: 5/5**

The skill is exceptionally well-prepared for this exact question. Key strengths:

1. **Explicit nuclear restart procedure**: The skill provides a copy-paste-ready block of commands that handles the complete restart sequence, including killing processes, removing lock files, restarting with the correct `task run` command, waiting for startup, and verifying with health checks.

2. **Clear restart decision table**: The "When to Restart What" table in both SKILL.md and operations.md immediately identifies that adding/removing graphs requires a nuclear restart because the frontend caches the graph list. This is called out as Critical Rule #2.

3. **Full diagnostic chain**: The skill covers everything needed -- verifying the graph in property.json, keeping rebuild_property.py in sync, checking extension existence, the restart itself, verification via API and health checks, and log locations for troubleshooting.

4. **Common errors table**: The operations.md has a dedicated "No graphs available in playground" entry pointing directly to nuclear restart.

5. **Nothing missing**: For this specific scenario, the skill provides complete coverage. The only minor addition that could help would be explicit mention of checking JSON syntax validity (e.g., running `jq . property.json` to catch parse errors), but the overall procedure is comprehensive and actionable.
