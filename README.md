# JFrog Agent Skills — publishing and demo guide

This repository holds Cursor-style **agent skills** (each skill is a folder with a `SKILL.md` manifest and optional assets). This README is both a short **introduction to publishing skills to JFrog Artifactory Skills repositories** and a **personal checklist for demoing** the flow end to end—including **signed** publishes using JFrog Evidence keys.

Official reference: [Skills Repositories](https://docs.jfrog.com/artifactory/docs/skills-repositories) (documented as **open beta**).

## What you are publishing

- Artifactory treats each skill as a **zip** at `{slug}/{version}/{slug}-{version}.zip`.
- **Slug** and metadata come from your package; the server indexes **`SKILL.md`** YAML frontmatter (`name`, `description`, `version`, and other supported fields) for the Packages UI and search.
- The API is **ClawHub v1–compatible** for publish, resolve, and download.
- Publish the **entire skill directory** (not only `SKILL.md`) so references, samples, and assets ship together.

Example skill in this repo: [`add-github-build-provenance/`](add-github-build-provenance/) — includes `SKILL.md`, [`references/`](add-github-build-provenance/references/), and [`assets/`](add-github-build-provenance/assets/).

## Prerequisites (platform)

1. **Local Skills repository** in Artifactory: **Administration → Repositories → Create → Local →** package type **Skills** (for example `skills-local`). Requires **Admin** or **Project Admin**.
2. **JFrog CLI** [2.98.0 or newer](https://docs.jfrog.com/artifactory/docs/skills-repositories#configure-jfrog-cli-for-skills): `jf --version`.
3. **Credentials**: Identity token (or username/password) with permission to deploy to that repository.
4. **Signed publish**: [Evidence signing for `jf skills publish`](https://docs.jfrog.com/artifactory/docs/skills-repositories#publish-skills-with-evidence-signing) is documented as an **Enterprise+** capability. Without it, omit `--signing-key` / `--key-alias` and use an unsigned publish if your edition allows it.

## Demo narrative (two minutes)

Use this order when walking someone through the story:

1. **Problem**: Agent skills are reusable instructions; teams need a **private registry** with the same governance mindset as other artifacts.
2. **JFrog answer**: A **Skills** local repo in Artifactory + **`jf skills publish`** (and optional **Evidence** signing) so skills are versioned, discoverable, and attestable.
3. **Live beat**: Show the skill folder → run publish → open **Packages** (or list API) → optionally **`jf skills install`** and mention verification when signing is enabled.

## Prerequisites (signing) — summary

For a **signed** publish you need:

- A **private key** only on machines or CI that run `jf skills publish` (`--signing-key` or environment variable `EVD_SIGNING_KEY_PATH`).
- The matching **public key** registered under **Administration → Security → Keys Management → Public Keys**, with a **case-sensitive alias** that matches **`--key-alias`** / **`EVD_KEY_ALIAS`**.
- Consumers rely on Artifactory-held public keys when installing with evidence verification; see [`jf skills install`](https://docs.jfrog.com/artifactory/docs/skills-repositories#install-skills-with-jfrog-cli) in the Skills docs.

## Cryptographic setup before the first signed publish

JFrog’s **Evidence** onboarding order is documented in [Evidence Setup](https://docs.jfrog.com/governance/docs/evidence-setup): **[Create a Key Pair for Evidence](https://docs.jfrog.com/governance/docs/create-a-key-pair-for-evidence)**, then **[Upload the Public Key to Artifactory](https://docs.jfrog.com/governance/docs/upload-the-public-key-to-artifactory)** (recommended so the platform can verify signatures). The same trust anchor used for `jf evd create` applies to **`jf skills publish`** when signing is enabled.

### 1. Roles and access

- **Who generates keys**: Usually platform or security; demo CI can consume an existing key from a secret store.
- **Admin token**: Registering the public key (CLI default upload or admin flows) typically needs an **admin-capable** identity token; JFrog documents that **basic auth is not supported** for the default upload path on `jf evd generate-key-pair`. Confirm policy with your JFrog admin.
- **CLI versions**: **≥ 2.98.0** for `jf skills publish`; **`jf evd generate-key-pair`** from **2.82.0** — see [Generate Evidence Key Pair CLI](https://docs.jfrog.com/governance/docs/generate-evidence-key-pair-cli).

### 2. Configure JFrog CLI

```shell
jf config add <SERVER-NAME> \
  --url <JFrogPlatformURL> \
  --access-token <ADMIN_OR_DEPLOY_TOKEN> \
  --interactive=false
jf config use <SERVER-NAME>
```

Use a token that can **upload trusted public keys** when you will register the key via CLI or REST.

### 3. Create a key pair for Evidence

Per [Create a Key Pair for Evidence](https://docs.jfrog.com/governance/docs/create-a-key-pair-for-evidence): the **private** key signs evidence; the **public** key verifies it. Supported types include **RSA**, **EC**, and **ED25519**.

#### Option A — OpenSSL (commands from JFrog)

**RSA**

```shell
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

**EC (ECDSA P-256 / `secp256r1`)**

```shell
openssl ecparam -name secp256r1 -genkey -noout -out private.pem
openssl ec -in private.pem -pubout > public.pem
```

**ED25519**

```shell
openssl genpkey -algorithm ed25519 -out private.pem
openssl pkey -in private.pem -pubout -out public.pem
```

**Copy/paste caution**: Prefer piping or CLI copy (for example `pbcopy` on macOS). Avoid hand-copying the private key from the terminal UI—stray characters break the key.

Then continue with **section 4** to upload `public.pem` unless you automate upload via REST.

#### Option B — JFrog CLI (`jf evd generate-key-pair`)

Generates **ECDSA P-256** and, by default, uploads the public key. Details: [Generate Evidence Key Pair CLI](https://docs.jfrog.com/governance/docs/generate-evidence-key-pair-cli).

```shell
jf evd generate-key-pair \
  --key-file-path /secure/directory/for/keys \
  --key-file-name skills-publish-signing \
  --key-alias <CASE_SENSITIVE_ALIAS> \
  --server-id <SERVER-NAME>
```

- **`--key-alias`**: Use the **same** value later as `--key-alias` / `EVD_KEY_ALIAS` on `jf skills publish`. If omitted, the CLI may auto-generate an alias such as `evd-key-YYYYMMDD-HHMMSS`.
- **`--upload-public-key=true`** (default): Requires **admin** context; key appears under **Keys Management → Public Keys**.
- **`--upload-public-key=false`**: Local files only—you **must** complete **section 4** manually or via REST.

With `--upload-public-key=false`, no JFrog login is required to generate files; with default upload, an **admin token** is required.

### 4. Upload the public key to Artifactory

Per [Upload the Public Key to Artifactory](https://docs.jfrog.com/governance/docs/upload-the-public-key-to-artifactory), registering the public key is **recommended**.

If you used `jf evd generate-key-pair` with default upload, confirm the key in **Public Keys** and skip the manual steps if it is already there.

**UI steps (JFrog’s flow)**

1. **Administration** module.
2. **Security → Keys Management**.
3. **Public Keys** tab.
4. **Add Keys**.
5. Enter the **alias** (must match `--key-alias` / `EVD_KEY_ALIAS` when signing).
6. Paste the full public key into **Certificate Key** (complete PEM block, `BEGIN` through `END`).
7. **Add Public Key** — confirm it appears in the table.

**REST**

- `POST https://<JFrogPlatformURL>/artifactory/api/security/keys/trusted`
- JSON body: `alias`, `public_key` (full PEM text).

**Federation**: In federated setups, you may need the public key on **each member** that must verify evidence.

### 5. Map keys to `jf skills publish`

- **`--signing-key`** (or `EVD_SIGNING_KEY_PATH`): path to the **private** key (`private.pem` from OpenSSL, or `<name>.key` from `jf evd generate-key-pair`).
- **`--key-alias`** (or `EVD_KEY_ALIAS`): **exact** alias from Keys Management.

### 6. Protect the private key

- Store in a vault or CI secret; **never** commit to git.
- On runners: restrictive permissions (for example `chmod 600`); for RSA, `openssl rsa -in private.pem -check` is a common sanity check.
- **Rotation**: new pair, new alias, register new public key, migrate publishers, retire old alias when safe.

### 7. Optional smoke test (`jf evd create`)

Before relying on `jf skills publish --signing-key`, validate signing with the same key and alias against any subject your token can target:

```shell
echo '{"test":true}' > /tmp/evidence-predicate.json
jf evd create \
  --subject-repo-path <LOCAL_REPO>/<path-to-existing-test-artifact> \
  --key /secure/directory/for/keys/skills-publish-signing.key \
  --key-alias <CASE_SENSITIVE_ALIAS> \
  --predicate /tmp/evidence-predicate.json \
  --predicate-type https://jfrog.com/evidence/signature/v1
```

Adjust `--subject-repo-path` or use `--package-*` / `--build-*` as appropriate. If this fails, fix paths, alias, or permissions before publishing the skill.

## Publish a skill (CLI, with signing)

From a clone of this repository:

1. **Configure CLI** (if not already done):

```shell
jf config add <SERVER-NAME> \
  --url <JFrogPlatformURL> \
  --access-token <TOKEN> \
  --interactive=false
jf config use <SERVER-NAME>
jf config show
```

2. **Publish** the skill **folder** (path is relative to your machine):

```shell
jf skills publish ./add-github-build-provenance \
  --repo <REPOSITORY_KEY> \
  --signing-key /secure/path/to/private.key \
  --key-alias <YOUR_EVD_KEY_ALIAS>
```

Environment variable equivalents: `EVD_SIGNING_KEY_PATH`, `EVD_KEY_ALIAS`.

3. **Version**: Taken from the `version` field in `SKILL.md` frontmatter unless you pass `--version <semver>`.

4. **Semantic scanning** (JFrog AI Catalog, if enabled on the repo): publish may wait on Xray; see `--auto-delete-on-failure`, `--skip-scan`, and `JFROG_CLI_SKIP_SKILLS_SCAN=true` in the [Skills Repositories](https://docs.jfrog.com/artifactory/docs/skills-repositories) docs.

5. **Verify**: Artifactory **Packages** UI for the Skills repo; optionally `jf skills install <name> <REPOSITORY_KEY> --version latest` and note evidence verification behavior.

**Slug rule** (for API users): slugs should match `^[a-z0-9][a-z0-9-]*$`. The example skill’s `name` `add-github-build-provenance` matches.

## Install and discover (quick reference)

- **Latest SemVer**: `jf skills install <SKILL_NAME> <REPO> --version latest`
- **Specific version**: `jf skills install <SKILL_NAME> <REPO> --version <SKILL_VERSION>`
- **Delete a version**: `jf skills delete <SKILL_NAME> --version <VERSION> --repo <REPO>` (add `--dry-run` to preview)

## REST API (secondary)

Multipart deploy is documented under [Deploy Skills via REST API](https://docs.jfrog.com/artifactory/docs/skills-repositories#deploy-skills-via-rest-api). For **signing**, the primary documented path is **`jf skills publish`**. If you need REST plus attestation, confirm options for your JFrog edition with your administrator.

## Demo checklist (printable)

- [ ] Skills **local** repository exists; you know the **repo key**.
- [ ] `jf --version` ≥ 2.98.0.
- [ ] `jf config` targets the right platform; token can **deploy** to the Skills repo.
- [ ] For signed demo: public key in **Keys Management**; private key available locally; **alias** matches publish flags.
- [ ] Skill folder contains **`SKILL.md`** with valid frontmatter (`name`, `description`, `version`).
- [ ] Run `jf skills publish ./<skill-folder> --repo ...` (+ signing flags if applicable).
- [ ] Show **Packages** (or list API); optionally run **`jf skills install`**.
- [ ] Talking point: **open beta**, ClawHub compatibility, optional **Xray** semantic scanning for AI Catalog customers.

## GitHub Actions: skill publish workflow

The workflow [`.github/workflows/publish-skill.yml`](.github/workflows/publish-skill.yml) publishes `add-github-build-provenance` to the `agentskills` JFrog Project Skills repo (`agentskills-skills-dev-local`) using **`jf skills publish`** with `--version` set to the GitHub **`run_number`** (so the stored zip path uses that value as the skill version segment). It optionally attests GitHub build provenance (currently disabled with `if: false`), creates a Release Bundle v2 (`jf rbc`), attaches Evidence (`jf evd create`), then **promotes** the bundle to the **DEV** environment (`jf rbp` — see [Release Lifecycle Management CLI](https://docs.jfrog.com/governance/docs/release-lifecycle-management-cli)).

### Release Bundle file spec (Skills zip via AQL)

[Using File Specs](https://docs.jfrog.com/artifactory/docs/using-file-specs) allows each `files[]` entry to use **`aql`** with an **`items.find`** object (see also [Release Lifecycle Management CLI — Release Bundle creation](https://docs.jfrog.com/governance/docs/release-lifecycle-management-cli)). A spec may include **only one** AQL query total; combine other sources with `pattern`, `build`, or `package` entries if needed.

Criteria follow [Artifactory Query Language (AQL)](https://docs.jfrog.com/artifactory/docs/artifactory-query-language) on the `items` domain. For a published skill zip, the storage layout is `{slug}/{version}/{slug}-{version}.zip` under the Skills repo ([Skills Repositories](https://docs.jfrog.com/artifactory/docs/skills-repositories#metadata-and-package-identification)).

Example artifact on **tomjfrog**:

`https://tomjfrog.jfrog.io/artifactory/agentskills-skills-dev-local/add-github-build-provenance/0.0.2/add-github-build-provenance-0.0.2.zip`

**Succinct `jf rbc --spec` payload** (equivalent to the MCP-verified AQL `items.find` on `repo`, `path`, `name`):

```json
{
  "files": [
    {
      "aql": {
        "items.find": {
          "repo": "agentskills-skills-dev-local",
          "path": "add-github-build-provenance/0.0.2",
          "name": "add-github-build-provenance-0.0.2.zip"
        }
      }
    }
  ]
}
```

The workflow builds this structure from `SKILL.md` **slug** (`name`), **`github.run_number`** as the published version segment (matching `jf skills publish --version`), and `SKILLS_REPO`. To match several zips in one query, use AQL operators such as `$match` on `path` or `name` inside `items.find` (still a single `aql` block).

### Repository variables (`Settings → Secrets and variables → Actions → Variables`)

| Variable | Purpose |
|----------|---------|
| `JF_URL` | JFrog Platform base URL (e.g. `https://your-instance.jfrog.io`) |
| `JFROG_OIDC_PROVIDER_NAME` | OIDC provider name configured in JFrog for GitHub |
| `JFROG_OIDC_AUDIENCE` | OIDC audience expected by that provider |

### Repository secrets

| Secret | Purpose |
|--------|---------|
| `JF_EVD_PRIVATE_KEY` | Multiline **private key PEM** for Evidence and signed `jf skills publish` (written to a temp file on the runner). |
| `JF_EVD_PUBLIC_KEY_ALIAS` | Case-sensitive alias of the matching **public** key in **Administration → Security → Keys Management** (`--key-alias` for skills publish and `jf evd create`). |
| `JF_RELEASE_BUNDLE_SIGNING_KEY_PATH` | **Mandatory** for `jf rbc` and **`jf rbp`** (`--signing-key`): GPG/RSA key-pair name or path per your **Release Bundles v2** setup ([Release Lifecycle Management CLI](https://docs.jfrog.com/governance/docs/release-lifecycle-management-cli)); independent of EVD signing. Ensure a **DEV** lifecycle environment exists for promotion. |

### Validating the published artifact path

JFrog stores the skill zip at `{slug}/{published-version}/{slug}-{published-version}.zip`. In this workflow, **`published-version` is `github.run_number`** (not only `SKILL.md`’s `version`). The workflow resolves SHA256 via `jf rt curl api/storage/<repo>/<path>`. Confirm the log line **Expected Artifactory storage path** matches the UI.

## Optional next steps

- Bump `version` in `SKILL.md` before each release you want side by side in the repo.
- Automate with GitHub Actions: `setup-jfrog-cli`, private key from **encrypted secrets**, `jf skills publish ... --signing-key ... --key-alias ...`.

## Further reading

- [Use File Specs with JFrog CLI](https://docs.jfrog.com/artifactory/docs/using-file-specs)
- [Skills Repositories](https://docs.jfrog.com/artifactory/docs/skills-repositories)
- [Evidence Setup](https://docs.jfrog.com/governance/docs/evidence-setup)
- [Create a Key Pair for Evidence](https://docs.jfrog.com/governance/docs/create-a-key-pair-for-evidence)
- [Upload the Public Key to Artifactory](https://docs.jfrog.com/governance/docs/upload-the-public-key-to-artifactory)
- [Generate Evidence Key Pair CLI](https://docs.jfrog.com/governance/docs/generate-evidence-key-pair-cli)
