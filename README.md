# AJ Lending: an AI credit-risk and underwriting copilot on AWS

_Project showcase: deck, demo video and write-up._

> A credit score alone doesn't tell you *why*, or *what to do about it*. AJ Lending predicts the chance an applicant repays, explains the decision in plain words, shows a realistic path to approval, and drafts a policy-grounded letter that a person reviews. It was built end to end on AWS, from raw data to a deployed web app.
>
> **It is built for two kinds of users, applicants and staff, and the app looks and works differently for each.**

AJ Lending is **one app that serves two kinds of people, and it works differently for each**.

- **Applicants (clients)** sign in to a personal view: their credit score, their existing loans, what is holding the score back, and a way to apply for new loans or ask an AI assistant questions about their own file.
- **Staff** sign in to a separate workspace: the list of applicants, a portfolio view, a detailed review of each person with their risk and the reasons behind it, bulk uploads, letters to approve, and an AI agent that can answer questions across many applicants.

The person's login decides which experience they see.

**Portfolio deck:** [View the slides (PDF)](docs/AJ_Lending_Portfolio.pdf) · [Download the PowerPoint](docs/AJ_Lending_Portfolio.pptx)

**How the numbers are calculated:** [`docs/FORMULAS.md`](docs/FORMULAS.md) (credit score, probability of default, payments, fairness and drift formulas)

**Source code:** private repository, available to share on request.

AJ Lending is a fictional lender. This is a learning and portfolio project, not a real lending product and not legal advice.

## Screenshots

**Applicant view:** credit score, next best steps, existing loans, what is holding the score back, and a late-payment slider.

![Applicant view](docs/screenshots/02-applicant-view.png)

### More of the applicant experience

| Apply for a loan | Loan explorer |
|---|---|
| ![Apply](docs/screenshots/05-apply.png) | ![Loan explorer](docs/screenshots/06-loan-explorer.png) |
| **Five short steps, with the loan type setting the rate.** | **Compare loan types and amounts side by side, each scored on the applicant's own details.** |

| Chat with AI | Menu |
|---|---|
| ![AI chat](docs/screenshots/07-ai-chat.png) | ![Menu](docs/screenshots/08-menu.png) |
| **An assistant that can only see the applicant's own applications.** | **Applicants get four simple screens.** |

| Account menu | Profile |
|---|---|
| ![Account menu](docs/screenshots/09-account-menu.png) | ![Profile](docs/screenshots/10-profile.png) |
| **Role and access shown clearly.** | **Personal details stay in the account and never reach scoring, staff screens or the AI.** |

### The staff experience

Staff and admins sign in to a different app, marked **Admin console**, with a dark header and a separate menu.

![Applicant list](docs/screenshots/11-staff-applicants.png)
**The applicant list:** 302 applicants, filterable by source, with each person's decision and a link to the detailed review.

![Detailed review](docs/screenshots/18-staff-review.png)
**Detailed review:** probability of default, the credit score, the inputs (every edit is logged) and the applicant's existing loans.

![Reasons, scenarios and what-if](docs/screenshots/19-staff-review-reasons.png)
**Why and what-if:** the main reasons from SHAP, improvement scenarios re-scored by the model, and a what-if calculator for loan amount, term and rate.

![AI analyst summary and agent](docs/screenshots/20-staff-letter-and-agent.png)
**AI analyst summary:** drafted by Bedrock from the lending policy, with automatic checks (reasons match the model, guardrail passed) and token cost, then approved by a person. Below it, the Ask AI agent.

| Menu | Upload applicants |
|---|---|
| ![Staff menu](docs/screenshots/12-staff-menu.png) | ![Upload applicants](docs/screenshots/14-staff-upload.png) |
| **Applicants, upload, add one, AI chat and staff management.** | **Upload a CSV of up to 200 applicants, or fill in a form. Each is scored right away.** |

| Add an applicant | AI chat across the portfolio |
|---|---|
| ![Add an applicant](docs/screenshots/15-staff-add-applicant.png) | ![Staff AI chat](docs/screenshots/16-staff-ai-chat.png) |
| **The same five-step form that applicants use.** | **The agent compares applicants and answers portfolio and policy questions, next to live portfolio numbers.** |

| Manage staff (admin only) | Account menu |
|---|---|
| ![Manage staff](docs/screenshots/17-staff-manage-staff.png) | ![Admin account menu](docs/screenshots/13-staff-account-menu.png) |
| **Admins invite staff and give or remove access. Everyone else is an applicant.** | **The role and what it can access are always visible.** |


---

## One app, two experiences

| Staff (underwriters, admins) | Applicants |
|---|---|
| Applicant list and portfolio view | "My loan": credit score (300 to 850), what is holding it back |
| Detailed review: score, probability of default (PD), top reasons (SHAP), existing loans | Existing loans with rate, remaining balance and remaining time |
| Improvement scenarios and what-if explorer | Late-payment slider to see the effect of paying on time |
| Bulk or single upload of applicants | Apply wizard for different loan types, each with its own rate |
| AI chat across applicants (the staff agent) | Plain-language AI chat about their own file |
| Draft adverse-action and analyst letters, human approval | Profile page: name, date of birth, phone, address, password |

Sign-in is by email through Cognito; a person's group (admin, staff or applicant) decides which screens they see. Personal details such as date of birth, phone and address are kept in the user's profile and are never sent to the scoring API, to staff views or to the AI.

## Architecture

![AJ Lending architecture on AWS](docs/architecture.png)

Five CDK stacks (Data, ML, GenAI, API, Monitoring) deploy everything in `us-east-2`.

## What it demonstrates

**Data engineering.** Home Credit data (307,511 applicants, several related tables) lands in an encrypted S3 lake. A Glue PySpark job builds one profile per applicant, with Athena for SQL exploration and SageMaker Feature Store for serving.

**Machine learning.** XGBoost trained on SageMaker with monotonic constraints, so more late payments can never *raise* a score. Deployed model (version 6): AUC 0.741, Gini 0.481, KS 0.364. The raw probability is turned into a 300 to 850 score using standard scorecard scaling. Every model version waits for a person to approve or reject it in the registry; versions 4 and 5 were rejected.

**Fairness.** An early model leaned on age-related signals. Default rates differed by age band and the four-fifths ratios failed (0.632 at worst). After removing age proxies and testing eight fixes, the ratios came out at 0.825 and 0.965. Protected attributes are used only to measure fairness, never to score.

**Generative AI.** Letters are written by Bedrock Nova and grounded in a lending-policy knowledge base (RAG on S3 Vectors). Bedrock Guardrails filter the output, and automatic checks confirm each letter cites exactly the model's real reasons and no protected terms. A person approves every letter.

**Agentic AI.** A Strands agent running on Bedrock AgentCore has six tools that call the same Lambda API as the web app, so scoring logic lives in one place.

**Security and privacy.** Least-privilege IAM roles, KMS encryption, Cognito groups, an API authorizer, PII kept out of the API, staff views and AI prompts (a test enforces it), and an AWS Budget with email alerts.

**MLOps and LLMOps.** A model registry with approval, in-place endpoint updates with a rollback configuration, a PSI drift Lambda with CloudWatch alarms, per-letter token and cost metrics, and offline letter evaluation.

## Skills demonstrated

### Agentic AI
- **Built a tool-using agent** with the Strands Agents framework, running on Amazon Bedrock AgentCore Runtime (containerized, built with CodeBuild) and powered by Amazon Nova Pro at low temperature for steady answers.
- **Six tools:** `get_applicant`, `what_if`, `lookup_policy`, `draft_letter`, `compare_applicants` and `portfolio_overview`. The agent decides which to call, in which order, to answer questions such as "compare these two applicants" or "why was this applicant declined?".
- **One backend for people and agent:** every tool calls the same Lambda API the web app uses, so scoring, permissions and fairness rules exist in one place and the agent cannot see more than the app would.
- **Guardrails around the AI:** Bedrock Guardrails on outputs, a retrieval step over a policy knowledge base (RAG on S3 Vectors), automatic checks that a letter cites exactly the model's real reasons and no protected terms, and a person approving every letter.
- **Agent operations:** invoking the deployed agent from scripts, per-request cost and latency tracking, and an offline evaluation of 25 generated letters.

### Data engineering
- Designed an encrypted S3 data lake and a **PySpark job on AWS Glue** that combines several related tables (applications, bureau history, installment payments, previous applications) into one profile per applicant, 307,511 in all.
- Registered the data in **Athena** for SQL exploration and loaded profiles into **SageMaker Feature Store** for serving.
- Kept training and serving features identical with shared derived-ratio code and a **parity test**, avoiding training/serving skew.
- Handled messy real data: missing values, bureau records without interest rates (estimated and flagged as estimates), and a reproducible train/validation/test split.

### Machine learning
- Trained **XGBoost** on SageMaker with **monotonic constraints**, so the model's behaviour matches what a credit reviewer would sign off on.
- Evaluated with **AUC, Gini and KS** (0.741, 0.481, 0.364 for the deployed version) and compared against a logistic-regression baseline.
- Turned probabilities into a **300 to 850 credit score** using standard scorecard scaling, and chose an explicit decision threshold as a policy choice.
- Explained every decision with **SHAP reason codes** and built a "path to approval" counterfactual (the smallest loan reduction that flips a decline).
- Tested **fairness** with Fairlearn and the four-fifths rule, found age bias, removed age proxies and measured the fix (0.632 to 0.825 and 0.965).
- Ran **MLOps**: model registry with human approval, in-place endpoint updates with rollback, PSI drift monitoring with alarms, and 63 automated tests.

All formulas are written out in [`docs/FORMULAS.md`](docs/FORMULAS.md).

## Tech stack

| Area | Tools |
|---|---|
| ML | Python, XGBoost, SHAP, scikit-learn, Fairlearn |
| AWS data and ML | S3, Glue (PySpark), Athena, SageMaker (training, Feature Store, Registry, serverless endpoint) |
| AWS GenAI | Bedrock (Nova Lite and Pro, Titan embeddings, Guardrails, Knowledge Bases), S3 Vectors, AgentCore, Strands Agents |
| Backend | Lambda, API Gateway, Cognito, DynamoDB, CloudWatch, Budgets, KMS |
| Infrastructure | AWS CDK (Python), CodeBuild |
| Frontend | Next.js 15, React 19, TypeScript, Tailwind CSS, Amplify Auth |
| Quality | pytest (63 tests, no AWS needed), GitHub Actions |

## Problems solved along the way

- **Age bias:** found it from the data, removed the proxies, and measured the result with the four-fifths rule.
- **Drift false alarm:** a drift alarm fired on the first run; I traced it to how the baseline was built and corrected it.
- **Serverless endpoint and dependency conflicts:** pinned the container versions so training and inference agree.
- **Private IDs in a public repo:** moved deployment-specific values into a git-ignored `infra/cdk.context.json` and verified the synthesized stack was identical.
- **29-second API limit:** kept the web path fast (a light mode for lists) and moved heavier work to the agent.

## Cost

With nothing running, the stack costs roughly $1 a month, mostly KMS keys. The serverless endpoint is free when idle, and Bedrock is pay per use. An AWS Budget emails you at 80% of a monthly cap. A single command tears everything down.

## How it was built

I built this with an AI coding assistant.

---

© 2026 Jyothsna Anne. All rights reserved, see [LICENSE](LICENSE). The Home Credit dataset belongs to its owners (Kaggle competition terms) and is not included.
