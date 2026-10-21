# figma-restish-cli

A lightweight Figma CLI using [Restish](https://rest.sh/) and Figma's official [OpenAPI spec](https://github.com/figma/rest-api-spec). Covers the **full Figma REST API: 54 operations** — files, nodes, image exports, comments, reactions, versions, projects, components, styles, webhooks, variables and dev resources.

No code generation. No build step. Restish reads the OpenAPI spec at runtime, so new Figma endpoints are available as soon as you refresh the spec.

## Why this exists

Restish v2 sends a stored credential value literally. It has no syntax for reading a token from an environment variable, so the usual setup puts your Figma token in plain text inside `restish.json`. Figma personal access tokens also expire after at most 90 days, and an expired token just starts failing with 401.

This setup gives you:

| Capability | This |
|---|---|
| All 54 Figma REST operations | **Yes**, named subcommands from the spec |
| Token kept out of `restish.json` | **Yes**, injected per call by a 50-line bash wrapper |
| Token from env var or env file | **Yes**, `FIGMA_TOKEN` or `FIGMA_ENV_FILE` |
| Expiry warning | **Yes**, once a day in the last 14 days |
| Clear message on 401/403 | **Yes**, prints the rotation steps, exits 3 |
| Token cost for AI agents | **Low**, JSON in and out, filter with `-f` |

## Setup

### 1. Install Restish (v2.3.0 or later)

```bash
# Linux (amd64)
gh release download --repo rest-sh/restish --pattern 'restish-*-linux-amd64.tar.gz' --dir /tmp
tar -xzf /tmp/restish-*-linux-amd64.tar.gz -C /tmp
mv /tmp/restish ~/.local/bin/restish
chmod +x ~/.local/bin/restish

# Or via Homebrew
brew install restish
```

### 2. Get a Figma personal access token

**Figma → Settings → Security → Personal access tokens → Generate new token.** Tokens last at most 90 days. Scopes:

- File content: read
- File metadata: read
- Comments: read and write
- Current user: read

### 3. Store the token

```bash
mkdir -p ~/.config/figma-cli
cat > ~/.config/figma-cli/env <<'EOF'
FIGMA_TOKEN=figd_your_token_here
FIGMA_TOKEN_EXPIRES=2027-01-31
EOF
chmod 600 ~/.config/figma-cli/env
```

`FIGMA_TOKEN_EXPIRES` is optional; set it to the expiry date Figma showed you and the wrapper warns you before it lapses. Point `FIGMA_ENV_FILE` at a different file if you keep secrets elsewhere, or just export `FIGMA_TOKEN`.

### 4. Configure Restish

```bash
mkdir -p ~/.config/restish
curl -sSL https://raw.githubusercontent.com/figma/rest-api-spec/main/openapi/openapi.yaml \
  -o ~/.config/restish/figma-openapi.yaml
```

Merge the `figma` entry from `apis-template.json` into the `apis` object of `~/.config/restish/restish.json`.

Restish v2 refuses every operation unless a credential is bound to the spec's security scheme. Bind one, and when it asks for the value, enter any placeholder, **not** your real token:

```bash
restish api auth add figma PersonalAccessToken
# choose: api-key, in: header, name: X-Figma-Token, value: placeholder
```

The command needs a terminal. From a script, wrap it: `script -qc "restish api auth add figma PersonalAccessToken" /dev/null`.

### 5. Install the wrapper and verify

```bash
cp figma-cli ~/.local/bin/figma-cli
chmod +x ~/.local/bin/figma-cli
figma-cli get-me -f status   # expect 200
```

## Usage

`figma-cli` takes anything `restish figma` takes. `FILE_KEY` is the segment after `/design/` (or `/file/`) in a Figma URL.

```bash
# Who am I (smoke test)
figma-cli get-me

# Top-level pages of a file
figma-cli get-file FILE_KEY --depth 1 -f 'body.document.children[].name'

# Export nodes as PNG at 2x (ids are positional; image URLs expire after 30 days)
figma-cli get-images FILE_KEY 1:2,1:3 --format png --scale 2

# Read comments
figma-cli get-comments FILE_KEY -f 'body.comments[].{id,message,user:user.handle}'

# Post a comment
echo '{"message":"Looks good"}' | figma-cli post-comment FILE_KEY

# Everything else
restish figma --help
```

## How the wrapper works

1. Loads `FIGMA_TOKEN` from the environment, or from `$FIGMA_ENV_FILE` (default `~/.config/figma-cli/env`).
2. If the token is missing, prints the rotation steps and exits 3.
3. If `FIGMA_TOKEN_EXPIRES` is within 14 days, warns on stderr, at most once a day.
4. Runs `restish figma "$@" -H "X-Figma-Token: $FIGMA_TOKEN"`. A `-H` header overrides the placeholder credential.
5. If Figma answers 401 or 403, prints the rotation steps and exits 3. Any other exit code passes through.

## Token rotation

1. Generate a new token with the same scopes.
2. Replace `FIGMA_TOKEN` and update `FIGMA_TOKEN_EXPIRES` in your env file.
3. `figma-cli get-me -f status` should print 200.
4. Revoke the old token in Figma.

## Known limitations

- **The REST API cannot edit canvas content.** Creating or changing frames, layers or text needs a Figma plugin. This is a Figma API limit, not a tooling gap.
- **Image export URLs expire after 30 days.** Download what you need.
- **Variables endpoints need an Enterprise plan.** Expect 403 otherwise.
- **A 403 can also mean a missing scope**, not only a bad token. Check the scopes before rotating.

## License

MIT
