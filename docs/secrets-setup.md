# Secrets setup

No credential lives in a shell rc file. Secrets are stored in an Azure Key Vault
and loaded into the environment per project directory with
[direnv](https://direnv.net), only while you are inside that directory.

## The model

- **Store:** one Azure Key Vault. Access uses your own `az login` session, so
  nothing is written to disk except a short-lived cache in a per-user temp dir.
- **Loader:** `direnv`. Each project directory has an `.envrc` (never committed
  unless it is secret-free like the one in this repo). Leaving the directory
  unloads the variables.
- **Helpers:** [`direnv/direnvrc`](direnv/direnvrc) defines `kv_export` and
  `gh_token_export`. Copy it to `~/.config/direnv/direnvrc` on every machine.
- **Prefer no secret at all.** GitHub tools should read `GH_TOKEN` from
  `gh auth token`; Azure tooling should use the interactive `az` session rather
  than a service-principal secret.

## New machine

Linux / WSL:

```sh
sudo apt install direnv          # or: curl -sSL https://direnv.net/install.sh | bash
mkdir -p ~/.config/direnv
cp docs/direnv/direnvrc ~/.config/direnv/direnvrc
printf '[global]\nwarn_timeout = "5s"\n' > ~/.config/direnv/direnv.toml
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
az login && az account set --subscription <subscription-id>
gh auth login
```

macOS: same, with `brew install direnv` instead of apt.

Then, in each project directory that needs variables, create an `.envrc` and
run `direnv allow` once. Examples:

```sh
# A tool that talks to GitHub: reuse the gh CLI login.
gh_token_export                  # exports GH_TOKEN

# An identifier, not a secret: fine to put inline.
export ARM_SUBSCRIPTION_ID=<subscription-id>

# A real secret: fetched from Key Vault, cached for 8 hours.
kv_export SOME_API_KEY some-api-key            # default vault
kv_export OTHER_KEY other-key <vault-name>     # another vault
```

The default vault name is set at the top of `direnvrc` (`KV_VAULT`).

## Add a secret

```sh
az keyvault secret set --vault-name <vault-name> --name some-api-key --value '<value>'
```

then add one `kv_export SOME_API_KEY some-api-key` line to the `.envrc` of the
project that needs it. Secret names may contain letters, digits and dashes.

## Rotate a secret

Set the new value with the same command, then wipe the local cache on each
machine so the next directory entry fetches it again:

```sh
rm -rf "${XDG_RUNTIME_DIR:-$TMPDIR}/kv"
```

## Vault access

The vault uses Azure RBAC. Your user needs the **Key Vault Secrets Officer**
role on the vault (Secrets User is enough for read-only machines):

```sh
az role assignment create --assignee <your-upn> \
  --role "Key Vault Secrets Officer" \
  --scope "$(az keyvault show -n <vault-name> --query id -o tsv)"
```

## Periodic check

Nothing secret should be in shell config, history, CLI caches or project trees.
Run occasionally, expecting no output:

```sh
grep -nE 'TOKEN=|SECRET=|PASSWORD=' ~/.zshrc ~/.zprofile ~/.zshenv ~/.bashrc ~/.profile 2>/dev/null
grep -rlE 'github_pat_|ghp_|gho_' ~/.zsh_history ~/.bash_history ~/.azure ~/.config ~/projects \
  --exclude-dir=target --exclude-dir=node_modules --exclude-dir=.git 2>/dev/null
env | grep -iE 'token|secret|password'
```
