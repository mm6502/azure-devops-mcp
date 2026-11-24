# Azure DevOps MCP Server (on-prem fork)

MCP server that gives AI agents access to Azure DevOps. This is a fork of [microsoft/azure-devops-mcp](https://github.com/microsoft/azure-devops-mcp) that adds support for **on-premises Azure DevOps Server / TFS** with Windows single sign-on (SSPI) or a personal access token. It is not affiliated with or endorsed by Microsoft. Upstream targets the cloud service and is not merged automatically, because it would replace the API version 7.1 that on-prem servers need.

Package: [`@mm6502/azure-devops-mcp`](https://www.npmjs.com/package/@mm6502/azure-devops-mcp) · Licence: [MIT](./LICENSE.md)

## Quick start

Requirements: Node.js 22.15 or newer. SSPI needs Windows on a domain-joined machine and a server with Negotiate/NTLM enabled.

Add the server to your MCP client configuration (for example `.mcp.json` or `.vscode/mcp.json`). Replace the URL with your collection URL:

```json
{
  "mcpServers": {
    "ado": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@mm6502/azure-devops-mcp@latest", "https://your-server/tfs/YourCollection"],
      "env": { "NODE_OPTIONS": "--use-system-ca" }
    }
  }
}
```

VS Code's own `mcp.json` uses the key `servers` instead of `mcpServers`; the rest is the same.

On Windows with an on-prem URL, SSPI is selected automatically and no credentials are needed. `NODE_OPTIONS=--use-system-ca` makes Node trust the Windows certificate store (internal CA).

For a personal access token add `"--authentication", "pat"` to `args` and set `PERSONAL_ACCESS_TOKEN` in `env` to the base64 encoding of `<email>:<pat>` (the e-mail can be any non-empty string, only the token part is used). Do not commit tokens.

### Installing and updating

- **`@latest`** (above): every start checks the registry (a fraction of a second) and picks up new releases. The first start after a release installs about 270 dependencies, which takes 35 s to 98 s on Windows; later starts take a few seconds.
- **Pinned version** (`@mm6502/azure-devops-mcp@1.0.1`): no surprise updates. Change the number to update, and use it if a release misbehaves.
- **Global install** (`npm i -g @mm6502/azure-devops-mcp`, command `mcp-server-azuredevops`): instant start, but it never updates itself.
- `npx` does not work inside a clone of this repository (same package name); see [Development](#development).

## Options

| Option                        | Meaning                                                                                                                                                                                      |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `<organization>`              | Collection URL for on-prem (`https://server/tfs/Collection`) or organization name for the cloud service.                                                                                     |
| `-a, --authentication <type>` | `sspi` (on-prem, Windows), `pat`, `env`/`envvar`, `azcli`, `interactive` (cloud). On-prem URLs on Windows default to `sspi`.                                                                 |
| `-d, --domains <list>`        | Load only some tool groups (default `all`): `core`, `work`, `work-items`, `repositories`, `search`, `test-plans`, `wiki`, `pipelines`, `advanced-security`, `mcp-apps`. Keep `core` enabled. |
| `-t, --tenant <id>`           | Tenant for `interactive` and `azcli` (cloud).                                                                                                                                                |
| `--allow-untrusted-cert`      | Disable TLS verification. Insecure, last resort. Prefer `NODE_OPTIONS=--use-system-ca` or `NODE_EXTRA_CA_CERTS=<ca-bundle.pem>`.                                                             |
| `--allow-http`                | Allow `http://` URLs (credentials travel in cleartext).                                                                                                                                      |

Example that loads only work item tools: `"args": ["-y", "@mm6502/azure-devops-mcp@latest", "https://your-server/tfs/YourCollection", "-d", "core", "work", "work-items"]`.

Default project and team (so tools skip the selection prompts) come from environment variables in `env`: `ado_mcp_project` and `ado_mcp_team`.

Logs go to stderr only. Set `LOG_LEVEL` to `debug` in `env` for details; see [troubleshooting](./docs/TROUBLESHOOTING.md) for where to find them.

Supported on-prem versions: Azure DevOps Server 2020 and newer (API 6.0+, requests use 7.1). The list of tools is in [docs/TOOLSET.md](./docs/TOOLSET.md); example prompts are in [docs/EXAMPLES.md](./docs/EXAMPLES.md).

## Development

```bash
npm ci
npm run build
```

To run the server from the clone, create a `.mcp.json` in the repository root (it is listed in `.gitignore`, so your collection URL never gets committed):

```json
{
  "mcpServers": {
    "ado": {
      "type": "stdio",
      "command": "node",
      "args": ["./dist/index.js", "https://your-server/tfs/YourCollection", "--authentication", "sspi"],
      "tools": ["*"],
      "env": {
        "LOG_LEVEL": "debug",
        "NODE_OPTIONS": "--use-system-ca"
      }
    }
  }
}
```

Run `npm run build` again after changing the source. The tracked [mcp.json](./mcp.json) is an alternative that starts the globally installed command `mcp-server-azuredevops` and asks for the collection URL.

Useful commands: `npm test`, `npm run validate-tools`, `npm run eslint-fix`, `npm run format`, `npm run inspect` (MCP Inspector). See [CONTRIBUTING.md](./CONTRIBUTING.md) and [AGENTS.md](./AGENTS.md).

## Documentation

- [Troubleshooting](./docs/TROUBLESHOOTING.md): start-up problems, logs, cache, certificates.
- [Fork management](./docs/solutions/best-practices/fork-management-github-install-2026-06-10.md): branches, releases, installation.
- [Rebasing onto upstream](./docs/solutions/conventions/rebasing-on-prem-fork-onto-upstream.md): what to keep and what to take.
- `docs/GETTINGSTARTED.md`, `docs/FAQ.md`, `docs/HOWTO.md`: upstream documents for the cloud service. They still name the upstream package `@azure-devops/mcp`.

Microsoft, Azure DevOps and related names are trademarks of Microsoft. Their use here only describes compatibility.
