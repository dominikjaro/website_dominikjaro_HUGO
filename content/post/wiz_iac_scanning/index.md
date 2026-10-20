---
title: "How I integrated Wiz Security Scanning into our Infrastructure Workflow"
date: 2026-10-20
description: ""
categories: ["Infrastructure as Code"]
tags: ["Wiz","Terraform"]
image: "wiz_iac.jpg"
---

### 🧙‍♂️ How I integrated Wiz Security Scanning into our Infrastructure Workflow

**1. 💻 Local Testing: Shifting Security Left**
To help developers identify and remediate vulnerabilities before committing code, I introduced a local scanning workflow. This allows the team to catch and fix misconfigurations early in the development cycle.

- 🛠️ Created a central `Makefile` and helper script (`scripts/wiz-scan-tfplan/wiz-scan-tfplan.sh`).
- 🚀 Developers can simply run `make wiz-scan env=[PATH_TO_TERRAFORM_DIRECTORY]` to initialize, plan, and scan their Terraform modules locally.
- 🔍 The script utilizes `wizcli iac scan` to authenticate and evaluate the generated `.tfplanjson` against our security baselines.

**2. 🤖 CI/CD Integration: Atlantis Automation**
To ensure continuous security and centralized visibility, we integrated `wizcli` directly into our Atlantis pull request workflow.

- **🐳 Custom Environment:** Built and deployed a custom Atlantis Docker image pre-configured with `wizcli`.
- **🔄 Automated Pipeline:** During the PR plan phase, Atlantis generates the plan, parses it via `terraform show -json`, and executes our custom `wiz-atlantis-scan.sh` script.
- **💬 Actionable PR Feedback:** Scan results are emitted to the Wiz platform for central tracking, while a clean, formatted summary is automatically posted back to the GitHub PR as a standalone comment.
- 🧹 All temporary plan and JSON files are securely removed immediately after the scan completes.

**3. 🚧 Next Steps: Policy Enforcement & Blocking**
Our next phase focuses on establishing a firm security baseline and enforcing compliance on all new infrastructure:

- **📋 Triage & Baseline:** Review current scan results and apply global policy exclusions.
- **📜 Policy Configuration:** Create a dedicated IaC policy in the Wiz Portal targeting the "Build" lifecycle to define our exact security thresholds.
- **🛑 Block Deployments:** Update the Atlantis workflow script to exit with a non-zero status upon detecting new policy violations (e.g., CRITICAL or HIGH findings), effectively failing the plan step and blocking `terraform apply`.