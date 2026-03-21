# Competitor Research — Setup Wizard

Run `/competitor-research setup` once to configure all dependencies. Can be re-run to reconfigure.

## Setup Step 0: Node.js & npm (v20+)

Check if Node.js v20+ and npm are installed (required for MCP servers and npx commands):

```bash
if command -v node >/dev/null 2>&1; then
  NODE_VER=$(node --version | sed 's/v//' | cut -d. -f1)
  if [ "$NODE_VER" -ge 20 ]; then
    echo "Node.js v$(node --version) OK"
  else
    echo "Node.js v$(node --version) is too old (need v20+). Upgrading..."
    curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
    sudo apt-get install -y nodejs
  fi
else
  echo "Node.js not installed. Installing latest LTS..."
  curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
  sudo apt-get install -y nodejs
fi
node --version && npx --version
```

If sudo is unavailable, tell the user to install Node.js v20+ manually (https://nodejs.org).

## Setup Step 1: Environment Detection

```bash
if [ -n "$DISPLAY" ] && xdpyinfo >/dev/null 2>&1; then
  HEADLESS=false
  echo "Graphical display detected"
else
  HEADLESS=true
  echo "Headless server detected — VNC will be needed for browser auth"
fi
```

## Setup Step 2: Virtual Display (headless only)

If headless:
1. Check if `/vnc-service:setup` skill is available in the skills list
2. If available: invoke `/vnc-service:setup` to install and configure the virtual display + VNC
3. If NOT available: install the vnc-service plugin automatically via Bash:
   ```bash
   claude plugin marketplace add 3D-Stories/vnc-service
   claude plugin install vnc-service@vnc-service
   ```
   Then tell the user: "vnc-service plugin installed. Run `/reload-plugins` to activate it."
   (`/reload-plugins` is a session command that must be run by the user — it cannot be
   called from Bash.)
   After reload, verify `/vnc-service:setup` is now in the available skills list.
   Then invoke `/vnc-service:setup`.
4. After setup, invoke `/vnc-service:run` to ensure it's running and get connection info
5. Wait for user to confirm VNC connection before proceeding

## Setup Step 3: NotebookLM CLI

1. Check: `notebooklm --version`
2. If missing, install via **pipx** (not pip — modern Ubuntu/Debian use PEP 668 which blocks
   direct pip install into the system Python):
   ```bash
   # Install the CLI
   pipx install notebooklm-py

   # Inject Playwright (needed for browser-based auth and source management)
   pipx inject notebooklm-py playwright

   # Install Chromium browser for Playwright (must use the venv's playwright binary)
   # Find the actual venv path:
   pipx list  # Look for the notebooklm-py venv location
   # Then install chromium using that path:
   $(pipx environment --value PIPX_LOCAL_VENVS)/notebooklm-py/bin/playwright install chromium

   # Install system dependencies for Chromium (libatk, libcairo, libpango, etc.)
   # Without this, Chromium crashes with "cannot open shared object file" errors
   sudo $(pipx environment --value PIPX_LOCAL_VENVS)/notebooklm-py/bin/playwright install-deps chromium
   ```
   If sudo is unavailable for `install-deps`, the user must install the packages manually.
   Run without sudo to see the list of required packages.

   Then install the Claude Code skill and reload:
   ```bash
   notebooklm skill install
   ```
   ```
   /reload-plugins
   ```

3. Check: `notebooklm status` for auth
4. If not authenticated:
   - If headless: invoke `/vnc-service:run`, then run login with DISPLAY=:99
   - If graphical: run login directly

   **Important:** The `notebooklm login` command opens a browser and waits for ENTER after
   OAuth completes. When running from Claude Code's Bash tool, stdin sends EOF immediately
   which aborts the login. Use this two-step pattern:

   **Step A: Launch login in background so user can interact with the browser:**
   ```bash
   DISPLAY=:99 notebooklm login &
   LOGIN_PID=$!
   echo "Login launched (PID: $LOGIN_PID). Complete OAuth in VNC, then tell me when done."
   ```
   (On graphical desktops, omit `DISPLAY=:99`.)

   Tell the user: "Complete OAuth in the browser (VNC for headless). Let me know when done."

   **Step B: After user confirms OAuth is complete, save the auth:**
   ```bash
   # Kill the background login (it served its purpose — browser profile has the session)
   kill $LOGIN_PID 2>/dev/null
   # Re-run with immediate ENTER to save auth from the persistent browser profile
   echo "" | DISPLAY=:99 notebooklm login
   ```

   The persistent browser profile (`~/.notebooklm/browser_profile`) retains the Google
   session, so the second run opens an already-authenticated browser and immediately saves.

   **If login fails** (e.g., missing dependencies, Chromium crash): kill the background
   process before retrying to avoid orphaned browser instances:
   ```bash
   kill $LOGIN_PID 2>/dev/null
   # Fix the issue, then retry from Step A
   ```

5. Verify: `notebooklm status` shows authenticated

## Setup Step 4: Google AI Mode MCP

1. Check if `mcp__google-ai-search__search_ai` is in the available tools list
2. If missing: install the MCP server via Bash:
   ```bash
   claude mcp add google-ai-search npx google-ai-mode-mcp@latest
   ```
   Note: Do NOT use `xvfb-run` wrapper — use the persistent display from `/vnc-service:setup`.
   The MCP server will use `DISPLAY=:99` from the environment.

   Inform user: "Google AI Mode MCP installed. You'll need to **restart the session** for the
   MCP server to initialize. MCP servers only load at session start."
   **HARD STOP — Do NOT proceed to Step 5 or any later step.** Steps 5-7 require the MCP
   server to be running, which only happens after a session restart. If you skip ahead, the
   user will be asked setup questions out of order and the config file will be written before
   MCP is verified. Tell the user to restart and re-run `/competitor-research setup` — it will
   detect that Steps 1-3 are already complete and resume from Step 4 verification.

3. If present: test with a simple search:
   ```
   mcp__google-ai-search__search_ai(query="test", headless=true)
   ```

4. If the test search fails with **"Chromium distribution 'chrome' is not found"**:
   The MCP server uses **patchright** (a Playwright fork), NOT the Playwright Chromium
   installed in Step 3. These are separate browser installations:
   - Step 3 installed Playwright Chromium for **NotebookLM** (login, source management)
   - This step installs patchright Chrome for the **Google AI Mode MCP** (web search)

   Install it automatically via Bash:
   ```bash
   sudo npx patchright install chrome
   ```
   If sudo is unavailable, tell the user to run that command in a separate terminal.

   Note: `apt install google-chrome-stable` does NOT work — it's not in default Ubuntu repos.
   The patchright installer downloads the correct Chromium build.

   After install, retry the test search.

5. If the test search fails with **CAPTCHA required**:
   - Invoke `/vnc-service:run` for connection info
   - Retry with `headless: false` so browser renders on VNC display
   - User solves CAPTCHA in VNC
   - Verify a headless search works after CAPTCHA is cleared

## Setup Step 5: NotebookLM Notebook Selection

Ask the user:
```
Which NotebookLM notebook should competitor research use?
  1. Use an existing notebook (I'll list them)
  2. Create a new notebook

Which option?
```

**Option 1:** Use `notebooklm list --json` (NOT `notebooklm list` which truncates IDs):
   ```bash
   notebooklm list --json
   ```
   Parse the full UUIDs from the JSON output. Truncated IDs from the table view cause RPC errors.
   User picks a notebook → `notebooklm use <full_uuid>`
**Option 2:** `notebooklm create "Market Research: {project_name}"` → set context

## Setup Step 6: Critique Tool Detection

Check for critique tools in priority order:

1. **BMAD Party Mode** — search for installation (check common paths first, then broad search):
   ```bash
   # Check common locations first (fast)
   for d in ~/bmad ~/BMAD ~/.bmad; do
     [ -f "$d/_bmad/_config/agent-manifest.csv" ] && echo "$d/_bmad" && break
   done
   # Fallback: broader search with depth limit (slower but thorough)
   find ~ -maxdepth 4 -name "agent-manifest.csv" -path "*/_bmad/*" 2>/dev/null | head -1
   ```
   If found: store the **parent directory of `_config/`** as `bmad_path` (e.g., `/home/user/bmad/_bmad`).
   Step 7 will reference the agent manifest at `{bmad_path}/_config/agent-manifest.csv`.

2. **reflexion:critique** — check if available in skills list.
   If available: use as critique tool.

3. **Neither found** — offer to install a critique tool. BMAD is preferred:
   ```
   No critique tool found. Which would you like to install?

   1. BMAD Method (Recommended) — multi-agent party mode with specialized personas
      (analyst, architect, PM, QA, tech writer) for interactive review
   2. Reflexion — multi-judge review with consensus building
   3. Skip — use basic self-critique (always available)
   ```

   **If BMAD chosen:** Requires Node.js >= 20 (verified in Step 0). Install via Bash
   with non-interactive flags (the default TUI hangs in Claude Code):
   ```bash
   npx bmad-method install --directory . --tools claude-code --yes
   ```
   The `--yes` flag accepts defaults and skips interactive prompts.
   `--tools claude-code` selects the Claude Code toolset.
   `--directory .` installs in the current working directory.
   After install, re-run the BMAD detection search to find the installed path.
   Store `bmad_path` in config.

   **If Reflexion chosen:** Install via Bash:
   ```bash
   claude plugin marketplace add NeoLabHQ/context-engineering-kit
   claude plugin install reflexion@NeoLabHQ/context-engineering-kit
   ```
   Then tell user: "Run `/reload-plugins` to activate reflexion."

4. **Self-critique** as last resort — always available, no install needed.

## Setup Step 7: Save Configuration

Save all configuration to `~/.config/competitor-research/config.json`.

The config supports **named profiles** for users researching competitors across multiple
projects or notebooks. Setup creates a `default` profile. Users can add more by re-running
setup or manually editing the config.

```json
{
  "version": 2,
  "created_at": "ISO timestamp",
  "updated_at": "ISO timestamp",
  "headless": true,
  "critique_tool": "bmad|reflexion|self",
  "bmad_path": "/home/user/bmad/_bmad",
  "mcp_installed": true,
  "vnc_configured": true,
  "default_profile": "chorestory",
  "profiles": {
    "chorestory": {
      "notebook_id": "uuid",
      "notebook_title": "ChoreStory Investor Research",
      "project_path": "/home/user/projects/chorestory_business"
    }
  }
}
```

Global settings (headless, critique_tool, mcp_installed, vnc_configured) are shared across
profiles. Per-profile settings are notebook_id, notebook_title, and project_path.

```bash
mkdir -p ~/.config/competitor-research
```

Report:
```
Setup complete!
  ✓ Virtual display (Xvfb :99 + VNC on 5999)
  ✓ NotebookLM (authenticated)
  ✓ Google AI Mode MCP (CAPTCHA cleared)
  ✓ Notebook: {notebook_title}
  ✓ Critique: {bmad|reflexion|self}

Run /competitor-research <name> to start researching.
```
