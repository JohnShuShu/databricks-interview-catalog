# Databricks interview debrief


For each topic below:
* **Asked:** what the interviewer wanted.
* **You said:** a condensed version of your answer.
* **Concepts in depth:** the underlying Databricks or AWS concepts, explained fully.
* **Corrections / updates:** where the answer was imprecise or out of date.
* **Stronger answer:** a tighter version to reuse in follow-up rounds.

Facts about GovCloud availability were checked against Databricks and AWS docs on 2026-10-05 (sources at the bottom).
GovCloud changes month to month, so re-check them before quoting.

---

## 0. The role in one paragraph

[00:00–01:00] A Deloitte senior manager, who is also **the architect** on the team, is standing up **Databricks on AWS
GovCloud** for a federal agency that handles a lot of financial data. They want one hands-on person to **own staging and
setting up the platform** and to guide a team of smart engineers who are new to Databricks. They expect the
**Terraform from the Databricks vendor** (account team / professional services) as the starting point. Separate teams
handle the **ATO**, security (an "advisory" team) and AWS infrastructure, so the Databricks/AI-analytics team has to
coordinate with all of them. The work is planned in sprints and nothing moves until the client approves the plan.

**Engagement facts mentioned** [28:56–34:05]:
| Item | Detail |
|---|---|
| Team today | An analytics team (Jupyter notebooks on **EMR**, O&M of the old stack). Over the last ~6 months: a chat app and **OpenSearch** semantic search |
| Why Databricks | Data is spread around (mainly **Postgres**) and needs to be centralised for AI. The contract finally came through |
| Mandate | "Next 10 months: implement Databricks, get it ATO'd, start building applications on it" |
| Size | 150+ people on the whole engagement; ~30 engineers on AI/analytics; the **new Databricks team is 4–5 people**, plus an apps team that will consume it. You'd be one of the first |
| Contract | 10-month extension that bridges to a **new BPA** (Blanket Purchase Agreement). Deloitte has been there 5 years and plans to stay |
| AI tooling | The client environment has AI coding assistants ("Claude Code or some AI assistance"); they're teaching people to build agents with **BMAD** (the BMad Method agent framework). They **expect** you to use these tools heavily |
| Clearance | Asked to confirm an **active Secret** clearance: you said yes |
| Hardware | No GFE (government-furnished equipment). Deloitte provides laptops; Macs work as of last month. You connect with a PIV/CAC card ("group card") → agency VPN → **Amazon WorkSpaces** |
| Close | "Giving me a lot of good confidence, but still interviewing candidates" |

### Glossary of the federal terms that came up
* **GovCloud (US)**: AWS's isolated regions (`us-gov-west-1`, `us-gov-east-1`) for regulated US government workloads.
  They have separate accounts, separate IAM, a separate partition (`arn:aws-us-gov:`) and US-person operators. Databricks
  runs a separate GovCloud deployment with its own account console. Features arrive months behind commercial AWS.
* **ATO (Authority to Operate)**: the formal sign-off from the agency's Authorizing Official that a system's risk is
  acceptable. It is built on the NIST RMF (SP 800-37) and SP 800-53 controls. The evidence package is the **SSP**
  (System Security Plan), SAR, POA&M and continuous monitoring. The Databricks platform you build **inherits** controls
  from Databricks' FedRAMP authorization and from AWS GovCloud's. What you configure (IAM, network, logging, encryption,
  access reviews) is the "customer responsibility" part of the SSP.
* **FedRAMP High / DoD IL5**: Databricks on AWS GovCloud supports both. The workload's FIPS 199 impact level decides
  which one applies. Financial data across agencies is likely Moderate or High.
* **BPA**: the procurement vehicle the task orders run under. It matters for how long the work lasts, not for the
  technology.

---

## 1. "Introduce yourself / how would you set it up?" [01:14–05:45]

**You said:** You gave an end-to-end overview: Azure is PaaS and AWS is SaaS; get Databricks from the Marketplace;
Terraform the metastore, workspaces and credentials; set up "SSL termination, Route 53"; design the Unity Catalog
hierarchy around the data and how the team works; Terraform everything because there will be 100+ users; SCIM from
an identity governance tool (Saviynt, SailPoint, ServiceNow); grant to groups, put users in groups, least privilege;
promote dev → UAT → prod; an IAM role for S3 access, with storage credentials in Databricks.

The interviewer cut in at [05:46]: _"let's slow down… this agency is a pretty high secure environment… things don't go
that fast."_ **Takeaway:** lead with the *plan and approval gates*, then the tech. They are buying a safe pair of hands
in a slow, approval-driven environment more than raw speed.

### Concepts in depth

**Architecture: control plane vs compute plane.** Every Databricks deployment has two parts:
* The **control plane** is run by Databricks in its own account: web UI, REST APIs, job scheduler, notebook storage,
  the Unity Catalog service.
* The **compute plane** is where data is processed. *Classic* compute runs as EC2 instances **in your AWS account**,
  inside a VPC you provide. *Serverless* compute runs in **Databricks' account**, in a serverless compute plane that is
  isolated per workspace.

So the Azure-vs-AWS difference isn't really PaaS vs SaaS. **Azure Databricks** is a first-party Azure service: Microsoft
sells and bills it, Entra ID is built in, and you create it in the Azure portal. On **AWS**, Databricks is a
partner-operated service. You subscribe directly or via AWS Marketplace, create an *account*, then register AWS resources
with it. The cloud split is the same on both clouds.

**What "provisioning a workspace" really means on AWS** (this is exactly what the vendor's Terraform will do. Your
`~/Documents/Repos/databricks-terraform` has the same layers, see `databricks-terraform.md`):
1. **Account level** (`accounts.cloud.databricks.us` on GovCloud, `accounts.cloud.databricks.com` on commercial):
   the account admin, a Terraform **service principal** with OAuth M2M, and the account-level SCIM/identity setup.
2. **Credential configuration**: a cross-account IAM role that Databricks assumes to launch EC2 instances in your
   account. The trust policy uses Databricks' account ID with your Databricks account ID as the `sts:ExternalId`.
3. **Storage configuration**: the workspace **root bucket** (DBFS root, system data). Its bucket policy allows the
   Databricks account.
4. **Network configuration** with a customer-managed VPC: two or more private subnets in different AZs, security
   groups, and **PrivateLink VPC endpoints** for the workspace (REST API) and the secure cluster connectivity relay. On
   **GovCloud, PrivateLink is required for both front-end and back-end**. Users reach the UI through a front-end
   endpoint, and DNS resolves the workspace URL to it.
5. **Workspace** (`databricks_mws_workspaces`), which ties together 2–4 plus optional customer-managed keys (CMK)
   for managed services and storage. Agencies often require CMK in KMS.
6. **Unity Catalog metastore**: one per region per account. Assign it to the workspaces.

**Where "SSL termination / Route 53" actually fits.** You don't terminate TLS for Databricks; Databricks does. What
*does* involve DNS is PrivateLink. The agency's DNS (Route 53 private hosted zones or the agency's resolvers) has to
resolve `<workspace>.cloud.databricks.us` to the front-end VPC endpoint, from the VPN and WorkSpaces networks. That's
usually a cross-team ticket with the AWS infrastructure team, and it's worth naming as a dependency.

**GovCloud defaults to know.** The **Compliance Security Profile is on by default** for every GovCloud workspace. It
enforces FIPS 140 endpoints, Nitro instance types with encryption in transit, enhanced security monitoring and automatic
cluster updates. Single sign-on is **mandatory**. Data residency is always enforced to US government regions.

**Unity Catalog object model** (the hierarchy you described):
```
metastore (one per region)
 └─ catalog            ← top-level isolation unit; can be bound to specific workspaces
     └─ schema         ← a.k.a. database
         ├─ table (managed or external), view, materialized view, streaming table
         ├─ volume     ← governed files (non-tabular: PDFs, CSV drops, model artifacts)
         ├─ function   ← SQL/Python UDFs, also used for row filters & column masks
         └─ model      ← MLflow registered models
plus securables outside the tree: storage credential, external location, connection (federation),
service credential, share / recipient / provider (Delta Sharing), clean room
```

**Identity: SCIM vs identity governance.** **SCIM** is the protocol that pushes users and groups into the
**Databricks account**, not into each workspace. The source is normally the **IdP**: Okta, Entra ID, or Ping, which
federal shops often use alongside PIV/CAC. Saviynt, SailPoint and ServiceNow are **IGA** tools (access requests, approvals,
recertification). They usually drive *group membership in the IdP*, and the IdP's SCIM connector then syncs to
Databricks. The resulting flow, which an ATO assessor will like because it's auditable end to end:
```
access request (ServiceNow) → approval → IGA updates IdP group → IdP SCIM → Databricks account group
→ workspace assignment (Terraform) → UC grants on the group (Terraform)
```
Rules of thumb: grant to **groups only, never to individual users**; jobs run as **service principals**; Terraform
owns groups that aren't from the IdP plus every grant; the IdP owns group membership.

**AWS ↔ Databricks data access** (your "set up a role, grant it S3, create credentials" point). In UC terms:
* **Storage credential**: wraps an IAM role that UC assumes. The trust policy names the UC master role plus an
  external ID, and the role must be **self-assuming**. This is the external-ID chicken-and-egg problem your Terraform
  repo handles with `skip_validation`.
* **External location** = storage credential + S3 path. Grant `READ FILES`, `WRITE FILES` and
  `CREATE EXTERNAL TABLE` on it.
* **Managed storage location** on a catalog or schema: where UC puts managed tables. Use a bucket per agency or
  domain for physical isolation (see §5).
* **Service credential** (separate from storage): lets code call other AWS services, such as **Bedrock**, Secrets
  Manager or SQS, without keys. This is how you'd wire up Bedrock in §6.

**Promotion dev → UAT → prod.** Use separate workspaces per environment, ideally separate AWS accounts, with catalogs
bound per environment (`dev_*`, `uat_*`, `prod_*`). The same Terraform runs with per-environment variables. Code
(jobs, pipelines) is promoted with **Databricks Asset Bundles** (`databricks bundle deploy -t prod`), run from CI
under a service principal.

### Stronger answer (opening)
> "I'd split it into a plan the client can approve and an implementation. Plan: workspace topology (dev/test/prod
> workspaces in separate GovCloud accounts), network design (customer-managed VPC with front- and back-end PrivateLink,
> which GovCloud requires), identity (IdP SCIM, groups only, service principals for jobs), Unity Catalog layout per
> agency, and the control mapping the ATO team needs: what we inherit from Databricks' FedRAMP High and AWS GovCloud,
> and what we own. Then we implement it as Terraform in that order: account → network → workspaces → metastore and
> storage credentials → catalogs and grants → cluster policies. Each layer is reviewable by the security and AWS teams
> before we apply it."

---

## 2. Terraform experience [06:50–08:25]

**You said:** Hands-on with Terraform Enterprise and community (CLI) editions; enterprise changes go through a pipeline;
secrets in AWS Secrets Manager; big orgs already have modules; AWS and Databricks providers; templates in JSON or
Jinja feed the modules.

### Concepts in depth
* **Two providers, two auth contexts.** The `databricks` provider is used twice: an **account-level** alias
  (`host = https://accounts.cloud.databricks.us`, plus `account_id`) for workspaces, metastores, users, groups and
  SCIM; and **workspace-level** aliases for cluster policies, grants, jobs and secrets. Authenticate with an **OAuth
  M2M service principal**, never personal access tokens (PATs).
* **Layered state** (stacks): bootstrap → account → network/workspace → UC → workspace config. This limits the blast
  radius, and the security team can review smaller plans. Your project uses exactly this layout: `stacks/00…40`.
* **Config-driven modules.** Rather than Jinja templates, the idiomatic Terraform approach is **YAML/JSON files
  decoded with `yamldecode()`** and expanded with `for_each`. For example, `groups.yaml` gives one
  `databricks_group` per entry, and `catalogs.yaml` gives catalogs, schemas and grants. Onboarding a user or a dataset
  becomes a one-line merge request (MR) that's easy to audit. Jinja is still fine for generating cluster-policy JSON.
* **CI on GitLab or Jenkins:** `plan` on MR → human review → `apply` on merge, with protected prod applies. Drift
  detection runs on a schedule. In GovCloud CI, runners need network reach to the PrivateLink endpoints and use OIDC to
  assume roles (no stored keys).
* **Vendor Terraform:** Databricks publishes `terraform-databricks-sra` (the Security Reference Architecture). It
  includes an AWS GovCloud variant with PrivateLink, CMK, the compliance profile and audit log delivery. That's very
  likely what "Terraform scripts from the Databricks vendor" means, so read it before the next round.

### Stronger answer
> "Yes, hands-on. Most recently I built a full Databricks-on-AWS Terraform layout: separate stacks for bootstrap,
> account, workspace, Unity Catalog and workspace config. Identity and catalogs come from YAML, CI runs plans on merge
> requests with manual promoted applies, OIDC replaces stored keys, and drift detection runs nightly. I'd start from the
> Databricks Security Reference Architecture's GovCloud module and adapt it to the agency's network and naming."

(That's your side project in `databricks-terraform.md`. It's concrete, recent and very relevant, so use it.)

---

## 3. Certifications [08:27–10:03]

**You said:** You've done ~4–5 Databricks Academy courses (on LinkedIn) but haven't taken the main exam because
projects keep getting in the way, and two weeks of review would be enough.
**Interviewer:** "If you have it, nobody cares. If you don't, people question you. Get it and put it aside."

**Action:** book an exam. In order of relevance to this role:
1. **Databricks Certified Data Engineer Associate**: fastest to get, and the baseline people expect.
2. **Platform Administrator accreditation** (free, in Databricks Academy): directly matches "own platform setup".
3. **Data Engineer Professional**: later, as the differentiator.

Mentioning a booked exam date in a follow-up email turns this objection into a non-issue.

---

## 4. Your role and on-call [10:03–11:25]

**You said:** Senior and sometimes staff engineer. You bring pros and cons (cost, security, maintainability) to the
lead, then implement hands-on, including 2 a.m. failures and PagerDuty.

**Fit note:** they want an **owner/guide** for 4–5 engineers who are new to Databricks. In later rounds, add evidence of
*enablement*: runbooks you wrote, pairing sessions and onboarding guides. That's the "guide us" part of the job.

---

## 5. Serverless: pros and cons [11:25–13:41]

**Asked:** the Databricks account team pushed serverless hard on last week's call. What are the trade-offs?

**You said:** Up in ~30 s, good for developers, AI and BI (SQL warehouses). For scheduled pipelines, use job clusters on
spot instances because they're cheaper; if a spot node is reclaimed, the next node picks up the work; batch runs at
night when nobody is waiting.

### Concepts in depth

**Kinds of compute:**
| Compute | Where it runs | Billing | Best for |
|---|---|---|---|
| All-purpose (classic) | Your VPC | Higher DBU rate + your EC2 | Interactive dev, shared notebooks |
| Job compute (classic) | Your VPC; created per run, then terminated | Lowest DBU rate + your EC2 (spot possible) | Scheduled production pipelines |
| SQL warehouse (classic/pro) | Your VPC | DBU + EC2 | BI, if serverless isn't allowed |
| **Serverless** notebooks/jobs/pipelines | Databricks' serverless plane | One higher DBU rate that *includes* the VMs | Spiky, interactive, short jobs |
| **Serverless SQL warehouse** | Databricks' serverless plane | DBU incl. VMs | BI and dashboards: fast start, auto-stop in minutes |

**Pros of serverless:** starts in seconds (not ~5 min for classic clusters); no cluster sizing, AMIs or spot
management; automatic upgrades; idle cost close to zero (a warehouse auto-stops after a few minutes); Databricks
handles autoscaling. Serverless jobs have a **"standard" mode** that's cheaper but slower to start, as well as
"performance-optimized" mode.

**Cons, especially in a federal setting:**
* **Data plane location.** The compute runs in Databricks' account, not the agency's VPC. The ATO and security teams
  will ask about this first. Reaching private resources (Postgres, S3 with VPC-only policies) needs **Network
  Connectivity Configurations (NCC)**, plus private endpoint rules or an NLB-backed PrivateLink. Docs on 2026-10-05
  say private connectivity from serverless to AWS resources is **not yet available on GovCloud**, except through
  PrivateLink to an NLB in your VPC. Egress controls (serverless network policies) are another thing to confirm.
* **Less control.** No init scripts, no custom instance types or GPUs on generic serverless, limited Spark configs,
  only Python and SQL, and only the libraries that environment versions allow.
* **Cost visibility.** It costs more per DBU, and long, steady jobs are usually cheaper on classic job clusters with
  spot. Track it with `system.billing.usage` and **budget policies** (serverless tagging).
* **GovCloud maturity (check this).** The feature/region table lists serverless for notebooks, jobs and pipelines in
  `us-gov-west-1` as **Public Preview**, and **serverless SQL warehouses as Preview/Beta**. A preview feature is a hard
  sell inside an ATO boundary. This is the direct answer to "the vendor sells it, but is it on GovCloud?"

**The spot-instance claim, made precise.** Spark recovers from a lost *executor*: the driver re-runs the tasks that
node had (and recomputes lost shuffle data). Losing the **driver** kills the run. So:
* set `aws_attributes.first_on_demand = 1` so the driver stays on-demand;
* use `availability = "SPOT_WITH_FALLBACK"` on job clusters;
* add job-level retries (`max_retries`) to handle the rare whole-run failure;
* make tasks idempotent (Delta `MERGE`, overwrite partitions) so a retry is safe.

**Cluster policies** turn this from advice into enforcement: they cap node types and worker counts, force
auto-termination, force tags, and require spot with on-demand for the driver. Give users "create cluster" only through
a policy.

### Stronger answer
> "Serverless for interactive and BI work: developers, SQL warehouses for dashboards, ad-hoc analysis. Classic job
> clusters for heavy, steady pipelines, with spot workers, an on-demand driver and cluster policies to enforce it. But in
> GovCloud the first questions are compliance ones: serverless compute runs in Databricks' account, not the agency VPC,
> private connectivity from serverless is limited on GovCloud today, and the docs still mark it as preview there. I'd
> put that in front of the security team early and plan on classic compute inside the boundary for the first ATO, with
> serverless SQL as a fast follow once it's covered."

---

## 6. The 22% compute saving at USCIS [13:43–15:58]

**You said:** Pipelines were inferring schemas on wide (~90-column) datasets, 2–3 runs a day across ~70 pipelines.
Defining explicit schemas removed the inference pass, so jobs ran shorter and clusters shut down sooner.

### Concepts in depth
* **Why inference is expensive.** With `spark.read.option("inferSchema", "true")` on CSV, Spark reads the data
  **an extra time** just to guess types. For JSON, it samples or scans to merge the schema. On wide data, at several
  runs a day, that's a full extra I/O pass each time, and type guesses can also *change between runs* (a column that was
  all ints gets one string value), which breaks downstream jobs.
* **The fix in modern Databricks terms:** use **Auto Loader** (`cloudFiles`) with:
  * `cloudFiles.schemaLocation`, where the schema is inferred once and stored;
  * `schemaHints` or a full schema, to pin the types;
  * `schemaEvolutionMode` (`addNewColumns` / `rescue` / `failOnNewColumns`);
  * the **`_rescued_data`** column, which catches malformed or unexpected values instead of dropping them.
  Auto Loader also processes each file only once (incremental, checkpointed), which is a second big saving over
  re-listing whole directories.
* **Other cost levers to mention alongside it:** job clusters instead of all-purpose for production (lower DBU rate);
  auto-termination; spot with an on-demand driver; Photon for SQL-heavy work (more DBU per hour but often fewer hours);
  liquid clustering or `OPTIMIZE` to fix small files; **predictive optimization** (available on GovCloud); and measuring
  all of it through `system.billing.usage` joined to job tags.

### Stronger answer
> "I profiled the 70-odd pipelines and found every run re-inferring schemas on ~90-column files, which meant an extra
> full read each time and unstable types. I moved them to declared schemas (Auto Loader with schema hints and a rescued
> data column for drift), so runs got shorter and clusters shut down sooner. That's where the 22% came from. I measured
> it from the billing data by job tag."
(Only claim the Auto Loader detail if that's what you actually used. Otherwise say "explicit StructType schemas".)

---

## 7. Isolation per agency and a central view [16:00–18:22]: _the most important design question_

**Asked:** several external agencies send sensitive (financial) data. Each needs **isolation**, but a **central
office** needs to see across all of them. How do you set up the catalogs?

**You said:** Central office: a group with read access (`USE CATALOG`, `USE SCHEMA`, read). Sharing: **Delta Sharing**.
Create a share, add tables, add recipients, grant access, no copies, and revoke whenever you want.

### Concepts in depth: and a correction
Delta Sharing is a real feature, but it's mainly for sharing **outside your metastore**: another Databricks account or
region (*Databricks-to-Databricks*), or a non-Databricks consumer (*open sharing*, token-based). Here every agency's data
lands in **your** platform under one metastore, so the main tools are **Unity Catalog isolation and fine-grained
access**, not shares. Delta Sharing becomes the right answer only if an agency runs its *own* Databricks and wants to
consume or provide data.

The GovCloud docs say **D2D sharing works "within the same environment type"**: GovCloud to GovCloud, not to commercial.

**A layout that gives both isolation and a central view:**
```
metastore (us-gov-west-1)
├─ agency_a        ← catalog per agency; managed location = s3://…-agency-a (own bucket, own KMS key)
│   ├─ raw / bronze  (only agency_a_engineers + ingestion SP)
│   ├─ silver
│   └─ gold          ← curated, documented, tagged
├─ agency_b        ← same pattern, own bucket/key
├─ …
└─ central         ← no copies: views over the gold layers + cross-agency aggregates
    └─ reporting   ← views with row filters / column masks; central_office_analysts get SELECT here
```
Isolation controls, from strongest to finest:
1. **Physical**: a separate S3 bucket and KMS key per agency catalog (managed storage location). Even a misconfigured
   grant can't reach data whose key policy excludes it, and the per-agency key is a clean ATO story.
2. **Workspace–catalog binding**: bind `agency_a` only to the workspaces that should see it. Set the catalog to
   *ISOLATED* so it's invisible elsewhere (your Terraform already does this per environment).
3. **Privileges**: reading a table needs `USE CATALOG` + `USE SCHEMA` + **`SELECT`**. There's no "read" privilege;
   `BROWSE` lets people discover metadata without reading data. Grant to groups. Remember that privileges inherit
   downward, so a `SELECT` on a catalog covers every future table in it.
4. **Fine-grained access**: **row filters** and **column masks** (SQL UDFs attached to a table), or
   **ABAC policies**: tag a column `pii=ssn` once and apply a mask policy everywhere that tag appears. For a
   financial agency, mask account numbers and SSNs for the central office unless a group is explicitly cleared.
5. **Auditing**: `system.access.audit` records who read what, and you can export it to the agency's SIEM. Lineage
   (`system.access.table_lineage`) shows what feeds the central views.

The central office then **queries views over each agency's gold layer** with no data copied, which is the property you
liked about Delta Sharing, but delivered with plain UC grants inside one metastore.

### Stronger answer
> "One catalog per agency, each on its own S3 bucket and KMS key, bound only to the workspaces that need it, so
> isolation is physical as well as logical. The central office gets a `central` catalog of views over each agency's
> curated gold layer, with column masks or tag-based ABAC on sensitive fields and row filters where they only need a
> slice. Everything is audited in system tables. If an agency runs its own Databricks in GovCloud, we'd use
> Databricks-to-Databricks Delta Sharing for that boundary: no copies, and access can be revoked instantly."

---

## 8. AI roadmap: what now vs later on GovCloud [18:22–22:12]

**Asked:** The future is AI (Genie, dashboards, apps), but "the moment you check GovCloud it's 'coming soon'." What can
we do now and what later, without over-promising?

**You said:** Two paths. (1) Native Databricks AI: Genie is easy to turn on but "a piece of crap"; connect Genie to
**Bedrock** models in GovCloud ("Claude 4.5, maybe 5") using the AWS credentials already set up. (2) Train models in
Databricks, register them in Unity Catalog and build apps on them, working with data scientists on requirements and
compute.

### Corrections
* **Don't call a vendor product "a piece of crap" in an interview,** especially to a Deloitte architect who is
  partnered with Databricks. The defensible version: *"Genie's answer quality depends on how well the space is
  curated, and out of the box it's weak."*
* **You can't point Genie at a Bedrock model.** Genie runs on Databricks-managed models. What you *can* do with
  Bedrock is:
  * call it from Databricks code using a **UC service credential** (an IAM role with `bedrock:InvokeModel`, no keys),
    from notebooks, jobs, Databricks Apps or agent code;
  * or register it as a **Model Serving external model** endpoint so it's governed and swappable. External models
    weren't listed for GovCloud in the region table on 2026-10-05, and **AI Functions** (`ai_query`) and **AI
    Playground** are listed as *not available* on GovCloud, so the service-credential route is the safe plan today.
* **Bedrock models in GovCloud (verified 2026-10-05):** **Claude Sonnet 5** (GA in GovCloud July 2026), **Claude Opus
  4.8** (June 2026), Sonnet 4.5 and older. So "Claude 5" is real now; say "Sonnet 5 and Opus 4.8".

### Concepts in depth: the GovCloud AI menu (as of 2026-10-05)
| Capability | GovCloud status | Use it for |
|---|---|---|
| **Genie** (AI/BI natural-language → SQL spaces), **Genie Code** (ex-Databricks Assistant) | GA (Jan / Mar 2026) | Business Q&A on curated tables; code assistance |
| **Databricks Apps** | GA (Jun 2026) | Hosted Streamlit/Dash/Flask/React apps inside the boundary, using UC auth |
| **Custom Model Serving** (CPU/GPU), **Foundation Model APIs** provisioned throughput | Available | Serving your own MLflow models; hosted open models |
| **AI Search** (vector search) | GA (Jun 2026) | RAG over agency documents. Could replace or complement their OpenSearch semantic search |
| **Unity Gateway** (AI gateway) | Public Preview (Sep 2026), no budgets/inference tables yet | Governing LLM endpoints |
| **AI Functions** (`ai_query`), **AI Playground**, AI Gateway inference tables, Marketplace, Lakebase | **Not available** | — |
| **Bedrock** (Claude Sonnet 5, Opus 4.8) | GA in GovCloud | Called from Databricks via a service credential |

**What makes Genie good (the useful version of "it's bad"):** a Genie space is only as good as its curation:
* a small set of **well-modelled gold tables**, not raw ones;
* **table and column comments** in UC;
* **instructions** (business definitions, such as "fiscal year starts October 1");
* **example SQL queries**, and **trusted assets** (parameterized queries or functions Genie should use for key
  metrics);
* a **benchmark** set of questions to measure accuracy before you put it in front of the client.

This ties back to the data model: good AI depends on good gold tables.

**MLflow in UC:** models are registered as `catalog.schema.model` with versions and aliases (`@champion`). They get the
same grants and lineage as tables, and Model Serving deploys from there.

### Stronger answer (now / next / later)
> "**Now** (in the ATO boundary, GA on GovCloud): curated gold tables, Genie spaces for the central office with
> instructions and benchmark questions, AI/BI dashboards, Databricks Apps for internal tools, and Claude Sonnet 5 on
> Bedrock called through a UC service credential. **Next:** RAG with Databricks AI Search over agency documents, custom
> models in MLflow/UC served through Model Serving. **Later**, when it's GA on GovCloud: serverless everywhere, AI
> Functions in SQL, gateway budgets and inference tables. I'd keep a feature-availability tracker against the GovCloud
> release notes so we only promise what's GA."

---

## 9. Time to a demo [22:12–24:30]

**Asked:** with no blockers, how long from day zero to something you can show the client?

**You said:** About a week with AI help for workspaces, UC and some permissions; the org structure takes longer; two to
three weeks without AI help.
**Interviewer:** they have AI coding tools, rely on them heavily and *expect* new people to use them well. They're
teaching everyone to build agents with **BMAD**.

**Notes:**
* "A week" is credible for a **sandbox** if the network (PrivateLink, DNS, VPN reach) is already in place. In GovCloud,
  that network work and SSO usually take longest, not the Terraform. A realistic way to frame it: *"Platform in days
  once the network and SSO prerequisites exist; first demo (one agency's data landed, a gold table, a Genie space and a
  dashboard) in 2–3 weeks."*
* **This interviewer values AI fluency.** Lean in next time with specifics: using Claude Code to generate and test
  Terraform modules (your offline `terraform test` setup with mocked providers), to review plans and to write runbooks.
  The **BMAD Method** is an open-source agent framework that gives AI agents roles (analyst, PM, architect, dev, QA)
  and runs spec-driven workflows. Look it up before the next round.

---

## 10. The pipeline: fork the existing feed into Databricks [24:30–28:38]

**Asked:** today a data engine moves sensitive data into **Postgres**. They want a **parallel branch** that also sends
the data to Databricks without disturbing the main pipeline. What tools? (They have Jenkins, Git and GitLab.)

**You said:** You took "fork" as forking the *repository*, then: create the UC resources, wire the names into the
pipeline, sort out secrets, write the Python transformations. Two options: (A) Databricks as a client, reading Postgres
in place as a foreign catalog, for a demo in two days; (B) land Parquet in S3 with a Glue catalog and point Databricks at
it, which keeps the platform open (Palantir, Starburst later), or move fully into Databricks tables.

**Note:** they meant forking the **data flow** (a tee or branch), not the git repo. It's worth saying so explicitly
next time: *"So we tap the feed without touching the existing path."*

### Concepts in depth
**Option A: Lakehouse Federation (your "Databricks as a client").** Create a UC **connection** to Postgres
(credentials stored in UC), then a **foreign catalog** that mirrors its schemas. Queries are pushed down to Postgres;
nothing is copied; UC grants and auditing apply. It's great for a fast demo. The drawbacks: load lands on the
**production** Postgres, performance is limited, and there's no history. Postgres also has to be reachable from
Databricks compute: classic compute in the VPC is fine, while serverless has the GovCloud limits in §5. GovCloud
availability of Lakehouse Federation wasn't in the region tables, so confirm it.

**Option B: ingest into the lakehouse.** Ways to branch the feed:
| Approach | How | Notes |
|---|---|---|
| **CDC from Postgres** | Logical replication → **AWS DMS** (or Debezium on Kafka/MSK) → S3 → **Auto Loader** → Delta | The standard "tap without touching". Captures inserts, updates and deletes; low load on the source |
| **Lakeflow Connect** PostgreSQL connector | Managed CDC ingestion built into Databricks | The simplest option *if* it's available on GovCloud (not in the region tables; verify) |
| **Tee in the existing data engine** | Have the current engine also write files to an S3 landing zone | Zero load on Postgres. Needs a change to their pipeline |
| **JDBC batch pulls** | A scheduled job reads Postgres tables via JDBC | Simple, but needs watermark columns and loads the source |

Then use the **medallion** pattern: *bronze* (raw, append-only, with `_rescued_data`) → *silver* (cleaned, typed,
de-duplicated, `MERGE` for CDC with `APPLY CHANGES` in Lakeflow Declarative Pipelines, formerly DLT) → *gold*
(business aggregates that Genie and dashboards use). Data-quality **expectations** in the pipeline give you documented
checks, which is good ATO evidence.

**On openness (your Palantir/Starburst point), updated.** Parquet plus Glue was the 2022 answer. Today you get the same
openness without leaving UC:
* **Delta tables with UniForm** (Iceberg metadata) in your own S3, governed by Unity Catalog;
* external engines (Starburst/Trino, Spark, Snowflake, and possibly Palantir) read through UC's **Iceberg REST
  catalog** API with UC credentials;
* one copy, one set of grants, open formats.

Note that **Managed Iceberg** is listed as not available on GovCloud, so it's UniForm on Delta there. Glue can still be
federated into UC (catalog federation) if the agency already uses it.

**CI/CD for pipelines (the Jenkins/GitLab question):** the tool matters less than the packaging. Use **Databricks
Asset Bundles**: a `databricks.yml` that defines jobs, pipelines and targets (dev/uat/prod). The CI job (GitLab or
Jenkins) runs `databricks bundle validate` and then `deploy -t <env>`, authenticated as a service principal with OAuth.
Secrets come from AWS Secrets Manager or Databricks secret scopes, never from the repo. Infrastructure (Terraform)
and code (bundles) live in separate repos or pipelines. You already work with GitLab CI, which is also on their list.

### Stronger answer
> "So we tap the existing feed without touching it. For a demo in days: a Lakehouse Federation foreign catalog over
> Postgres, read-only, with UC grants and auditing. For production: CDC from Postgres logical replication through DMS
> to an S3 landing zone, Auto Loader into bronze, and a declarative pipeline applying the changes into silver, so there's
> almost no load on their Postgres and the main path is untouched. The tables are Delta with UniForm in agency-owned S3,
> so Starburst or another engine can read the same copy through Unity Catalog's Iceberg API later. Jobs and pipelines
> deploy as Asset Bundles from GitLab or Jenkins under a service principal."

---

## 11. Your questions to them [28:41–34:05]
You asked about team size, project duration and start date, and Mac vs GFE. Good questions. For the next round, add:
* What does the vendor Terraform cover? Is it the SRA GovCloud module? Who owns the AWS accounts, VPCs and
  PrivateLink DNS?
* FedRAMP High or Moderate impact level? Will the Databricks system boundary be its own ATO or a change to an existing
  one?
* Which IdP does SCIM come from? Is identity governance done in ServiceNow, SailPoint or Saviynt?
* What is the agency's position on serverless (data processed outside the agency account)?
* What's the target date for the first production workload, and which agency's data comes first?

---

## Follow-ups / to-do
1. **Thank-you email** to the interviewer that recaps the plan-first approach and per-agency isolation design (§7),
   and gives a booked certification date.
2. **Book the Data Engineer Associate exam** and do the free Platform Administrator accreditation.
3. Read **`terraform-databricks-sra` (AWS GovCloud)** and the GovCloud feature/release-notes pages; compare them with
   your own `databricks-terraform` repo. Gaps you already know about: PrivateLink and CMK are in your `PLAN.md` §10
   backlog.
4. Look up **BMAD Method** and prepare one concrete Claude Code workflow example.
5. Prepare a one-page answer on the **ATO shared-responsibility split** for Databricks on GovCloud.

## Sources (checked 2026-10-05)
* [Databricks on AWS GovCloud: features and requirements](https://docs.databricks.com/aws/en/security/privacy/gov-cloud)
* [Databricks on AWS GovCloud release notes 2026](https://docs.databricks.com/aws/en/release-notes/gov-cloud/2026)
* [Databricks feature region support](https://docs.databricks.com/aws/en/resources/feature-region-support)
* [Genie, Foundation Model API and Assistant GA on AWS GovCloud (blog)](https://www.databricks.com/blog/aibi-genie-foundational-model-api-and-databricks-assistant-now-generally-available-aws)
* [Claude Sonnet 5 on Amazon Bedrock in AWS GovCloud (US)](https://aws.amazon.com/about-aws/whats-new/2026/07/claude-sonnet-5-govcloud/)
* [Amazon Bedrock user guide document history](https://docs.aws.amazon.com/bedrock/latest/userguide/doc-history.html)
