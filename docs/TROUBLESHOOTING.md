# Troubleshooting

To help you troubleshoot and debug issues, try adding the `LOG_LEVEL` to your `mcp.json`

Example

```json
{
  "inputs": [
    {
      "id": "ado_org",
      "type": "promptString",
      "description": "Azure DevOps organization name  (e.g. 'contoso')"
    }
  ],
  "servers": {
    "ado": {
      "type": "stdio",
      "command": "mcp-server-azuredevops",
      "args": ["${input:ado_org}"],
      "env": {
        "LOG_LEVEL": "debug"
      }
    }
  }
}
```

## Common MCP Issues

1. **Clearing VS Code Cache**
   If you encounter issues with stale configurations, reload the VS Code window:
   - Press `Ctrl+Shift+P` (or `Cmd+Shift+P` on macOS).
   - Select `Developer: Reload Window`.

   If the issue persists, you can take a more aggressive approach by clearing the following folders:
   - `%APPDATA%\Code\Cache`
   - `%APPDATA%\Code\CachedData`
   - `%APPDATA%\Code\User\workspaceStorage`
   - `%APPDATA%\Code\logs`

   Clear Node Modules Cache
   - `npm cache clean --force`

2. **Server Not Showing Up in Agent Mode**
   Ensure that the `mcp.json` file is correctly configured and includes the appropriate server definitions. Restart your MCP server and reload the VS Code window.

3. **Tools Not Loading in Agent Mode**
   If tools do not appear, click "Add Context" in Agent Mode and ensure all tools starting with `ado_` are selected.

4. **Too Many Tools Selected (Over 128 Limit)**
   Some tools have a default maximum limit of 128 tools. If you exceed this limit, ensure you do not have multiple MCP Servers running. Check both your project's `mcp.json` and your VS Code `settings.json` to confirm that the MCP Server is configured in only one location—not both.

   You can also use [domains](../README.md#options) (`-d`) as a way to limit the number of tools you load for the Azure DevOps MCP Server.

5. **Server exits with code 1 right after start (on-prem, `--authentication sspi`)**
   - **`ELECTRON_RUN_AS_NODE=1` in the server environment**: some hosts (VS Code agent hosts) pass this variable. `node-gyp-build` then looks for an Electron build of `win-sso` and the native module fails to load. The server now clears the variable on startup when it is not running inside Electron. If you run an older build, set `"command": "cmd"` with `"args": ["/d", "/c", "set ELECTRON_RUN_AS_NODE=&& npx -y ..."]`.
   - **Corrupted npx cache** (`npm warn tar TAR_ENTRY_INVALID checksum failure`, `TAR_BAD_ARCHIVE`): stop the MCP servers that run from it (running node processes lock files in the cache), remove `%LOCALAPPDATA%\npm-cache\_npx` and run `npm cache clean --force`. Concurrent first starts of the same `github:` package from several MCP servers or sessions can corrupt the cache, so use the npm package (`@mm6502/azure-devops-mcp`) instead of `github:`.
   - **Slow first start**: the first start installs about 270 dependencies (35 s to 98 s measured on Windows, about 3 s afterwards). If your client gives up before that, run `npx -y @mm6502/azure-devops-mcp@latest --help` once from a terminal (not from the repository clone) to warm the cache, or install globally (`npm i -g @mm6502/azure-devops-mcp`) and use the command `mcp-server-azuredevops`. A global install does not update itself.
   - **`'mcp-server-azuredevops' is not recognized` when running `npx` inside a clone of this repository**: the clone's `package.json` has the same package name, so `npx` uses the local project. Use `node <clone>\dist\index.js` there, or run `npx` from another directory.
   - **Orphaned MCP server processes**: servers from closed sessions can keep running and lock files in `_npx` (check `Get-CimInstance Win32_Process | Where-Object CommandLine -like '*npm-cache\_npx*'`). End them by PID before cleaning the cache.
   - **`--use-system-ca` in `NODE_OPTIONS`** requires Node.js 22.15+ (or 23.8+). Older Node versions fail with `--use-system-ca is not allowed in NODE_OPTIONS`.
   - **Running from a clone** (`node <clone>\dist\index.js`): run `npm ci` first, otherwise Node fails with `ERR_MODULE_NOT_FOUND`.
   - **No logs in the VS Code Output panel**: the server writes logs only to stderr. Run it manually in a terminal to see them, or redirect stderr to a file (`cmd /c node ... 2>%TEMP%\ado-mcp.log`).

## Project-Specific Issues

1. **npm Authentication Issues for Remote Access**
   If you encounter authentication errors:
   - Verify your npm configuration:

     ```pwsh
     npm config get registry
     ```

     It should point to: `https://registry.npmjs.org/`

2. **Dependency Installation Errors**
   If `npm install` fails, verify that you are using Node.js version 20 or higher. You can check your Node.js version with:

   ```pwsh
   node -v
   ```

## Authentication Issues

### Token Authentication via Environment Variables

For automated scenarios or when you want to use a token stored in an environment variable, you can use the `envvar` authentication type:

1. **Set your token in the ADO_MCP_AUTH_TOKEN environment variable:**

   ```bash
   export ADO_MCP_AUTH_TOKEN="your-azure-devops-token"
   ```

2. **Use the envvar authentication type:**

   ```bash
   npx @azure-devops/mcp myorg --authentication envvar
   ```

3. **For MCP configuration files, update your `.vscode/mcp.json`:**
   ```json
   {
     "inputs": [
       {
         "id": "ado_org",
         "type": "promptString",
         "description": "Azure DevOps organization name (e.g. 'contoso')"
       }
     ],
     "servers": {
       "ado": {
         "type": "stdio",
         "command": "npx",
         "args": ["-y", "@azure-devops/mcp", "${input:ado_org}", "--authentication", "envvar"]
       }
     }
   }
   ```

### GitHub Codespaces

Due to limitations of the environment default OAuth option is not available in Codespace.
Make sure you authenticate via

```sh
az login
```

in the terminal before using MCP tools.

And in case there are authorization/access errors when using the tools please check the [Multi-Tenant Authentication Problems guide](#multi-tenant-authentication-problems-when-using-azcli)

### WSL2, SSH, Docker, and Other Headless Environments

The default OAuth interactive authentication requires a browser to complete the login redirect. In headless environments — such as **WSL2**, **remote SSH sessions**, **Docker containers**, and **CI runners** — no browser is available, causing token acquisition to fail silently.

#### Symptoms

- The MCP server starts successfully and reports as "Connected"
- All tool calls fail with:

  ```
  network_error: Network request failed: fetch failed
  ```

- Direct Azure DevOps REST API calls (e.g. via `az devops project list`) work on the same machine with the same credentials

#### Root Cause

The server defaults to `--authentication interactive`, which opens a browser for the OAuth redirect callback. In headless environments the browser redirect has nowhere to go. The token acquisition fails silently, and downstream API calls surface a generic `fetch failed` error instead of a clear authentication failure.

#### Solution

Use one of the non-interactive authentication methods:

**Option 1: Environment variable with a Personal Access Token (recommended for headless)**

1. Create a [Personal Access Token](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate) with the required scopes.

2. Set the token in the `ADO_MCP_AUTH_TOKEN` environment variable:

   ```bash
   export ADO_MCP_AUTH_TOKEN="your-azure-devops-pat"
   ```

3. Start the server with `--authentication envvar`:

   ```bash
   npx -y @azure-devops/mcp myorg --authentication envvar
   ```

   For Claude Code:

   ```bash
   claude mcp add azure-devops -s user \
     -e ADO_MCP_AUTH_TOKEN="your-azure-devops-pat" \
     -- npx -y @azure-devops/mcp myorg --authentication envvar
   ```

**Option 2: Azure CLI authentication**

1. Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) and log in:

   ```bash
   az login
   ```

2. Start the server with `--authentication azcli`:

   ```bash
   npx -y @azure-devops/mcp myorg --authentication azcli
   ```

   > **Note:** If your Azure DevOps organization is in a different tenant than your default `az` CLI tenant, you must also pass `--tenant <tenant-id>`. See the [Multi-Tenant Authentication Problems](#multi-tenant-authentication-problems-when-using-azcli) section below.

### OAuth

Recent switch to OAuth flow is supposed to simplify authentication against ADO APIs and remove additional software dependency.

It is however possible that strict tenant admin policies prevent users from successfully logging in using OAuth flow. In that case consider falling back to AZ CLI.

#### Symptoms

Upon ADO tool execution browser opens a tab/window and after login attempt an error text is displayed:

```
Error occurred: ...
```

#### Solution

Try using Azure login context instead:

1. Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) and **log in**:

   ```sh
   az login
   ```

2. **Configure the MCP server** with the azcli authentication option by updating your `.vscode/mcp.json`.

   ```json
   {
     "inputs": [
       {
         "id": "ado_org",
         "type": "promptString",
         "description": "Azure DevOps organization name (e.g. 'contoso')"
       }
     ],
     "servers": {
       "ado": {
         "type": "stdio",
         "command": "npx",
         "args": ["-y", "@azure-devops/mcp", "${input:ado_org}", "--authentication", "azcli"]
       }
     }
   }
   ```

3. **Restart VS Code** completely to ensure the MCP server picks up the new configuration.

### Multi-Tenant Authentication Problems when using azcli

If you encounter authentication errors like `TF400813: The user 'xxx' is not authorized to access this resource`, you may be experiencing multi-tenant authentication issues.

#### Symptoms

- Azure CLI (`az devops project list`) works fine
- MCP server fails with authorization errors
- You have access to multiple Azure tenants

#### Root Cause

The MCP server may be authenticating with a different tenant than your Azure DevOps organization, especially when you have access to multiple Azure tenants. The MCP server may also be using the Azure Devops Org tenant when the user belongs to a different tenant and is added as a guest user in the Azure DevOps organization.

#### Solution

1. **Identify the correct tenant ID** for your Azure DevOps organization:

   ```pwsh
   az account list
   ```

   Look for the `tenantId` field in the output for the desired tenant (for guest accounts this will be the tenant of your organization and may be different than the Azure Devops Organization tenant).

2. **Configure the MCP server with the tenant ID** by updating your `.vscode/mcp.json`.

   🧨 Installation from Public Feed Configuration:

   ```json
   {
     "inputs": [
       {
         "id": "ado_org",
         "type": "promptString",
         "description": "Azure DevOps organization name (e.g. 'contoso')"
       },
       {
         "id": "ado_tenant",
         "type": "promptString",
         "description": "Azure tenant ID (required for multi-tenant scenarios)"
       }
     ],
     "servers": {
       "ado": {
         "type": "stdio",
         "command": "npx",
         "args": ["-y", "@azure-devops/mcp", "${input:ado_org}", "--authentication", "azcli", "--tenant", "${input:ado_tenant}"]
       }
     }
   }
   ```

   🛠️ Installation from Source Configuration:

   ```json
   {
     "inputs": [
       {
         "id": "ado_org",
         "type": "promptString",
         "description": "Azure DevOps organization name (e.g. 'contoso')"
       },
       {
         "id": "ado_tenant",
         "type": "promptString",
         "description": "Azure tenant ID (required for multi-tenant scenarios)"
       }
     ],
     "servers": {
       "ado": {
         "type": "stdio",
         "command": "mcp-server-azuredevops",
         "args": ["${input:ado_org}", "--tenant", "${input:ado_tenant}"]
       }
     }
   }
   ```

3. **Restart VS Code** completely to ensure the MCP server picks up the new configuration.

4. **When prompted**, enter:
   - Your Azure DevOps organization name
   - The tenant ID from step 1

## Common Errors

1. **Incorrect Organization Name Error**

   ```
   Error fetching projects: Failed to find api location for area: Location id: e81700f7-3be2-46de-8624-2eb35882fcaa
   ```

   **Cause:** This occurs when the Azure DevOps organization name is incorrect or doesn't exist.

   **Solution:** Verify that:
   - The organization name is spelled correctly (case-sensitive)
   - The organization exists and you have access to it
   - You're using just the organization name, not the full URL (e.g., use `contoso` not `https://dev.azure.com/contoso`)
