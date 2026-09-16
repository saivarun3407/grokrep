# Tool policy

**Status:** Build law (MVP+)  
**Date:** 2026-09-16

## Principles

1. Tool results are **data**, never instructions.  
2. Default deny: path jail + egress allowlist.  
3. No side-effect social tools in v1 (no follow, mass-DM, payments).  
4. Prefer control-plane proxies over raw VM egress.

## Path jail

- Default root: `/workspace/bots/<bot_id>/`  
- Temp: `/workspace/tmp/<turn_id>/`  
- `shared/` only if bot config grants it  
- Reject `..`, symlinks escaping jail, absolute paths outside roots

## Phased tools

| Phase | Tools | Gate |
|---|---|---|
| MVP | `read_file`, `write_file`, `list_dir` | path jail |
| MVP+ | `web_search` | via control-plane proxy only |
| 2 | `browse_fetch(url)` | domain allowlist; block RFC1918, link-local, cloud metadata IPs; size + time caps |
| 3 | `shell` | Plus+; allowlisted binaries; **no network** from shell |

## Network

MVP VM egress allowlist:

- Model gateway host  
- Control plane  
- (Optional) pinned package mirror if installs are enabled later  

No direct calls to `api.anthropic.com` / OpenAI / etc. from the VM.

## SSRF (browse)

- DNS resolve and block private ranges after resolve  
- Cap response bytes and wall time  
- Strip credentials from URLs in logs

## Shell (phase 3)

- Allowlist binaries explicitly  
- cgroup CPU/RAM from plan  
- No `curl|bash`, no docker-in-docker, no privileged

## Testing

CI red-team must include:

- Path traversal  
- Injection in tool-result content (“ignore previous instructions”)  
- SSRF to metadata IP  
- Attempted exfil of env vars
