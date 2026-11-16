# sandbox-kits

Docker Sandboxes (`sbx`) kits and reference material.

| Path | What it is |
| ---- | ---------- |
| `spec.yaml` | Mixin kit: .NET 10 SDK, glab, and the Claude Code plugins ponytail and [account-badge](https://github.com/CNSteuerwald/account-badge) |
| `ado-nuget/` | Kit with its own `spec.yaml`: .NET 10 SDK, Node 22 + OpenSpec, Azure Artifacts NuGet auth, and the same plugins |
| `sbxenv.yaml` | `sbx env` file that creates a sandbox from the `ado-nuget` kit (usage is in the file's comments) |
| `Docker Sandbox Cheatsheet.md` | Quick reference for the `sbx` CLI |

Add the root kit to an existing sandbox:

```
sbx kit add <sandbox> git+https://github.com/CNSteuerwald/sandbox-kits.git
```
