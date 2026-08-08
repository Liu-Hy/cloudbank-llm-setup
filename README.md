# Setting Up CloudBank-Funded LLM APIs

> **Note:** CloudBank and the cloud platforms behind it change quickly. This guide is accurate as of **August 7, 2026**. If a step no longer matches what you see, please [open an issue](https://github.com/Liu-Hy/cloudbank-llm-setup/issues).

CloudBank gives NSF-funded projects access to commercial clouds (Azure, Google Cloud, AWS), and those clouds host frontier LLM APIs. Getting from a CloudBank login to a working API call involves more pitfalls than you would expect. This guide walks the whole path and ends with a working `.env` file. It applies to any project funded through CloudBank and ACCESS (the NSF allocation program that CloudBank serves); the `.env` field names match one example project, so rename them to whatever your own code expects.

Once you are on a fund (Section 1), you have your own login to its cloud billing accounts and can set all of this up yourself. There is no shared secret to wait for. We route each model family to the platform where it is easiest to use:

| Models | Platform | Where |
|---|---|---|
| GPT | Azure (Microsoft Foundry) | [Section 2](#2-gpt-on-azure) |
| Claude, plus Grok, DeepSeek, and many other catalog models | Azure (Microsoft Foundry), on the same resource and key as GPT | [Section 3](#3-claude-on-azure-foundry) |
| Gemini | Google Cloud (Vertex AI) | [Section 4](#4-gemini-on-google-cloud-vertex-ai) |

**What about AWS Bedrock?** Bedrock also serves Claude, and an earlier version of this guide treated it as the default Claude route. We now discourage it, for reasons we learned the hard way:

- **The newest Claude models are blocked.** AWS gates the flagship tier behind a "contact AWS Sales" request. When we tested in May 2026, that meant the then-flagship Claude Opus 4.7; we re-checked on August 7, 2026, and the current flagship Claude Opus 5 is blocked the same way.
- **Credentials are short-lived.** Self-serve Bedrock API keys expire after at most 12 hours, so they cannot carry an unattended multi-day job. Long-lived credentials are available only by request to CloudBank, per project.
- **A form gates first use.** Claude calls fail until Anthropic's use-case form has been submitted and approved on the account.
- **The rest of the catalog is thin.** Beyond Anthropic models, Bedrock offers relatively few models, and mostly older versions.

Foundry has none of these problems: it is fully self-serve with a single key, and its catalog carries the latest GPT and Claude models alongside many other proprietary and open-source ones. Foundry does not serve Gemini, which is why Gemini goes through Google Cloud. In case Bedrock improves later, the complete setup we validated is preserved in [Appendix A](#appendix-a-claude-on-aws-bedrock-not-recommended).

## 1. Log in through CloudBank

One prerequisite: the fund owner, typically your PI, must add you to the fund before any of this works. If the billing-accounts page below comes up empty, ask them to add you; CloudBank has short how-tos for [Azure](https://community.cloudbank.org/t/how-do-i-add-users-to-my-azure-account/30), [GCP](https://community.cloudbank.org/t/how-do-i-add-users-to-my-gcp-account/31), and [AWS](https://community.cloudbank.org/t/how-do-i-add-users-to-my-aws-account/29).

Once you are on the fund, always enter each cloud console through the CloudBank portal, never with a personal Microsoft, Google, or Amazon account:

1. Sign in at https://www.cloudbank.org with your **institutional credentials**.
2. Go to **Dashboard → Access CloudBank Billing Accounts**.
3. Click the **login** link for the cloud you are setting up (Azure for Sections 2 and 3, GCP for Section 4, AWS only if you end up using Appendix A).

Each console has its own sign-in quirks; they are covered where they come up.

## 2. GPT on Azure

You will create your own Azure OpenAI resource, deploy the GPT models you need, and copy out a key. Nothing here requires waiting on another person.

### 2a. Open the Azure portal

1. Enter the Azure portal from CloudBank (Section 1).
2. If a **"Limited or No Access"** pop-up appears, or you do not see any subscription, you were probably signed in automatically with a wrong account. **Sign out**, then sign back in as **`<your-username>@cloudbank.org`** (your CloudBank username followed by `@cloudbank.org`). Azure recognizes this domain and forwards you to the ACCESS login page; sign in there with the same institutional credentials you used at cloudbank.org.
3. You are in the right place once a subscription named **`access-...`** appears in the portal.

### 2b. Create the resource

1. In the top search bar, search for and select **Azure OpenAI**, then click **Create**.
2. The wizard offers two resource types, **Microsoft Foundry** and **Azure OpenAI**. Choose **Azure OpenAI**. In Section 3 you will upgrade this resource to Foundry, which unlocks Claude and the wider catalog without changing your key or endpoint; this two-step path is the one we tested.
3. On the **Basics** tab:
   - **Resource group**: pick **default**, or create a new one with any name.
   - **Region**: **East US 2**. It had the broadest GPT availability in the US when we set up (May 2026); check the Azure docs if you want to confirm it is still the best choice.
   - **Name**: something short, such as your first name. The name becomes your endpoint hostname, for example `https://<your-first-name>.openai.azure.com`, so it must be unique across Azure; append a few digits if your first choice is taken.
4. **Network**: keep **All networks**. That is fine for a research workload; tighten it later if you need to.
5. **Tags**: leave blank.
6. Click **Review + create**, then **Create**. Provisioning takes about 30 seconds.

### 2c. Deploy the GPT models you need

A model is not callable until you create a deployment for it.

1. Open your resource and click **Go to Foundry portal**. (The Foundry portal manages plain Azure OpenAI resources too; the upgrade in Section 3 is a separate step.)
2. In the left panel, choose **Deployments** (under **Shared resources**), search for a model, and click **Deploy**. For each model:
   - **Deployment type**: **Global Standard**.
   - **Deployment name**: keep it **identical to the model id** (for example `gpt-5.5`, `gpt-5.4`, `gpt-4o-mini`). Your code passes this string as the model name, and a mismatch produces a `DeploymentNotFound` 404 with no other hint, which is confusing to debug.
3. Deploy whatever your project needs. As one example, a project of ours uses `gpt-4o-mini`, `gpt-4o`, `o4-mini`, `gpt-5.5`, `gpt-5.4`, and `gpt-5.4-mini`. Each deployment finishes in about 15 seconds and is callable immediately.

### 2d. Copy your key and endpoint

In the Azure portal, open your resource, go to **Keys and Endpoint**, and copy **Endpoint** and **KEY 1** into your `.env`:

```bash
AZURE_OPENAI_API_KEY=<KEY 1>
AZURE_OPENAI_ENDPOINT=https://<your-first-name>.openai.azure.com
```

Because each deployment is named after its model id (Section 2c), your code can use model ids directly, and there is nothing else to record. A few optional client settings (API mode `v1`, token-limit parameter `max_completion_tokens`, reasoning effort) live in [`.env.example`](.env.example).

Now the smoke test. Export your `.env` first (`set -a; source .env; set +a`), then run the call below, replacing `gpt-4o-mini` with any deployment name you created:

```bash
curl -s "$AZURE_OPENAI_ENDPOINT/openai/v1/chat/completions" \
  -H "api-key: $AZURE_OPENAI_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"ping"}],"max_completion_tokens":8}'
```

A 200 response with `choices[0].message.content` confirms your key, endpoint, and deployment name all work. A 404 `DeploymentNotFound` means a deployment name does not match what you created in Section 2c.

## 3. Claude on Azure (Foundry)

Claude runs on the same Azure resource you just set up. Foundry serves it over an Anthropic-compatible Messages API, bills it to the same Azure CloudBank fund as your GPT usage, and needs no extra account, form, or key. The catalog stays current, so the newest Claude models, including the flagship tier that Bedrock refuses to serve, are available here without any special approval.

### 3a. Upgrade your resource to Foundry

The plain Azure OpenAI resource from Section 2 exposes only OpenAI models. Upgrading it to a Foundry resource unlocks the rest of the catalog (Claude, Grok, DeepSeek, and more) plus Foundry's agent tooling:

1. Open your resource's overview page in the Azure portal and find the banner **Want to try the latest industry models and Agents?**. Click **Get started**. (The same banner appears on the resource overview page in the Foundry portal. If it is missing or the upgrade fails, Microsoft's [upgrade guide](https://learn.microsoft.com/en-us/azure/foundry/how-to/upgrade-azure-openai) lists the prerequisites and a template-based alternative.)
2. Enter a name for your first project when asked; a project is just a folder that organizes your work in Foundry.
3. The upgrade preserves your resource name, API key, and endpoint, so everything from Section 2 keeps working. You can tell it succeeded when Claude models become deployable in the next step.

### 3b. Deploy the Claude models you need

Same flow as GPT (Section 2c): in the Foundry portal, go to **Deployments → Deploy**, pick **Global Standard**, and name each deployment **identically to the model id**, for example `claude-sonnet-4-6` or `claude-haiku-4-5`. Your first Claude deployment also asks you to accept the Azure Marketplace terms (**Agree and Proceed**), a click-through with no separate signup. New Claude models show up in the same catalog as they are released; search for "claude" to see what is available, or check Microsoft's [Claude in Foundry guide](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/how-to/use-foundry-models-claude) for the current model list and code samples.

### 3c. Smoke test and `.env` block

Call the resource's `/anthropic` endpoint with the **KEY 1** you copied in Section 2d, passed in the `x-api-key` header:

```bash
curl -s "https://<resource>.services.ai.azure.com/anthropic/v1/messages" \
  -H "x-api-key: <KEY 1>" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":16,"messages":[{"role":"user","content":"Say hi in one word."}]}'
```

`<resource>` is your Azure resource name, the same one that appears in your `<resource>.openai.azure.com` GPT endpoint. A 200 response with a `content[0].text` reply confirms the route.

Requests and responses follow Anthropic's Messages format exactly, so any Anthropic-compatible client works: point its base URL at `https://<resource>.services.ai.azure.com/anthropic` and send the same headers. In your `.env`:

```bash
CLAUDE_PROVIDER=foundry
AZURE_ANTHROPIC_ENDPOINT=https://<resource>.services.ai.azure.com/anthropic
AZURE_ANTHROPIC_API_KEY=<KEY 1, same as AZURE_OPENAI_API_KEY>
AZURE_ANTHROPIC_CLAUDE_SONNET_4_6_MODEL=claude-sonnet-4-6
AZURE_ANTHROPIC_CLAUDE_HAIKU_4_5_MODEL=claude-haiku-4-5
```

## 4. Gemini on Google Cloud (Vertex AI)

Foundry does not serve Gemini, so Gemini goes through Google Cloud. You authenticate as yourself with **Application Default Credentials (ADC)**: a one-time `gcloud` login stores a credential file that Google's client libraries discover automatically, so there is no API key and no shared secret. The two APIs need to be enabled once per project (Section 4a); after that, each user just runs the ADC login on their own machine (Section 4b).

Two things to know up front:

- Vertex AI was rebranded to **Gemini Enterprise Agent Platform** at Google Cloud Next 2026. The API identifiers are unchanged (`aiplatform.googleapis.com`, `generativelanguage.googleapis.com`); the console now displays them as **Agent Platform API** and **Gemini API**.
- If you are wondering why not a plain Gemini API key: Google now requires Gemini API keys to be bound to a service account, and CloudBank principals cannot grant that service account the role it needs, so the API-key path is effectively closed on these accounts. ADC as yourself sidesteps all of it.

### 4a. Enable the APIs (once per project)

1. Enter the GCP console from CloudBank (Section 1). In the project picker, select your CloudBank project; its ID has the form `access-<allocation>-<id>` (for example `access-abc123456-789012`).
2. Go to **APIs & Services → Library** and enable both:
   - **Agent Platform API** (identifier `aiplatform.googleapis.com`, the renamed Vertex AI API)
   - **Gemini API** (identifier `generativelanguage.googleapis.com`, the renamed Generative Language API)

   The Vertex route in this guide calls `aiplatform.googleapis.com`, but some Google client libraries reach Gemini through `generativelanguage.googleapis.com` instead, so enabling both spares you a surprise later.

One trap: the global search box at the top of the console returns documentation pages, which have no **Enable** button. Search inside the Library page itself, or open these direct links and click **Enable**:

- `https://console.cloud.google.com/apis/library/aiplatform.googleapis.com?project=<your-project-id>`
- `https://console.cloud.google.com/apis/library/generativelanguage.googleapis.com?project=<your-project-id>`

### 4b. Log in with ADC (once per user)

First make sure you have been added to the GCP billing account (**CloudBank portal → fund → GCP billing → People/Access**). Then, on your laptop:

```bash
gcloud auth application-default login
gcloud auth application-default set-quota-project access-<allocation>-<id>
gcloud config set project access-<allocation>-<id>
```

When the browser opens, sign in as your CloudBank-federated identity (the account with `cloudbank` in the domain), not a personal Google account. The second command sets the quota project, the project that is billed and rate-limited for your calls; pointing it at your CloudBank project keeps the spend on the fund.

**On an SSH-only machine** (for example an HPC login node), add `--no-browser` to the first command. It prints a `--remote-bootstrap=...` command; run that in a laptop terminal with a browser, sign in there, and paste the verification URL it returns back into the remote prompt.

The flow ends with `Credentials saved to file: [~/.config/gcloud/application_default_credentials.json]`. That file is your credential; Google's client libraries find it automatically through `google.auth.default()`. In your `.env`:

```bash
GEMINI_API_MODE=vertex
# Different tools read different names for the project id; set both.
GOOGLE_CLOUD_PROJECT=access-<allocation>-<id>
GOOGLE_PROJECT_ID=access-<allocation>-<id>
GOOGLE_CLOUD_LOCATION=global
# Don't set GOOGLE_API_KEY or GOOGLE_APPLICATION_CREDENTIALS; Google's client
# libraries find the ADC file under ~/.config/gcloud/ automatically.
GEMINI_2_5_PRO_MODEL=gemini-2.5-pro
GEMINI_2_5_FLASH_MODEL=gemini-2.5-flash
GEMINI_2_5_FLASH_LITE_MODEL=gemini-2.5-flash-lite
# -1 lets Gemini set its thinking budget adaptively.
GEMINI_THINKING_BUDGET=-1
```

### 4c. Smoke test

Export your `.env` as in Section 2d, so `$GOOGLE_CLOUD_PROJECT` is set. Then:

```bash
TOKEN=$(gcloud auth application-default print-access-token)
curl -s -w "\nHTTP %{http_code}\n" -X POST \
  "https://aiplatform.googleapis.com/v1/projects/$GOOGLE_CLOUD_PROJECT/locations/global/publishers/google/models/gemini-2.5-flash:generateContent" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"contents":[{"role":"user","parts":[{"text":"Reply pong"}]}],"generationConfig":{"maxOutputTokens":64,"thinkingConfig":{"thinkingBudget":0}}}'
```

How to read the result:

- **HTTP 200 with `candidates[0].content.parts[0].text`**: everything works end to end.
- **`finishReason: MAX_TOKENS` with empty `parts`**: Gemini 2.5 spent the whole output budget on internal thinking. The `thinkingBudget: 0` override in the command above avoids this during smoke tests; real runs should keep the adaptive default (`GEMINI_THINKING_BUDGET=-1`) with a larger output-token cap.
- **HTTP 403 `PERMISSION_DENIED`**: your account lacks `aiplatform.endpoints.predict`. Email `help@cloudbank.org` and ask them to grant `roles/aiplatform.user` **to your user principal** (not to a service account, which would take you off ADC).
- **HTTP 404 `Publisher Model ... not found`**: `GOOGLE_CLOUD_LOCATION` and the URL must both use `global`.

### 4d. Running on Slurm / HPC

ADC fits clusters well: `$HOME` is shared, so `~/.config/gcloud/application_default_credentials.json` is visible on every compute node and jobs find the credential automatically. On a cluster such as NCSA Delta:

1. Run the `--no-browser` login from Section 4b once on the login node.
2. In your sbatch script, optionally make the credential path explicit:

   ```bash
   export GOOGLE_APPLICATION_CREDENTIALS="$HOME/.config/gcloud/application_default_credentials.json"
   export GOOGLE_CLOUD_QUOTA_PROJECT=access-<allocation>-<id>
   ```

3. (Recommended) Check ADC at job start, so an expired credential fails fast instead of dying mid-batch:

   ```bash
   python -c "import google.auth, google.auth.transport.requests as r; c,_=google.auth.default(); c.refresh(r.Request())" \
     || { echo "ADC expired; re-run gcloud auth application-default login on the login node" >&2; exit 1; }
   ```

Compute nodes need outbound HTTPS to refresh tokens and reach `aiplatform.googleapis.com`. Delta's standard GPU partitions allow this; on clusters that firewall compute nodes, set `HTTPS_PROXY` in the sbatch script.

## 5. Assemble your `.env`

Combine the blocks from Sections 2d, 3c, and 4b into one gitignored `.env` of your own. A template with all of them is in [`.env.example`](.env.example):

```bash
cp .env.example .env     # then fill in your own values
set -a; source .env; set +a
```

This repo deliberately stops at a working `.env`. The code that calls the models is yours to write, and it is a thin layer over each provider's SDK or REST API. Before wiring the `.env` into that code, confirm each provider with its self-contained smoke test (GPT in Section 2d, Claude in Section 3c, Gemini in Section 4c). Three green responses mean your credentials are good.

## 6. Guardrails

A few habits that protect a capped research fund:

1. **Set budget alerts** at 25%, 50%, and 80% of your fund in each cloud you use. Azure: **Cost Management → Budgets**. GCP: **Billing → Budgets & alerts**. AWS, if you use Bedrock: **Billing → Budgets**.
2. **Cache model responses.** Rerunning an experiment repeats many identical calls; caching responses to disk means you pay for each one only once. Sharing the cache across machines extends the savings.
3. **Keep flagship models for final runs.** Debug and ablate with the cheap tiers (for example `gpt-4o-mini`, `claude-haiku-4-5`, `gemini-2.5-flash`), and reserve the flagships for the runs you actually report.
4. **Rotate credentials when the project wraps up.** Azure: regenerate KEY 1 on your resource. Vertex: `gcloud auth application-default revoke`. Bedrock, if you used it: email CloudBank to delete the credential.

## Appendix A. Claude on AWS Bedrock (not recommended)

Bedrock was this guide's original Claude route. We moved it here after concluding that Foundry (Section 3) is the better choice; the reasons are listed in the introduction, and the blocked flagship models were the decisive one. Everything below worked when we last tested it in May 2026, except where noted, and we keep it in full in case the platform improves.

The shape of the setup: calls to `bedrock-runtime` (via the AWS CLI or an SDK such as boto3) read AWS credentials from the standard chain (environment variables or `~/.aws`). You need Anthropic's use-case form approved once for the account (A.1), a credential in that chain (A.2, A.3), and `us.*` model ids in your `.env` (A.4, A.5).

Two account-level quirks, both confirmed by testing:

- The old **Model access** page is retired. Anthropic models are auto-enabled on first invocation, but the use-case form is still required, and it is now reached from each model's detail page in the **Model catalog**.
- **CloudBank's parent-org service control policy (SCP) denies `global.*` inference profiles.** Some background: Bedrock addresses models through cross-region inference profiles, model ids with a routing prefix (`us.`, `global.`) that spreads requests across regions. An SCP is a rule that CloudBank's parent organization applies to the whole account, so you cannot lift it; you can only avoid it by sticking to `us.*` ids. We confirmed the deny in the Bedrock console's Playground (`explicit deny in a service control policy: ...p-tnlm356a`), and it applies to API keys too, so this is not just a console-session limitation.

### A.1 Submit the Anthropic use-case form (skip if already done for your account)

1. Enter the AWS console from CloudBank (Section 1). Region: **us-east-1**.
2. In the Bedrock console, open **Model catalog** in the left nav, filter provider = **Anthropic**, and click into any Claude model (for example *Sonnet 4.6*).
3. On the model detail page, fill in the **use-case details** banner:
   - Company: `<your institution>`
   - URL: `https://<your-institution>.edu`
   - Industry: `Education`
   - Intended users: **Internal users** only
   - Description: a few sentences on your research use. Adapt this template:

     ```text
     Academic research at a university for a peer-reviewed NLP paper. We evaluate Claude alongside other LLM families on public benchmarks within a multi-agent method. Used only by the PI and a small group of academic collaborators. No PII, no public-facing product, no commercial deployment. Outputs are aggregated for analysis and reported in the paper. Funded via NSF ACCESS / CloudBank.
     ```

4. Approval is quick. Confirm it by pinging Sonnet 4.6 in **Test → Playground**; any response means the gate is open for the whole account.

### A.2 Get an AWS credential

boto3 and the AWS CLI read credentials from the standard chain, so any of the shapes below works once it is in place. They differ mainly in lifetime:

| Credential | How to get it | Lifetime | Good for |
|---|---|---|---|
| Your CloudBank federated session | sign in through the CloudBank AWS portal (self-serve) | short, tied to the login session | console work such as the Playground; spend is attributed to you personally |
| Short-term Bedrock API key (`AWS_BEARER_TOKEN_BEDROCK`) | Bedrock console → API keys (self-serve) | at most 12 hours | quick, console-free testing |
| Long-term Bedrock API key, or static IAM access keys | CloudBank issues them on request | long-lived | unattended jobs, such as 24-48 h Slurm runs |

For interactive testing, the two self-serve rows are immediate: the federated console session covers Playground work, and a short-term API key covers CLI or SDK calls. For unattended jobs, ask CloudBank (`help@cloudbank.org`) to issue a long-lived credential for your project: minting a long-term Bedrock API key calls `iam:CreateUser`, which the CloudBank PowerUser role lacks, and static IAM keys are disabled by default, so neither is self-serve. One more wrinkle: the account signs in through a SAML federation proxy (`federation-proxy.cloudbank.org`) rather than an IAM Identity Center portal, so `aws configure sso` may not apply. Until CloudBank issues you a durable credential, keep batch runs short enough to finish within one credential's lifetime.

### A.3 Verify the credential

With the credential in place, verify it from the AWS CLI:

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

How to read the result:

- **A JSON response with a Claude reply**: done, the credential can invoke Claude.
- **`AccessDeniedException` mentioning use case or `PutUseCaseForModelAccess`**: the use-case form is not actually approved; redo A.1.
- **`AccessDeniedException` on `bedrock:InvokeModel` with no SCP mention**: the access policy is missing; ask CloudBank to confirm `AmazonBedrockLimitedAccess` is attached to your user.
- **`AccessDeniedException` citing `explicit deny in a service control policy`**: you invoked a `global.*` profile, which CloudBank's parent-org SCP blocks. Switch the call, and your `.env`, to the corresponding `us.*` profile.
- **`ValidationException` on the model id**: the profile string is misspelled or does not exist in this region. Check the exact spelling under **Bedrock console → Cross-region inference**. If Haiku 4.5 is not available at all, fall back to Haiku 4.0 (`anthropic.claude-haiku-4-0`).

### A.4 The flagship is blocked; substitute an older Opus

In our Playground test, `anthropic.claude-opus-4-7` (the flagship at the time) returned "not available for this account / contact AWS Sales", an account-level gate AWS applies to its newest flagship. The gate has not loosened since: as of August 7, 2026, Claude Opus 5 is blocked the same way. Sonnet 4.6 and Opus 4.6 both invoke cleanly, so on Bedrock the practical ceiling is **Opus 4.6**. Point any flagship references in your config or code at the 4.6 id, and file an AWS Sales request in parallel if you want to try. Or skip the whole problem: Foundry (Section 3) serves the current flagship with no access request.

### A.5 `.env` block

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

Every Claude id is pinned to a `us.*` inference profile because the SCP denies `global.*`. If Bedrock rejects one of these ids, confirm the exact profile strings for your account under **Bedrock console → Cross-region inference**, as in A.3. boto3 (>= 1.39) or the AWS CLI auto-detects whichever credential is present, in environment variables or `~/.aws`.

## Sources

- CloudBank: [Add users to AWS](https://community.cloudbank.org/t/how-do-i-add-users-to-my-aws-account/29) / [Azure](https://community.cloudbank.org/t/how-do-i-add-users-to-my-azure-account/30) / [GCP](https://community.cloudbank.org/t/how-do-i-add-users-to-my-gcp-account/31); [CloudBank vs IAM user](https://community.cloudbank.org/t/difference-between-a-cloudbank-user-and-a-iam-user/111); [Help configuring AWS access key](https://community.cloudbank.org/t/help-configuring-aws-account-access-key/112/4)
- Azure OpenAI and Foundry: [Create a resource](https://learn.microsoft.com/en-us/azure/foundry-classic/openai/how-to/create-resource), [v1 API lifecycle](https://learn.microsoft.com/en-us/azure/foundry/openai/api-version-lifecycle), [Azure OpenAI → Foundry upgrade](https://learn.microsoft.com/en-us/azure/foundry/how-to/upgrade-azure-openai), [Claude in Foundry](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/how-to/use-foundry-models-claude)
- Vertex AI: [Get an API key](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/start/api-keys?usertype=expressmode), [Authenticate to Vertex AI](https://docs.cloud.google.com/vertex-ai/docs/authentication)
- Bedrock (Appendix A): [Generate API key](https://docs.aws.amazon.com/bedrock/latest/userguide/api-keys-generate.html), [Use a Bedrock API key](https://docs.aws.amazon.com/bedrock/latest/userguide/api-keys-use.html), [Identity and access management](https://docs.aws.amazon.com/bedrock/latest/userguide/security-iam.html), [Access Anthropic models](https://repost.aws/knowledge-center/bedrock-access-anthropic-model)
