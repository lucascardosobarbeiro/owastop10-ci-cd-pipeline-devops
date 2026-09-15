# DevSecOps pipeline lab

This fork adds a CI/CD security pipeline to OWASP Juice Shop. Juice Shop is
intentionally vulnerable: scan findings are expected and are the subject of the
exercise. Never expose this application to the internet or deploy it to a
production environment.

## Pipeline design

The workflow is defined in `.github/workflows/devsecops.yml`. Every security
control is an independent GitHub Actions job so reports and failures can be
reviewed separately.

| Job | Control | Tool | Blocking threshold |
| --- | --- | --- | --- |
| `sast` | Source analysis | Semgrep | `ERROR` findings after triage |
| `sca` | Dependency analysis | `npm audit`, Trivy | Trivy high and critical |
| `secrets` | Credential detection | Gitleaks | Any verified secret |
| `container` | OCI image analysis | Trivy | High and critical, fixed findings only |
| `iac` | Dockerfile, Terraform and workflow analysis | Checkov | Triaged high/critical rule IDs |
| `dast` | Running application analysis | OWASP ZAP baseline | ZAP failures; warnings are reported |

All jobs use read-only repository permissions. Actions are fixed to commit
SHAs, scanner images are fixed to immutable digests, scan
outputs are kept as workflow artifacts for 7 days, and DAST runs against a
container on a private Docker network without publishing port 3000.

## Audit mode and enforcement mode

The pipeline was introduced in **audit mode** so the initial findings could be
reviewed without making every learning commit fail. The repository now contains
exact, reviewed baselines for intentional Juice Shop training findings; new
findings are still evaluated by the enforcement steps.

For each finding:

1. Confirm it is reproducible and identify the affected component.
2. Mark it as a true positive, false positive, or accepted lab risk.
3. Fix true positives in pipeline code and dependencies where doing so does not
   remove an intentional Juice Shop challenge.
4. Review exact intentional-secret fingerprints in `.gitleaksignore`, add
   confirmed high/critical Checkov IDs to `.checkov-enforce.txt`, and document
   accepted Trivy risks in `.trivyignore`. Include justification, owner and
   review date. Never use broad path or severity exclusions.
5. Re-run the workflow and review every uploaded report.

After triage, create this GitHub repository variable:

```text
SECURITY_ENFORCEMENT=enforce
```

In **enforcement mode**, new Semgrep error findings, new Gitleaks fingerprints,
and Trivy high/critical dependency and container findings
fail their jobs. `npm audit` remains a report-only second opinion. Checkov first
produces a complete report and then blocks on the reviewed IDs in
`.checkov-enforce.txt`; an empty file means no IaC finding has yet been confirmed
for blocking. ZAP warnings remain informational (`-I`); rules changed to `FAIL` in
`.zap/rules.tsv` block DAST. Protect `master` and require all six jobs before
merge.

Create the variable under **Settings > Secrets and variables > Actions >
Variables** only after reviewing the committed baselines.

## Local use

Build and start the target on an isolated network:

```bash
docker network create devsecops-lab
docker build -t juice-shop-lab .
docker run --rm --name juice-shop --network devsecops-lab \
  -p 127.0.0.1:3000:3000 juice-shop-lab
```

Binding to `127.0.0.1` prevents access from other hosts. Open
<http://localhost:3000>, then run a baseline scan from another terminal:

```bash
mkdir -p zap-reports
cp .zap/rules.tsv zap-reports/rules.tsv
docker run --rm --network devsecops-lab \
  -v "$PWD/zap-reports:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:2.16.1@sha256:7840969c7c9fead565bf9734b12f49f6886db90b1d35b1f74d79710bbd081dab \
  zap-baseline.py -t http://juice-shop:3000 \
    -c rules.tsv -r zap-report.html -J zap-report.json -I
```

Manual testing can be performed with Burp Suite Community. Configure the browser
proxy only for the local URL, keep interception inside the lab, and never target
systems without explicit authorization.

Clean up when finished:

```bash
docker rm -f juice-shop
docker network rm devsecops-lab
```

## Reading results

Open a workflow run in GitHub and download the artifacts from its summary page.
JSON reports can be processed by tooling; the ZAP HTML report is intended for
manual review. Artifacts may contain source paths or vulnerable package names,
so do not publish them without reviewing their contents.

The pipeline does not claim to make Juice Shop safe. Its purpose is to teach
detection, classification, remediation and policy enforcement against an
application deliberately designed to contain security flaws.
