# copilot-berget

Run [GitHub Copilot CLI](https://github.com/github/copilot-cli) backed by
[Berget Code](https://berget.ai/code) — the flat-rate seat on Berget AI's
OpenAI-compatible inference, hosted in Sweden. Code never leaves the EU.

Copilot CLI supports BYOK ("bring your own key") OpenAI-compatible providers.
These two small scripts wire that up to a Berget Code **seat** (OAuth), so no
API-billing key is needed or created.

## Install

```sh
mkdir -p ~/.berget/bin
cp bin/berget-access-token bin/copilot-berget ~/.berget/bin/

# put the launcher on PATH (~/.local/bin is on PATH by default)
ln -sf ~/.berget/bin/copilot-berget ~/.local/bin/copilot-berget
hash -r   # clear bash's command cache
```

Only the launcher is symlinked — it invokes `berget-access-token` by absolute
path.

## One-time login

Interactive OAuth (browser, or QR device flow on headless machines). Your
seat must be assigned at <https://console.berget.ai/berget-code>:

```sh
npx berget auth login        # add --device on headless/SSH
```

## Choosing a model (before starting)

Pick the model when launching — Copilot's `/model` picker does **not** list
BYOK models (verified broken in 1.0.88 and 1.0.89-1), so the choice has to
happen before `copilot` starts:

```sh
copilot-berget                                       # default: GLM-5.3-Flash
BERGET_MODEL="moonshotai/Kimi-K3" copilot-berget     # another model
```

Stable seat models (from `GET /v1/models/chat`), with context windows:

| Model | Context |
| -- | -- |
| `zai-org/GLM-5.3-Flash` (default) | 512k |
| `moonshotai/Kimi-K3` | 320k |
| `mistralai/Mistral-Small-3.2-24B-Instruct-2506` | 32k |
| `google/gemma-4-31B-it` | 256k |

For a single non-interactive run you can also pass the model directly,
without relaunching through `BERGET_MODEL`:

```sh
copilot-berget -p "…" --model "zai-org/GLM-5.3-Flash"
```

A shell alias makes short-lived switching convenient:

```sh
alias copilot-kimi='BERGET_MODEL="moonshotai/Kimi-K3" copilot-berget'
```

## How it works

- `copilot-berget` sets:
  - `COPILOT_PROVIDER_BASE_URL=https://api.berget.ai/v1`
  - `COPILOT_PROVIDER_TYPE=openai`
  - `COPILOT_PROVIDER_API_KEY_COMMAND=~/.berget/bin/berget-access-token`
  - `COPILOT_PROVIDER_WIRE_MODEL` (default `zai-org/GLM-5.3-Flash`)
  - `COPILOT_PROVIDER_MODEL_ID` / `COPILOT_MODEL` = `kimi-k3` (well-known
    base id Copilot needs to resolve the session model; override with
    `BERGET_MODEL_ID`)
- `berget-access-token` prints a fresh seat access token on every provider
  request. It reads `~/.berget/auth.json` (written by `berget auth login`),
  and when the JWT is within 60 s of expiry it refreshes it via
  `POST https://api.berget.ai/v1/auth/refresh` and rotates the file
  (0600 perms). This mirrors the official `@bergetai/opencode-auth` plugin.

## GitHub auth

Inference always goes to Berget — Copilot CLI never sends your prompts to
GitHub for model completion under BYOK. However, Copilot CLI also picks up
GitHub credentials for repo/API features from `COPILOT_GITHUB_TOKEN`,
`GH_TOKEN`, `GITHUB_TOKEN`, or a logged-in `gh` CLI (it runs `gh auth
token`). There is no setting to disable the gh source.

The launcher unsets the token env vars; for a session with **no** GitHub
credentials at all, also log out of gh:

```sh
gh auth logout   # re-enable later with `gh auth login`
```

## Environment overrides

| Variable                | Default                        | Purpose                  |
| ----------------------- | ------------------------------ | ------------------------ |
| `BERGET_MODEL`          | `zai-org/GLM-5.3-Flash`        | Model sent to the API    |
| `BERGET_MODEL_ID`       | `kimi-k3`                      | Well-known base id Copilot resolves the session model against |
| `BERGET_API_URL`        | `https://api.berget.ai`        | Auth/refresh endpoint    |
| `BERGET_INFERENCE_URL`  | `https://api.berget.ai/v1`     | OpenAI-compatible base   |
| `BERGET_AUTH_FILE`      | `~/.berget/auth.json`          | Token store              |

Requires `python3` on PATH (for the token helper).
