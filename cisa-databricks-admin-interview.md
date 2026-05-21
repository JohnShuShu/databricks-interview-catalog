# CISA Senior Databricks Admin — Interview Question Bank

> **Engagement context:** CISA ingests **terabytes/hour** of log data from dozens of federal agencies into Databricks for cyber threat detection. The platform spans **30–40 catalogs / 80+ schemas**, multiple AWS accounts, and a mix of structured, semi-structured, and unstructured data.

> **Diagram source:** `diagrams/*.drawio` (open in [draw.io](https://app.diagrams.net) / VS Code drawio extension)
> **Rendered images:** `images/*.png` (exported via drawio CLI at 2× scale)

---

## Diagram 1 — Platform Topology

![CISA Databricks Platform Topology](images/01-platform-topology.png)

*Federal agency sources → CISA AWS account (PrivateLink + storage credentials) → Databricks workspaces (Auto Loader → Bronze/Silver/Gold via Lakeflow + Photon) → Unity Catalog → SOC consumers.*

---

## Section A — Unity Catalog Governance at Scale

![Unity Catalog Governance at Scale](images/03-uc-governance.png)

1. With **30–40 catalogs and 80+ schemas**, walk me through how you'd structure the metastore → catalog → schema hierarchy. What's your rule for catalog vs. schema boundaries when isolating agencies, classification levels, and shared reference data?
2. Compare **row filters / column masks** vs. **dynamic views** for protecting mixed-classification log fields. When do you choose each?
3. A new agency onboards Monday. Describe the end-to-end provisioning sequence — metastore assignment, catalog binding, external locations, storage credentials, group grants — and which steps must be automated vs. one-time manual.
4. How do you detect **grant sprawl** across 80+ schemas? Which `system.information_schema` tables and audit logs do you query, and on what cadence?
5. Walk me through **catalog binding** to specific workspaces. Why is it critical in a multi-agency, multi-classification environment?
6. What's your **tag-based ABAC** strategy when 80+ schemas all need PII masking? How do you avoid writing 80 mask functions?

---

## Section B — Identity, Service Principals & OAuth

![Identity and Access Plane](images/02-identity-access-plane.png)

7. Differentiate **account-level SPs**, **workspace-level SPs**, and AWS IAM roles assumed via storage credentials. For cross-account ingestion from an agency S3 bucket, which combination do you choose?
8. Walk through configuring **OAuth M2M** for a GitHub Actions runner deploying bundles into a FedRAMP workspace. What are the secret rotation, federation, and trust-boundary failure modes you've hit in production?
9. How do you enforce that **no human user** can directly query raw log catalogs — only governed service principals can? Which controls layer together (SCIM groups, UC grants, cluster policies, IP ACLs, token policies)?
10. SCIM-provisioned groups (Azure AD / Okta) vs. Databricks-native account groups — when have you mixed them, and what bit you?
11. **Break-glass accounts**: how do you provide emergency access to a FedRAMP workspace without weakening the federation model?

---

## Section C — Cross-Account AWS Connectivity

![Cross-Account AWS Connectivity](images/04-cross-account-aws.png)

12. CISA's workspace is in one AWS account; agency buckets live in 20+ others. Design the **IAM trust + storage credential + external location** pattern. How do you avoid the "one role per bucket" explosion?
13. Compare **VPC peering vs. front-end PrivateLink vs. back-end PrivateLink** for a workspace handling sensitive log data. What's the trade-off matrix?
14. A job intermittently fails with `403 AccessDenied` reading from a cross-account bucket. Walk me through your debug path — what do you check, in what order? *(Hint: see the 403 debug order in the diagram.)*
15. How does Databricks' **Lakehouse Federation** change the cross-account picture if an agency wants to keep data in Redshift / Snowflake but expose it through UC?
16. Walk through **VPC endpoint policies** for S3 — when is the gateway endpoint policy your strongest defensive layer?

---

## Section D — Automation: CLI, REST API & Asset Bundles

![Asset Bundles CI/CD Flow](images/05-asset-bundles-cicd.png)

17. You discover **200 notebooks** were granted `ALL_PRIVILEGES` to `account users` by a former admin. Sketch (pseudocode is fine) the CLI / REST API script you'd run to audit and remediate. How do you make it idempotent and dry-run-able?
18. Compare **Databricks Asset Bundles**, **raw Terraform with the Databricks provider**, and the **REST API**. Where do bundles fall down at the *admin* layer (vs. the data engineer layer)?
19. A bundle deploys cleanly in dev but fails in prod with permission errors. Walk me through your debug checklist — variables, targets, run-as identity, workspace permissions. *(Hint: see the 4-point checklist in the diagram.)*
20. Build me a **policy-as-code** layer that prevents engineers from creating clusters outside approved policies, even via the API. What enforces it server-side, and what *can't* be enforced server-side?
21. Which admin tasks belong in **bundles**, which in **Terraform**, and which in **ad-hoc CLI/REST scripts**? Give me your decision tree.

---

## Section E — Semi-Structured & Unstructured Data

22. Hourly TB-scale JSON ingestion with **evolving schemas** from dozens of agencies. Compare **Auto Loader (`schemaEvolutionMode=addNewColumns`)** vs. **Lakeflow Declarative Pipelines** vs. **hand-rolled Structured Streaming**. What does the *admin* care about that the data engineer might not?
23. An agency emits deeply nested JSON with arrays of arrays. The data engineer flattens it into 400 columns. As the admin, what concerns do you raise about **file size, small-file problem, liquid clustering, and predicate pushdown**?
24. Unstructured **PCAP or binary forensic artifacts** arrive alongside logs. Where do they live — UC **Volumes**, external locations, or outside Databricks entirely? How does that decision interact with lineage and compliance?
25. Your candidate background is mostly structured data. How do you reason about **schema drift, malformed records, and bad-record quarantine** at the platform level?

---

## Section F — Performance & Pipeline Modernization

26. You've **converted RDD pipelines to DataFrames** before. At TB/hour scale, what specific Catalyst / AQE wins did you observe? Where did DataFrames *not* help (UDFs, complex stateful ops), and when would you still reach for the Dataset/RDD API?
27. A streaming job ingesting agency logs is **falling behind** — backlog growing. Walk me through triage: cluster sizing, shuffle partitions, trigger interval, source file sizing, downstream compaction. Which Spark UI / Lakeflow metrics do you check first?
28. How do **liquid clustering** and **predictive optimization** change the admin's job vs. the old `OPTIMIZE` + `ZORDER` + `VACUUM` cron model? What still needs babysitting?
29. **Photon**: when does it pay off, when does it not, and how do you decide for a high-cardinality log workload?

---

## Section G — Databricks Apps

30. CISA analyst team wants a self-service portal to request **auto-expiring catalog grants**. Would you build this as a Databricks App, an external app calling SCIM/UC APIs, or use Genie / AI-BI? What does the admin own in each case?
31. Apps run as the **developer's identity** by default. For a multi-tenant analyst tool spanning agencies, how do you propagate the *end user's* identity to UC for row/column enforcement?

---

## Section H — Scenarios & Judgment

![9 PM Friday Incident Triage](images/06-incident-triage.png)

32. **Incident** — 9 PM Friday. Ingestion lag jumps from minutes to 6 hours. Spark UI shows healthy executors. UC queries from BI tools are timing out. Account console shows nothing red. What's your first hour? *(Walk through the triage flow above — source → Auto Loader → metastore → compute → network — and tell us what artefacts you'd capture before mitigating.)*
33. **Drift** — A junior admin manually granted `MANAGE` on a production catalog three months ago via the UI. You only just discovered it. How do you (a) find every such drift across the account, (b) reconcile against your Terraform/bundle source of truth, and (c) prevent recurrence without blocking emergency access?
34. **Cost** — Finance flags a **$400K/month spike**. Walk me through attributing spend across agencies/catalogs/jobs using `system.billing.usage`, tags, and SQL warehouse monitoring — and what governance you'd put in place.
35. **The candidate's gap** — "My background is mostly structured data and Spark optimization; less platform admin." Sell us on why that's an asset, not a liability. What does your first 30 / 60 / 90 days look like?

---

## Scoring Rubric (quick reference)

| Signal | Strong candidate | Weak candidate |
|---|---|---|
| **UC mental model** | Catalog-as-boundary, binding-aware, system-tables fluent | Treats UC like Hive metastore |
| **Automation reflex** | Reaches for bundles/Terraform first; CLI for one-offs | Defaults to UI clicks |
| **Cross-account AWS** | Knows storage credential vs. instance profile, PrivateLink trade-offs, KMS key policy as a separate failure mode | Vague on IAM trust chains |
| **Semi-structured data** | Comfortable with schema evolution, rescue data, Volumes | Wants to flatten everything to a relational schema |
| **Cost & governance** | Tags + system.billing.usage + cluster policies as one story | Sees them as separate problems |
| **Incident response** | Layered hypothesis: source → ingest → metastore → compute → network; captures artefacts before mitigating | Starts with "restart the cluster" |

---

## File Layout

```
cisa-interview/
├── cisa-databricks-admin-interview.md   ← this file
├── diagrams/                             ← editable drawio sources
│   ├── 01-platform-topology.drawio
│   ├── 02-identity-access-plane.drawio
│   ├── 03-uc-governance.drawio
│   ├── 04-cross-account-aws.drawio
│   ├── 05-asset-bundles-cicd.drawio
│   └── 06-incident-triage.drawio
└── images/                               ← rendered PNGs (2× scale)
    ├── 01-platform-topology.png
    ├── 02-identity-access-plane.png
    ├── 03-uc-governance.png
    ├── 04-cross-account-aws.png
    ├── 05-asset-bundles-cicd.png
    └── 06-incident-triage.png
```

**Re-export PNGs after editing a diagram:**
```bash
/opt/homebrew/bin/drawio -x -f png --scale 2 \
  -o images/<name>.png diagrams/<name>.drawio
```
