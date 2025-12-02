# 触发器机制


## 通用说明

## 命中检验

## 工具调用前检验


## 工具调用后缓存


## end-session处理




防护栏/触发器配置（尽管在 Phase 8 中完全处理）将有占位符或初始规则，以确保模块添加运行所需的同步脚本（用于更新图和注册表），并且删除或多模块更改会被标记。我们将验证尝试在不遵循程序的情况下删除模块会触发警报。








## 参考内容：
本章节的内容是hook机制的灵感来源，你可以参考以下内容


### 概述
以下内容是仅供参考的hook机制概述：

```markdown

    # Hooks

    Claude Code hooks that enable skill auto-activation, file tracking, and validation.


    | Hook | Type | Essential? | Customization |
    |------|------|-----------|---------------|
    | skill-activation-prompt | UserPromptSubmit | ✅ YES | ✅ None needed |
    | post-tool-use-tracker | PostToolUse | ✅ YES | ✅ None needed |
    | tsc-check | Stop | ⚠️ Optional | ⚠️ Heavy - monorepo only |
    | trigger-build-resolver | Stop | ⚠️ Optional | ⚠️ Heavy - monorepo only |
    | error-handling-reminder | Stop | ⚠️ Optional | ⚠️ Moderate |
    | stop-build-check-enhanced | Stop | ⚠️ Optional | ⚠️ Moderate |

    **Start with the two essential hooks** - they enable skill auto-activation and work out of the box.


    ---

    ## What Are Hooks?

    Hooks are scripts that run at specific points in Claude's workflow:
    - **UserPromptSubmit**: When user submits a prompt
    - **PreToolUse**: Before a tool executes  
    - **PostToolUse**: After a tool completes
    - **Stop**: When user requests to stop

    **Key insight:** Hooks can modify prompts, block actions, and track state - enabling features Claude can't do alone.

    ---

    ## Essential Hooks (Start Here)

    ### skill-activation-prompt (UserPromptSubmit)

    **Purpose:** Automatically suggests relevant skills based on user prompts and file context

    **How it works:**
    1. Reads `skill-rules.json`
    2. Matches user prompt against trigger patterns
    3. Checks which files user is working with
    4. Injects skill suggestions into Claude's context

    **Why it's essential:** This is THE hook that makes skills auto-activate.

    **Integration:**
    ```
    # Copy both files
    cp skill-activation-prompt.sh your-project/.claude/hooks/
    cp skill-activation-prompt.ts your-project/.claude/hooks/

    # Make executable
    chmod +x your-project/.claude/hooks/skill-activation-prompt.sh

    # Install dependencies
    cd your-project/.claude/hooks
    npm install
    ```


    **Add to settings.json:**
    ```json
    {
    "hooks": {
        "UserPromptSubmit": [
        {
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
            }
            ]
        }
        ]
    }
    }
    ```

    **Customization:** ✅ None needed - reads skill-rules.json automatically

    ---

    ### post-tool-use-tracker (PostToolUse)

    **Purpose:** Tracks file changes to maintain context across sessions

    **How it works:**
    1. Monitors Edit/Write/MultiEdit tool calls
    2. Records which files were modified
    3. Creates cache for context management
    4. Auto-detects project structure (frontend, backend, packages, etc.)

    **Why it's essential:** Helps Claude understand what parts of your codebase are active.

    **Integration:**
        ```bash
        # Copy file
        cp post-tool-use-tracker.sh your-project/.claude/hooks/

        # Make executable
        chmod +x your-project/.claude/hooks/post-tool-use-tracker.sh
        ```

    **Add to settings.json:**
    ```json
    {
    "hooks": {
        "PostToolUse": [
        {
            "matcher": "Edit|MultiEdit|Write",
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
            }
            ]
        }
        ]
    }
    }
    ```

    **Customization:** ✅ None needed - auto-detects structure

    ---

    ## Optional Hooks (Require Customization)

    ### tsc-check (Stop)

    **Purpose:** TypeScript compilation check when user stops

    **⚠️ WARNING:** Configured for multi-service monorepo structure

    **Integration:**

    **First, determine if this is right for you:**
    - ✅ Use if: Multi-service TypeScript monorepo
    - ❌ Skip if: Single-service project or different build setup

    **If using:**
    1. Copy tsc-check.sh
    2. **EDIT the service detection (line ~28):**
    ``` bash
        # Replace example services with YOUR services:
        case "$repo" in
        api|web|auth|payments|...)  # ← Your actual services
    ```
    3. Test manually before adding to settings.json

    **Customization:** ⚠️⚠️⚠️ Heavy

    ---

    ### trigger-build-resolver (Stop)

    **Purpose:** Auto-launches build-error-resolver agent when compilation fails

    **Depends on:** tsc-check hook working correctly

    **Customization:** ✅ None (but tsc-check must work first)

    ---

    ## For Claude Code

    **When setting up hooks for a user:**

    1. **Read [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)** first
    2. **Always start with the two essential hooks**
    3. **Ask before adding Stop hooks** - they can block if misconfigured  
    4. **Verify after setup:**
    ``` bash
    ls -la .claude/hooks/*.sh | grep rwx
    ```

```


### 集成说明

以下内容是仅供参考的hook机制集成说明：

``` markdown
    ### Essential Hooks (Always Safe to Copy)

    #### skill-activation-prompt (UserPromptSubmit)

    **Purpose:** Auto-suggests skills based on user prompts

    **Integration (NO customization needed):**

    ```bash
    # Copy both files
    cp showcase/.claude/hooks/skill-activation-prompt.sh \\
    $CLAUDE_PROJECT_DIR/.claude/hooks/
    cp showcase/.claude/hooks/skill-activation-prompt.ts \\
    $CLAUDE_PROJECT_DIR/.claude/hooks/

    # Make executable
    chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh

    # Install dependencies if needed
    if [ -f "showcase/.claude/hooks/package.json" ]; then
    cp showcase/.claude/hooks/package.json \\
        $CLAUDE_PROJECT_DIR/.claude/hooks/
    cd $CLAUDE_PROJECT_DIR/.claude/hooks && npm install
    fi
    ```

    **Add to settings.json:**
    ```json
    {
    "hooks": {
        "UserPromptSubmit": [
        {
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
            }
            ]
        }
        ]
    }
    }
    ```

    **This hook is FULLY GENERIC** - works anywhere, no customization needed!

    #### post-tool-use-tracker (PostToolUse)

    **Purpose:** Tracks file changes for context management

    **Integration (NO customization needed):**

    ```bash
    # Copy file
    cp showcase/.claude/hooks/post-tool-use-tracker.sh \\
    $CLAUDE_PROJECT_DIR/.claude/hooks/

    # Make executable
    chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh
    ```

    **Add to settings.json:**
    ```json
    {
    "hooks": {
        "PostToolUse": [
        {
            "matcher": "Edit|MultiEdit|Write",
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
            }
            ]
        }
        ]
    }
    }
    ```

    **This hook is FULLY GENERIC** - auto-detects project structure!

    ---

    ### Optional Hooks (Require Heavy Customization)

    #### tsc-check.sh and trigger-build-resolver.sh (Stop hooks)

    ⚠️ **WARNING:** These hooks are configured for a specific multi-service monorepo structure.

    **Before integrating, ask:**
    1. "Do you have a monorepo with multiple TypeScript services?"
    2. "What are your service directory names?"
    3. "Where are your tsconfig.json files located?"

    **For SIMPLE projects (single service):**
    - **RECOMMEND SKIPPING** these hooks
    - They're overkill for single-service projects
    - User can run `tsc --noEmit` manually instead

    **For COMPLEX projects (multi-service monorepo):**

    1. Copy the files
    2. **MUST EDIT** tsc-check.sh - find this section:
    ```bash
    case "$repo" in
        email|exports|form|frontend|projects|uploads|users|utilities|events|database)
            echo "$repo"
            return 0
            ;;
    esac
    ```

    3. Replace with USER'S actual service names:
    ```bash
    case "$repo" in
        api|web|auth|payments|notifications)  # ← User's services
            echo "$repo"
            return 0
            ;;
    esac
    ```

    4. Test manually before adding to settings.json:
    ```bash
    ./.claude/hooks/tsc-check.sh
    ```

    **IMPORTANT:** If this hook fails, it will block Stop events. Only add if you're sure it works for their setup.

    ---

```

---

### 配置说明

以下内容是用以参考的配置说明（仅供参考）。
``` markdown
    # Hooks Configuration Guide

    This guide explains how to configure and customize the hooks system for your project.

    ## Quick Start Configuration

    ### 1. Register Hooks in .claude/settings.json

    Create or update `.claude/settings.json` in your project root:

    ```json
    {
    "hooks": {
        "UserPromptSubmit": [
        {
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
            }
            ]
        }
        ],
        "PostToolUse": [
        {
            "matcher": "Edit|MultiEdit|Write",
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
            }
            ]
        }
        ],
        "Stop": [
        {
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-prettier-formatter.sh"
            },
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-build-check-enhanced.sh"
            },
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/error-handling-reminder.sh"
            }
            ]
        }
        ]
    }
    }
    ```

    ### 2. Install Dependencies

    ```bash
    cd .claude/hooks
    npm install
    ```

    ### 3. Set Execute Permissions

    ```bash
    chmod +x .claude/hooks/*.sh
    ```

    ## Customization Options

    ### Project Structure Detection

    By default, hooks detect these directory patterns:

    **Frontend:** `frontend/`, `client/`, `web/`, `app/`, `ui/`
    **Backend:** `backend/`, `server/`, `api/`, `src/`, `services/`
    **Database:** `database/`, `prisma/`, `migrations/`
    **Monorepo:** `packages/*`, `examples/*`

    #### Adding Custom Directory Patterns

    Edit `.claude/hooks/post-tool-use-tracker.sh`, function `detect_repo()`:

    ```bash
    case "$repo" in
        # Add your custom directories here
        my-custom-service)
            echo "$repo"
            ;;
        admin-panel)
            echo "$repo"
            ;;
        # ... existing patterns
    esac
    ```

    ### Build Command Detection

    The hooks auto-detect build commands based on:
    1. Presence of `package.json` with "build" script
    2. Package manager (pnpm > npm > yarn)
    3. Special cases (Prisma schemas)

    #### Customizing Build Commands

    Edit `.claude/hooks/post-tool-use-tracker.sh`, function `get_build_command()`:

    ```bash
    # Add custom build logic
    if [[ "$repo" == "my-service" ]]; then
        echo "cd $repo_path && make build"
        return
    fi
    ```

    ### TypeScript Configuration

    Hooks automatically detect:
    - `tsconfig.json` for standard TypeScript projects
    - `tsconfig.app.json` for Vite/React projects

    #### Custom TypeScript Configs

    Edit `.claude/hooks/post-tool-use-tracker.sh`, function `get_tsc_command()`:

    ```bash
    if [[ "$repo" == "my-service" ]]; then
        echo "cd $repo_path && npx tsc --project tsconfig.build.json --noEmit"
        return
    fi
    ```

    ### Prettier Configuration

    The prettier hook searches for configs in this order:
    1. Current file directory (walking upward)
    2. Project root
    3. Falls back to Prettier defaults

    #### Custom Prettier Config Search

    Edit `.claude/hooks/stop-prettier-formatter.sh`, function `get_prettier_config()`:

    ```bash
    # Add custom config locations
    if [[ -f "$project_root/config/.prettierrc" ]]; then
        echo "$project_root/config/.prettierrc"
        return
    fi
    ```

    ### Error Handling Reminders

    Configure file category detection in `.claude/hooks/error-handling-reminder.ts`:

    ```typescript
    function getFileCategory(filePath: string): 'backend' | 'frontend' | 'database' | 'other' {
        // Add custom patterns
        if (filePath.includes('/my-custom-dir/')) return 'backend';
        // ... existing patterns
    }
    ```

    ### Error Threshold Configuration

    Change when to recommend the auto-error-resolver agent.

    Edit `.claude/hooks/stop-build-check-enhanced.sh`:

    ```bash
    # Default is 5 errors - change to your preference
    if [[ $total_errors -ge 10 ]]; then  # Now requires 10+ errors
        # Recommend agent
    fi
    ```

    ## Environment Variables

    ### Global Environment Variables

    Set in your shell profile (`.bashrc`, `.zshrc`, etc.):

    ```bash
    # Disable error handling reminders
    export SKIP_ERROR_REMINDER=1

    # Custom project directory (if not using default)
    export CLAUDE_PROJECT_DIR=/path/to/your/project
    ```

    ### Per-Session Environment Variables

    Set before starting Claude Code:

    ```bash
    SKIP_ERROR_REMINDER=1 claude-code
    ```

    ## Hook Execution Order

    Stop hooks run in the order specified in `settings.json`:

    ```json
    "Stop": [
    {
        "hooks": [
        { "command": "...formatter.sh" },    // Runs FIRST
        { "command": "...build-check.sh" },  // Runs SECOND
        { "command": "...reminder.sh" }      // Runs THIRD
        ]
    }
    ]
    ```

    **Why this order matters:**
    1. Format files first (clean code)
    2. Then check for errors
    3. Finally show reminders

    ## Selective Hook Enabling

    You don't need all hooks. Choose what works for your project:

    ### Minimal Setup (Skill Activation Only)

    ```json
    {
    "hooks": {
        "UserPromptSubmit": [
        {
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
            }
            ]
        }
        ]
    }
    }
    ```

    ### Build Checking Only (No Formatting)

    ```json
    {
    "hooks": {
        "PostToolUse": [
        {
            "matcher": "Edit|MultiEdit|Write",
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
            }
            ]
        }
        ],
        "Stop": [
        {
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-build-check-enhanced.sh"
            }
            ]
        }
        ]
    }
    }
    ```

    ### Formatting Only (No Build Checking)

    ```json
    {
    "hooks": {
        "PostToolUse": [
        {
            "matcher": "Edit|MultiEdit|Write",
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
            }
            ]
        }
        ],
        "Stop": [
        {
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-prettier-formatter.sh"
            }
            ]
        }
        ]
    }
    }
    ```

    ## Cache Management

    ### Cache Location

    ```
    $CLAUDE_PROJECT_DIR/.claude/tsc-cache/[session_id]/
    ```

    ### Manual Cache Cleanup

    ```bash
    # Remove all cached data
    rm -rf $CLAUDE_PROJECT_DIR/.claude/tsc-cache/*

    # Remove specific session
    rm -rf $CLAUDE_PROJECT_DIR/.claude/tsc-cache/[session-id]
    ```

    ### Automatic Cleanup

    The build-check hook automatically cleans up session cache on successful builds.

    ## Troubleshooting Configuration

    ### Hook Not Executing

    1. **Check registration:** Verify hook is in `.claude/settings.json`
    2. **Check permissions:** Run `chmod +x .claude/hooks/*.sh`
    3. **Check path:** Ensure `$CLAUDE_PROJECT_DIR` is set correctly
    4. **Check TypeScript:** Run `cd .claude/hooks && npx tsc` to check for errors

    ### False Positive Detections

    **Issue:** Hook triggers for files it shouldn't

    **Solution:** Add skip conditions in the relevant hook:

    ```bash
    # In post-tool-use-tracker.sh
    if [[ "$file_path" =~ /generated/ ]]; then
        exit 0  # Skip generated files
    fi
    ```

    ### Performance Issues

    **Issue:** Hooks are slow

    **Solutions:**
    1. Limit TypeScript checks to changed files only
    2. Use faster package managers (pnpm > npm)
    3. Add more skip conditions
    4. Disable Prettier for large files

    ```bash
    # Skip large files in stop-prettier-formatter.sh
    file_size=$(wc -c < "$file" 2>/dev/null || echo 0)
    if [[ $file_size -gt 100000 ]]; then  # Skip files > 100KB
        continue
    fi
    ```

    ### Debugging Hooks

    Add debug output to any hook:

    ```bash
    # At the top of the hook script
    set -x  # Enable debug mode

    # Or add specific debug lines
    echo "DEBUG: file_path=$file_path" >&2
    echo "DEBUG: repo=$repo" >&2
    ```

    View hook execution in Claude Code's logs.

    ## Advanced Configuration

    ### Custom Hook Event Handlers

    You can create your own hooks for other events:

    ```json
    {
    "hooks": {
        "PreToolUse": [
        {
            "matcher": "Bash",
            "hooks": [
            {
                "type": "command",
                "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/my-custom-bash-guard.sh"
            }
            ]
        }
        ]
    }
    }
    ```

    ### Monorepo Configuration

    For monorepos with multiple packages:

    ```bash
    # In post-tool-use-tracker.sh, detect_repo()
    case "$repo" in
        packages)
            # Get the package name
            local package=$(echo "$relative_path" | cut -d'/' -f2)
            if [[ -n "$package" ]]; then
                echo "packages/$package"
            else
                echo "$repo"
            fi
            ;;
    esac
    ```

    ### Docker/Container Projects

    If your build commands need to run in containers:

    ```bash
    # In post-tool-use-tracker.sh, get_build_command()
    if [[ "$repo" == "api" ]]; then
        echo "docker-compose exec api npm run build"
        return
    fi
    ```


```