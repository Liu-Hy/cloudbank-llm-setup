# `.env` Setup for CloudBank-Funded LLM Access

A self-contained guide to provisioning CloudBank-funded model access and assembling a local `.env`. It is written for a specific project, but only the model lists and `.env` field names are project-specific; the credential setup applies to any ACCESS/CloudBank-funded project that calls these clouds. It routes each model family to a different cloud:

- **GPT → Azure OpenAI**: you create your own resource and key (§2).
- **Claude → AWS Bedrock**: short-term access is self-serve; a durable credential for long unattended jobs is still being arranged with CloudBank (§3). If you've set up Azure, Foundry offers a fully self-serve Claude route on Azure too (§3).
- **Gemini → Google Vertex AI**: you authenticate as yourself via ADC (§4).

Because each user has their own access to the three CloudBank billing accounts, you set up your own Azure and Vertex access directly, with no shared secret to wait for. Finish by assembling `.env` (§5).

## 1. Log in through CloudBank

Always reach each cloud console *through* the CloudBank portal, never a personal Microsoft / Amazon / Google account:

1. Sign in at https://www.cloudbank.org with your **institutional credentials**.
2. **Dashboard → Access CloudBank Billing Accounts**.
3. Click the **login** link for the cloud you're setting up (Azure §2, AWS §3, GCP §4).

The per-cloud sign-in quirks are covered in each section below.

## 2. Azure → GPT

You create your own Azure OpenAI resource, deployments, and key.

### 2a. Open the Azure portal

1. Enter the Azure portal from CloudBank (§1).
2. If a **"Limited or No Access"** pop-up appears, click **Sign out** inside that window, then sign back in with **`<your-username>@cloudbank.org`** (your CloudBank username followed by `@cloudbank.org`). Azure recognizes the domain and routes you to the ACCESS login page.
3. You're in the right place once a subscription named **`access-…`** appears in the portal.

### 2b. Create the Azure OpenAI resource

1. In the top search bar, search for and select **Azure OpenAI**, then click **Create**.
2. The wizard offers two resource types: **Microsoft Foundry** and **Azure OpenAI**. Choose **Azure OpenAI**. In most cases you only need GPT models from Azure, and the Azure OpenAI resource exposes exactly the `…/openai/v1` endpoint you need.
3. **Basics**:
   - **Resource group**: pick **default** (or create a new one and give it any name).
   - **Region**: **East US 2** (broadest GPT availability in the US as of May 2026; check the Azure docs for up-to-date info).
   - **Name**: use **your first name**. This becomes your endpoint hostname, e.g. `https://<firstname>.openai.azure.com`.
4. **Network**: **All networks** (fine for a research workload; tighten later if needed).
5. **Tags**: leave blank.
6. **Review + create → Create**. Provisioning takes ~30 seconds.

> **Optional, anytime:** you can upgrade this resource to a Foundry resource by following the in-portal prompt and choosing a project name, which preserves your API key and endpoint. Not required for GPT models, but it unlocks a larger model catalog and agent tooling. The catalog includes **Claude** over an Anthropic-compatible endpoint (a self-serve alternative to Bedrock; see §3), plus models like Grok and DeepSeek.

### 2c. Deploy the models you need

1. Open your resource and click **Go to Foundry portal**.
2. In the left panel, choose **Deployments** (under **Shared resources**), search for a model, and click **Deploy**. For each model:
   - **Deployment type**: **Global Standard**.
   - **Deployment name**: keep it **identical to the model id** (e.g. `gpt-5.5`, `gpt-5.4`, `gpt-4o-mini`). The `.env` references these exact strings; a mismatch produces a `DeploymentNotFound` 404 with no other hint.
3. Deploy the set used here: `gpt-4o-mini`, `gpt-4o`, `o4-mini`, `gpt-5.5`, `gpt-5.4`, `gpt-5.4-mini`. Each finishes in ~15 seconds and is callable immediately.

### 2d. Copy your key and endpoint

Resource → **Keys and Endpoint** → copy **Endpoint** and **KEY 1** into your `.env`:

```bash
AZURE_OPENAI_API_KEY=<KEY 1>
AZURE_OPENAI_ENDPOINT=https://<your-first-name>.openai.azure.com
AZURE_OPENAI_API_MODE=v1
AZURE_OPENAI_TOKEN_PARAM=max_completion_tokens
AZURE_OPENAI_GPT_4O_MINI_DEPLOYMENT=gpt-4o-mini
AZURE_OPENAI_GPT_4O_DEPLOYMENT=gpt-4o
AZURE_OPENAI_GPT_5_5_DEPLOYMENT=gpt-5.5
AZURE_OPENAI_GPT_5_4_MINI_DEPLOYMENT=gpt-5.4-mini
AZURE_OPENAI_GPT_5_4_DEPLOYMENT=gpt-5.4
AZURE_OPENAI_O4_MINI_DEPLOYMENT=o4-mini
AZURE_OPENAI_REASONING_EFFORT=medium
```

Smoke test (with `.env` exported via `set -a; source .env; set +a`):

```bash
curl -s "$AZURE_OPENAI_ENDPOINT/openai/v1/chat/completions" \
  -H "api-key: $AZURE_OPENAI_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"ping"}],"max_completion_tokens":8}'
```

A 200 with `choices[0].message.content` confirms key + endpoint + deployment name. A 404 `DeploymentNotFound` means a deployment name in `.env` doesn't match what you created.

## 3. AWS Bedrock → Claude

Claude runs on Amazon Bedrock: calls to `bedrock-runtime` (via the AWS CLI or an SDK such as boto3) read AWS credentials from the standard chain (environment variables or `~/.aws`). You need three things: the Anthropic use-case form approved once for the account (§3a), an AWS credential that reaches that chain (§3b), and the `us.*` model ids wired into `.env`. Four account-level quirks, all confirmed by testing:

- The old **Model access** page is retired. Anthropic models are auto-enabled on first invocation, but Anthropic still requires a one-time use-case form, now reached from each model's detail page in the **Model catalog**.
- **Long-lived credentials are not self-serve.** Minting a long-term Bedrock API key calls `iam:CreateUser`, which the CloudBank PowerUser role lacks, and static IAM keys are disabled by default. CloudBank must issue either one (§3b).
- **CloudBank's parent-org SCP denies `global.*` cross-region inference profiles.** Confirmed via Playground (`explicit deny in a service control policy: ...p-tnlm356a`). Use `us.*` profiles instead; the SCP applies to API keys too, so this is not just a console-session limitation. The `.env` block below pins every Claude id to a `us.*` profile.
- **Claude Opus 4.7 is not enabled on new accounts.** AWS gates the current flagship behind a separate access request. Substitute Opus 4.6 (which works out of the box).

**Self-serve alternative: Claude on Azure (Foundry).** If you set up Azure (§2) and upgraded that resource to Foundry (§2b), you can call Claude through Azure instead of Bedrock. This needs no AWS account, Anthropic use-case form, or durable-credential wait, and is billed to the same Azure CloudBank fund as your GPT usage. Foundry serves Claude on the same resource over an Anthropic-compatible Messages API. Deploy the Claude models you need in the Foundry portal (same flow as §2c: **Deployments → Deploy**, **Global Standard**, deployment name identical to the model id, e.g. `claude-sonnet-4-6`, `claude-haiku-4-5`), then call the resource's `/anthropic` endpoint with your **KEY 1** (the key from §2d) in the `x-api-key` header:

```bash
curl -s "https://<resource>.services.ai.azure.com/anthropic/v1/messages" \
  -H "x-api-key: <KEY 1>" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":16,"messages":[{"role":"user","content":"Say hi in one word."}]}'
```

`<resource>` is your Azure resource name, the same one in your `<resource>.openai.azure.com` GPT endpoint. A 200 with a `content[0].text` reply confirms the route. Request and response bodies follow Anthropic's Messages format, so any Anthropic-compatible client works by pointing its base URL at `.../anthropic` and sending these headers. The rest of this section covers the default Bedrock route.

### 3a. Submit the Anthropic use-case form (skip if already done)

1. Enter the AWS console from CloudBank (§1). Region: **us-east-1**.
2. Bedrock console → left nav → **Model catalog** → filter provider = **Anthropic** → click into any Claude model (e.g. *Sonnet 4.6*).
3. On the model detail page, fill the **use-case details** banner:
   - Company: `<your institution>`
   - URL: `https://<your-institution>.edu`
   - Industry: `Education`
   - Intended users: **Internal users** only
   - Description (a few sentences):

     ```text
     Academic research at a university for a peer-reviewed NLP paper. We evaluate Claude alongside other LLM families on public benchmarks within a multi-agent method. Used only by the PI and a small group of academic collaborators. No PII, no public-facing product, no commercial deployment. Outputs are aggregated for analysis and reported in the paper. Funded via NSF ACCESS / CloudBank.
     ```

4. Confirm approval by pinging Sonnet 4.6 in **Test → Playground**. A response means the FTU gate is open account-wide.

### 3b. Get an AWS credential

boto3 reads credentials from the standard chain, so any of the shapes below works once it is present. They differ mainly in lifetime:

| Credential | How to get it | Lifetime | Use for |
|---|---|---|---|
| **Your CloudBank federated session** | sign in through the CloudBank AWS portal (self-serve) | short, tied to the login session | interactive testing; spend is attributed to you |
| **Short-term Bedrock API key** (`AWS_BEARER_TOKEN_BEDROCK`) | Bedrock console → API keys (self-serve) | ≤ 12 h | quick, console-free testing |
| **Long-term Bedrock API key** or **static IAM access keys** | CloudBank issues it on request | long-lived | unattended 24-48 h Slurm jobs |

For quick testing, either self-serve option works immediately, and the federated path keeps per-person spend attribution. The catch is lifetime: a 12-hour credential cannot carry an unattended multi-day job, and the long-lived shapes are not self-serve (the long-term key needs `iam:CreateUser`, which the PowerUser role lacks), so CloudBank has to issue them.

> **Pending: durable credentials for long jobs.** We are still confirming with CloudBank the supported way to obtain a long-lived AWS CLI/SDK credential for this account so 24-48 h Slurm jobs can run without re-authenticating. The account signs in through a SAML federation proxy (`federation-proxy.cloudbank.org`) rather than an IAM Identity Center portal, so `aws configure sso` may not apply. The open question is whether CloudBank will mint a long-term Bedrock API key, issue static IAM access keys, or provide a federation-proxy CLI path with a session long enough for 48-hour jobs. Until this is settled, use a self-serve short-term credential and keep batch runs short enough to finish inside one session.

### 3c. Verify your credential

With the credential in place, verify it from the AWS CLI (a clean per-gate signal).

```bash
# Auth: do exactly one of these:
# (A) IAM access keys:
aws configure --profile cloudbank-bedrock
# Region: us-east-1, output: json. Paste the access key ID + secret.

# (B) Bedrock API key:
export AWS_BEARER_TOKEN_BEDROCK=<paste>
# (and drop --profile cloudbank-bedrock from the commands below)

# Raw Bedrock smoke test:
aws bedrock list-foundation-models --region us-east-1 --profile cloudbank-bedrock | head -30
aws bedrock-runtime converse \
  --region us-east-1 --profile cloudbank-bedrock \
  --model-id us.anthropic.claude-haiku-4-5-20251001-v1:0 \
  --messages '[{"role":"user","content":[{"text":"Say hi in one word."}]}]'
```

Interpret outcomes:

- **JSON response with a Claude reply** → done; the credential can invoke Claude.
- **`AccessDeniedException` mentioning use case / `PutUseCaseForModelAccess`** → FTU not actually approved; redo §3a.
- **`AccessDeniedException` on `bedrock:InvokeModel`** without an SCP reference → policy not attached; ping CloudBank to confirm `AmazonBedrockLimitedAccess` is on the user.
- **`AccessDeniedException` with `explicit deny in a service control policy`** → you invoked a `global.*` profile that CloudBank's parent-org SCP blocks. Switch the call (and the `.env`) to the corresponding `us.*` profile.
- **`ValidationException` on the model id** → the profile string is malformed or doesn't exist in this region. Check the exact spelling in *Bedrock console → Cross-region inference*; if Haiku 4.5 isn't enabled at all, fall back to Haiku 4.0 (`anthropic.claude-haiku-4-0`).

### 3d. Model substitution: Opus 4.7 → Opus 4.6

The Playground test showed `anthropic.claude-opus-4-7` returns "not available for this account / contact AWS Sales", an account-level gate AWS applies to the current flagship. Sonnet 4.6 and Opus 4.6 both invoke cleanly. On a capped research fund, substituting Opus 4.6 is the right call regardless of whether you also file an AWS Sales access request in parallel.

The `.env` block below already reflects this: it sets `BEDROCK_CLAUDE_OPUS_4_6_MODEL=us.anthropic.claude-opus-4-6` (the US cross-region inference profile; `global.*` is denied by the CloudBank SCP). Point any Opus 4.7 references in your own config or code at the 4.6 id to match.

### `.env` block

```bash
CLAUDE_PROVIDER=bedrock
BEDROCK_API_MODE=invoke_model
AWS_REGION=us-east-1
AWS_DEFAULT_REGION=us-east-1

# Set the one credential you have and comment out the rest. Don't leave any of
# these defined-but-blank: botocore treats a blank AWS_BEARER_TOKEN_BEDROCK as a
# real (empty) token and fails instead of falling back to ~/.aws.
AWS_BEARER_TOKEN_BEDROCK=<Bedrock API key, if you have one>
# AWS_ACCESS_KEY_ID=<IAM access key, if CloudBank issued IAM keys>
# AWS_SECRET_ACCESS_KEY=<IAM secret>

BEDROCK_ANTHROPIC_VERSION=bedrock-2023-05-31
BEDROCK_CLAUDE_OPUS_4_6_MODEL=us.anthropic.claude-opus-4-6
BEDROCK_CLAUDE_SONNET_4_6_MODEL=us.anthropic.claude-sonnet-4-6
BEDROCK_CLAUDE_HAIKU_4_5_MODEL=us.anthropic.claude-haiku-4-5-20251001-v1:0
```

`boto3 ≥ 1.39` (or the AWS CLI) auto-detects whichever credential is present (env vars or `~/.aws`).

## 4. Google Vertex AI → Gemini

You authenticate as yourself via **per-user Application Default Credentials (ADC)**, with no shared secret. The two APIs only need to be enabled once on the project (§4a); everyone else just runs the ADC login (§4b).

Vertex AI was rebranded to **Gemini Enterprise Agent Platform** at Google Cloud Next 2026. The API IDs are unchanged (`aiplatform.googleapis.com`, `generativelanguage.googleapis.com`); the console now shows them as **Agent Platform API** + **Gemini API**.

ADC avoids the shared-API-key path, which is effectively closed on these accounts anyway: Google now requires Gemini API keys to be bound to a service account, and CloudBank principals can't grant that service account its role. Authenticating as yourself sidesteps all of it: no service account, no shared secret, no CloudBank ticket.

### 4a. Enable APIs (once per project)

Enter the GCP console from CloudBank (§1). The project picker shows your CloudBank project ID, of the form `access-<allocation>-<id>` (e.g. `access-abc123456-789012`). Then enable both APIs from **APIs & Services → Library** by their post-rebrand console names:

- **Agent Platform API** (identifier `aiplatform.googleapis.com`, the renamed Vertex AI API)
- **Gemini API** (identifier `generativelanguage.googleapis.com`, the renamed Generative Language API)

The top-of-page global console search returns documentation pages with no **Enable** button. Search inside the Library page itself, or open the direct Library URLs and click **Enable**:

- `https://console.cloud.google.com/apis/library/aiplatform.googleapis.com?project=<your-project-id>`
- `https://console.cloud.google.com/apis/library/generativelanguage.googleapis.com?project=<your-project-id>`

### 4b. Each user: one-time ADC login

Make sure you're added to the GCP billing account (**CloudBank portal → fund → GCP billing → People/Access**), then on your laptop:

```bash
gcloud auth application-default login                                       # sign in as the CloudBank-federated identity (cloudbank in the domain, NOT a personal Google account)
gcloud auth application-default set-quota-project access-<allocation>-<id>
gcloud config set project access-<allocation>-<id>
```

**On an SSH-only host (e.g. Delta login node)**, add `--no-browser` to the first command. It prints a `--remote-bootstrap=...` command to copy to a laptop terminal with a browser; sign in there, then paste the returned verification URL back into the remote prompt.

The flow ends with `Credentials saved to file: [~/.config/gcloud/application_default_credentials.json]`. That file is the credential; Google's client libraries discover it automatically through `google.auth.default()`.

### `.env` block

```bash
GEMINI_API_MODE=vertex
GOOGLE_CLOUD_PROJECT=access-<allocation>-<id>
GOOGLE_PROJECT_ID=access-<allocation>-<id>
GOOGLE_CLOUD_LOCATION=global
# Leave GOOGLE_API_KEY and GOOGLE_APPLICATION_CREDENTIALS blank; ADC auto-discovers ~/.config/gcloud/.
GEMINI_2_5_PRO_MODEL=gemini-2.5-pro
GEMINI_2_5_FLASH_MODEL=gemini-2.5-flash
GEMINI_2_5_FLASH_LITE_MODEL=gemini-2.5-flash-lite
GEMINI_SCHEMA_MODE=auto
GEMINI_THINKING_BUDGET=-1
```

### Smoke test

```bash
TOKEN=$(gcloud auth application-default print-access-token)
curl -s -w "\nHTTP %{http_code}\n" -X POST \
  "https://aiplatform.googleapis.com/v1/projects/$GOOGLE_CLOUD_PROJECT/locations/global/publishers/google/models/gemini-2.5-flash:generateContent" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"contents":[{"role":"user","parts":[{"text":"Reply pong"}]}],"generationConfig":{"maxOutputTokens":64,"thinkingConfig":{"thinkingBudget":0}}}'
```

Interpret outcomes:

- **HTTP 200 with `candidates[0].content.parts[0].text`** → end-to-end works.
- **`finishReason: MAX_TOKENS` and empty `parts`** → output cap is smaller than Gemini 2.5's adaptive thinking budget consumed. The `thinkingConfig.thinkingBudget=0` override above sidesteps this for smoke tests; real runs use the adaptive default (`GEMINI_THINKING_BUDGET=-1`) with a larger output-token cap.
- **HTTP 403 `PERMISSION_DENIED`** → your principal lacks `aiplatform.endpoints.predict`. Email `help@cloudbank.org` asking them to grant `roles/aiplatform.user` **to your user principal** (not to a service account, which keeps you on ADC).
- **HTTP 404 `Publisher Model … not found`** → `GOOGLE_CLOUD_LOCATION` and the URL must both use `global`.

### 4c. Slurm / HPC pattern

ADC fits Slurm naturally: shared `$HOME` makes `~/.config/gcloud/application_default_credentials.json` visible on every compute node, so jobs auto-discover the credential without ceremony. On NCSA Delta:

1. Run §4b's `--no-browser` login flow once on the Delta login node.
2. In your sbatch script, after activating your environment, optionally make the credential path explicit:

   ```bash
   export GOOGLE_APPLICATION_CREDENTIALS="$HOME/.config/gcloud/application_default_credentials.json"
   export GOOGLE_CLOUD_QUOTA_PROJECT=access-<allocation>-<id>
   ```

3. (Recommended) Sanity-check ADC at job start so a revoked refresh token fails fast instead of mid-batch:

   ```bash
   python -c "import google.auth, google.auth.transport.requests as r; c,_=google.auth.default(); c.refresh(r.Request())" \
     || { echo "ADC expired; re-run gcloud auth application-default login on the login node" >&2; exit 1; }
   ```

Delta's standard GPU partitions allow outbound HTTPS, which is all `google-auth` needs to refresh tokens and reach `aiplatform.googleapis.com`. On clusters that firewall compute nodes, set `HTTPS_PROXY` in the sbatch script.

## 5. Assemble your `.env`

Keep your own gitignored `.env`. Combine the three blocks above: your Azure values (§2d), your Bedrock credential (§3), and your Vertex project + ADC (§4). A template with all three lives in `.env.example`:

```bash
cp .env.example .env     # then fill in your own values
set -a; source .env; set +a
```

This repo stops at a working `.env`: the code that calls the models is yours to add, and it is a thin layer over each provider's SDK or REST API. Before wiring the `.env` into that code, confirm each provider is live with the self-contained smoke test in its section (Azure §2d, Bedrock §3c, Gemini §4). A green response on all three means the credentials are good.

## 6. Guardrails

1. **Budget alerts** at 25 / 50 / 80 % of your fund in each cloud. Azure: *Cost Management → Budgets*. AWS: *Billing → Budgets*. GCP: *Billing → Budgets & alerts*.
2. **Cache model responses.** Caching responses to disk avoids paying for identical calls twice; sharing that cache across machines extends the savings.
3. **Reserve flagship models for final reported runs.** Use Haiku 4.5 / GPT-4o-mini / Gemini Flash for ablations; Opus 4.6 / GPT-5.5 / Gemini 2.5 Pro for reported results.
4. **Rotate after submission.** Azure: regenerate KEY 1 on your own resource. Bedrock: email CloudBank to delete the credential. Vertex: `gcloud auth application-default revoke`.

## Sources

- Azure OpenAI: [Create a resource](https://learn.microsoft.com/en-us/azure/foundry-classic/openai/how-to/create-resource), [v1 API lifecycle](https://learn.microsoft.com/en-us/azure/foundry/openai/api-version-lifecycle), [Azure OpenAI → Foundry upgrade](https://learn.microsoft.com/en-us/azure/foundry/how-to/upgrade-azure-openai)
- Bedrock: [Generate API key](https://docs.aws.amazon.com/bedrock/latest/userguide/api-keys-generate.html), [Use a Bedrock API key](https://docs.aws.amazon.com/bedrock/latest/userguide/api-keys-use.html), [Identity and access management](https://docs.aws.amazon.com/bedrock/latest/userguide/security-iam.html), [Access Anthropic models](https://repost.aws/knowledge-center/bedrock-access-anthropic-model)
- Vertex AI: [Get an API key](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/start/api-keys?usertype=expressmode), [Authenticate to Vertex AI](https://docs.cloud.google.com/vertex-ai/docs/authentication)
- CloudBank: [Add users to AWS](https://community.cloudbank.org/t/how-do-i-add-users-to-my-aws-account/29) / [Azure](https://community.cloudbank.org/t/how-do-i-add-users-to-my-azure-account/30) / [GCP](https://community.cloudbank.org/t/how-do-i-add-users-to-my-gcp-account/31); [CloudBank vs IAM user](https://community.cloudbank.org/t/difference-between-a-cloudbank-user-and-a-iam-user/111); [Help configuring AWS access key](https://community.cloudbank.org/t/help-configuring-aws-account-access-key/112/4)
