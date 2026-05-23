# capa_agent_demo

A capability-bound LLM agent harness, in ~400 lines of
[Capa](https://github.com/nelsonduarte/capa-language). Run a
chat-style prompt; the model can call a small set of tools
(read a file, list a directory, fetch a GET-only URL, read the
clock); each tool's authority is statically narrowed by the
type system; the manifest is the audit contract.

**The pitch.** Today's LLM agent harnesses (LangChain, OpenAI
function-calling, MCP servers, etc.) ship tools as arbitrary
Python functions with no permission system. A prompt injection
that says *"now run `rm -rf /`"* succeeds or fails based on
whether the tool happens to call `subprocess.run` somewhere
deep in its body. The blast radius is implicit, dynamic, and
auditable only by reading every line of every tool.

Capa flips this. Each tool's signature names exactly the
capabilities it consumes. The agent loop's signature names the
tool dispatcher's union. The compiler refuses to let a tool
function call `Fs.write` if its signature said `ReadOnlyFs`.
The auditor runs `capa --manifest agent.capa`, reads one
column, and is done.

## The audit moment

```
$ capa --manifest agent.capa | jq -r '.functions[] | "\(.name): \(.declared_capabilities)"' | grep -E '^(tool_|dispatch_tool|run_agent_loop|main):'

tool_read_file:    ["ReadOnlyFs"]
tool_list_dir:     ["ReadOnlyFs"]
tool_get_url:      ["GetOnlyHttp"]
tool_current_time: ["Clock"]
dispatch_tool:     ["Clock", "GetOnlyHttp", "ReadOnlyFs"]
run_agent_loop:    ["Clock", "GetOnlyHttp", "LlmClient", "Logger", "ReadOnlyFs", "Stdio"]
main:              ["Clock", "Env", "Fs", "Stdio", "Unsafe"]
```

Read `run_agent_loop` and you are done. The LLM is in charge of
that function for up to 5 turns; its blast radius is exactly
the columns above. **No `Fs` (write-capable), no `Http` (POST-
capable), no `Net`, no `Db`, no `Unsafe`.** Even total prompt-
injection cannot escape this surface, because the compiler
refuses to type-check a call to an authority the function did
not name.

`main` is the wiring point: it holds `Unsafe` and raw `Fs`,
passes them through factories (`make_readonly_fs`,
`make_get_only_http`, `make_anthropic_client`), and hands the
agent loop only the attenuated versions. Past `main` the
unattenuated authorities are invisible.

## Quick start

```bash
git clone https://github.com/nelsonduarte/capa_agent_demo
cd capa_agent_demo
capa install                    # fetches + verifies the seed libraries
export ANTHROPIC_API_KEY=sk-ant-...
capa --run agent.capa -- "What's the weather in Lisbon today? Use get_url with https://wttr.in/Lisbon?format=3"
```

On Windows the console defaults to cp1252 and weather responses
contain emoji; export `PYTHONIOENCODING=utf-8` (Git Bash) or
`$env:PYTHONIOENCODING = 'utf-8'` (PowerShell) before running.

Sample transcript (real run, 2026-05-23):

```
[user] What's the weather in Lisbon today? Use get_url with https://wttr.in/Lisbon?format=3
[INFO] turn 1/5
[tool call] get_url({"url": "https://wttr.in/Lisbon?format=3"})
[tool result] lisbon: ☀️  +28°C
[INFO] turn 2/5
[llm] The weather in Lisbon today is **sunny** ☀️ with a temperature of **+28°C** (approximately 82°F).
```

## What the demo actually does

A four-tool agent loop:

| Tool             | Capability     | What it does                                          |
| ---------------- | -------------- | ----------------------------------------------------- |
| `read_file`      | `ReadOnlyFs`   | Read a file's contents.                               |
| `list_dir`       | `ReadOnlyFs`   | List a directory's entries.                           |
| `get_url`        | `GetOnlyHttp`  | HTTP GET on an allow-listed host (no POST/PUT/DELETE).|
| `current_time`   | `Clock`        | Return Unix seconds.                                  |

The model loops up to 5 turns: it can issue tool_use blocks,
read tool results, and emit a final `end_turn`. The agent
forwards each tool call through `dispatch_tool`, which is the
narrow waist; that function holds the entire authority union
the loop is allowed to exercise.

The two attenuated wrappers (`attenuated.capa`) are pure-Capa
cap-bearing structs: `ReadOnlyFsImpl { fs: Fs }` holds the
underlying `Fs`, exposes only `read_file` and `list_dir`;
`GetOnlyHttpImpl { http: Http, allowed_hosts }` enforces an
allow-list and forbids non-GET methods at the wrapper boundary.
**Capa's capability discipline is what makes the wrapper
narrow**: a holder of `ReadOnlyFs` cannot call `Fs.write`
because the wrapper interface does not expose it; the analyzer
catches any internal mistake.

## Install via capa.toml

This demo is itself a small library consumer; its
`capa.toml` consumes two seed libraries:

```toml
[dependencies.capa_http]
git = "https://github.com/nelsonduarte/capa_http"
tag = "v0.1.2"
verify_key = "6C1D222D491FB88031E041A536CFB426101AA24B"

[dependencies.capa_log]
git = "https://github.com/nelsonduarte/capa_log"
tag = "v0.1.2"
verify_key = "6C1D222D491FB88031E041A536CFB426101AA24B"
```

`capa install` runs three independent supply-chain checks on
each dep on every install:

1. SHA in `capa.lock` matches the resolved commit;
2. `git verify-tag` against your GPG keyring matches the
   pinned fingerprint;
3. SLSA L2 build-provenance attestation verifies through
   Sigstore Rekor (`gh attestation verify`).

The third layer is implicit; it runs whenever `verify_key` is
set and the repo is on GitHub. See the main Capa repo's
[`docs/packages.md`](https://github.com/nelsonduarte/capa-language/blob/main/docs/packages.md)
for the full model.

## Why this matters

Industry framings of the LLM-tool-use security problem (Simon
Willison's "lethal trifecta", Anthropic's CSP guidance, OpenAI
Assistants v2) all rest on convention: "this tool only reads,
trust me." Capa replaces the convention with a type-system
proof. The capability surface of a tool is its signature; the
capability surface of the agent loop is the union of its
tools' signatures; the SBOM emitter
([`capa --cyclonedx`](https://github.com/nelsonduarte/capa-language))
makes that union part of the artifact's manifest, machine-
checkable in CI.

A regulator (EU CRA, NIS2, NIST SSDF) reading the SBOM for an
AI agent product sees, statically, the agent's full authority
boundary. They do not need to read the agent's source; they do
not need to trust a SOC2 report. The bound is the contract.

## Repo layout

```
capa.toml              # two seed-library deps (capa_http, capa_log)
capa.lock              # locked SHAs + GPG fingerprints
agent.capa             # the agent loop + main()
attenuated.capa        # ReadOnlyFs + GetOnlyHttp wrappers
llm_client.capa        # LlmClient capability + AnthropicClient impl
tools.capa             # 4 tool functions + dispatch_tool + tool_decls
.github/workflows/
  release.yml          # SLSA L2 attestation on v* tag push
SECURITY.md            # publisher key + verification recipes
publisher.asc          # publisher's public GPG key
```

## Limitations (v0.1)

- **Single user prompt per run**. Conversation history does
  not persist across `capa --run` invocations. A REPL-style
  multi-turn shell is an obvious v0.2; the loop already
  supports it, the entry point doesn't expose it.
- **Anthropic Messages API only**. Adding OpenAI or Bedrock
  is one new `LlmClient` implementor; the agent loop and the
  tool dispatcher do not change. The capability surface
  argues against "swap providers in CI" being a real risk.
- **Tool input is a single string field per tool**. Tools that
  need multi-arg schemas would extend `dispatch_tool`'s JSON
  extraction. The point of the demo is the capability claim,
  not a maximalist schema language.
- **`get_url` host allow-list is hard-coded** in `agent.capa`.
  Production would source it from env / config / a Capa
  `Net.restrict_to` call. Demo simplifies for legibility.
- **No streaming**. Anthropic's streaming endpoint is a
  refinement; v0.1 is request-response.

## License

Dual MIT OR Apache-2.0, matching the parent
[capa-language](https://github.com/nelsonduarte/capa-language)
repo. See [`LICENSE-MIT`](LICENSE-MIT) and
[`LICENSE-APACHE`](LICENSE-APACHE).
