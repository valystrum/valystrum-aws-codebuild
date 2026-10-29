# Valystrum for AWS CodeBuild and CodePipeline

Two scripts and the CodeBuild buildspecs that run them:

- **`valystrum-scan.sh`** with [`examples/buildspec-security-scan.yml`](examples/buildspec-security-scan.yml) scans
  each pull request for **leaked secrets**, **code weaknesses (SAST)** and **IaC misconfiguration** (Terraform,
  CloudFormation, Kubernetes, Helm, Dockerfile), lists the findings as test cases in the build's **Reports** tab, and
  fails the build on a critical or high finding.
- **`valystrum-sbom.sh`** with [`examples/buildspec-sbom.yml`](examples/buildspec-sbom.yml) makes a **CycloneDX** bill
  of materials with the official CycloneDX generators and uploads it to your Valystrum software inventory. Its
  components are checked at upload against the vulnerabilities Valystrum already knows, again each night as the feeds
  bring new ones, and against your licence policy.

CodeBuild is also the build step AWS CodePipeline runs, so the same buildspecs serve a pipeline
([below](#inside-codepipeline)). Both are part of the Gold plan and above. This repository is licensed under the
Apache License 2.0 ([LICENSE](LICENSE)); the Valystrum platform it calls is not. The GitHub equivalents are
`valystrum/valystrum-pipeline` and `valystrum/valystrum-sbom`; the request, the signature and the retries are the same,
and so is what a failure does: the scan fails open, and the SBOM upload fails the build. Releases are listed in
[CHANGELOG.md](CHANGELOG.md).

> **Status, 2026-10-29.** Written for Valystrum v0.21.0, live on app.valystrum.io since 2026-10-29. Tested with AWS's
> CodeBuild local agent only: neither buildspec nor the CodePipeline configuration has run in a real AWS account yet.


## Quick start

Copy the two scripts of a release into `.valystrum/` in your repository, and the two buildspecs from `examples/`:

```bash
REL=https://raw.githubusercontent.com/valystrum/valystrum-aws-codebuild/v1.0.0
mkdir -p .valystrum
curl -fsSL -o .valystrum/valystrum-scan.sh "$REL/valystrum-scan.sh"
curl -fsSL -o .valystrum/valystrum-sbom.sh "$REL/valystrum-sbom.sh"
curl -fsSL -O "$REL/examples/buildspec-security-scan.yml"
curl -fsSL -O "$REL/examples/buildspec-sbom.yml"
```

Replace `acme` in their URLs with your organisation's slug, then follow
[Setup](#setup-from-the-key-to-the-first-green-build): issue the repository's CI key in Valystrum, store its secret in
AWS Secrets Manager, and create the two build projects.

## Setup, from the key to the first green build

**1. Issue the repository's CI key in Valystrum.** An organisation owner or admin opens **Settings, CI pipeline keys**
and issues a key for the repository as `owner/repo` (for GitLab, the full path: `group/subgroup/project`). That is
what the buildspec works out from the build's source URL; set `VALYSTRUM_REPO` (and `VALYSTRUM_SOURCE` for the SBOM)
yourself when the source is not a GitHub, GitLab or Bitbucket repository, or the build runs in CodePipeline. From a
CodeCommit or S3 source the buildspec does not come up with an empty name but a wrong one (`v1/repos/web-api` from a
CodeCommit URL; from an S3 source, the object's path inside its bucket), which Valystrum refuses with HTTP 401
`Invalid signature: sign with this repository's CI key`: set both variables on the build project to the name the key was issued for. Any name
works, such as `web-api`. The build log does not print the name it sent, so compare it with **Settings, CI pipeline keys**. The
secret (`cs_wh_...`) is shown once; the same key signs the scan and the SBOM upload. Each repository takes one of the
plan's repository slots; rotating a key does not. Every key names one repository: there is no organisation-wide key,
so a repository without a key of its own is refused with HTTP 401 `Invalid signature: sign with this repository's CI
key`, on the scan and on the SBOM upload alike. Over the API, from a signed-in admin session:

```bash
curl -X POST https://app.valystrum.io/api/v1/<org>/pipeline/config \
  -H "content-type: application/json" -H "cookie: <your signed-in session cookie>" \
  -d '{"repo": "acme/web-api"}'
```

**2. Store the secret in AWS Secrets Manager** as a secret named `valystrum/ci` with the key `webhookSecret`:

```bash
aws secretsmanager create-secret --name valystrum/ci --secret-string '{"webhookSecret":"cs_wh_..."}'
```

A secret on the command line lands in your shell history. To keep it out, here and when you rotate, write
`{"webhookSecret":"cs_wh_..."}` to a file, pass `--secret-string file://valystrum-ci.json` instead, and delete the
file.

The buildspec reads it with `env: secrets-manager: VALYSTRUM_SECRET: valystrum/ci:webhookSecret`. Give the build
project's service role `secretsmanager:GetSecretValue` on that secret (and `kms:Decrypt` if it is encrypted with a key
of your own). CodeBuild masks the value in the build log.

Each repository has a secret of its own. With more than one repository, store one secret per repository, for example
`valystrum/ci/web-api`, and name it in that repository's two buildspecs
(`VALYSTRUM_SECRET: valystrum/ci/web-api:webhookSecret`). A secret in another AWS account is named by its full ARN, and
reading it needs a customer managed KMS key and a resource policy that lets the build's role read it. A pull request's
build can read the secret too: see step 4 before you build pull requests from people outside your organisation.

Give each repository's build projects a role that reads only that repository's secret: a build can name any secret its
role may read, and a pull request's build runs the buildspec from the pull request. Secrets Manager adds six random
characters to a secret's ARN, so the policy names it with `-??????` at the end
(`arn:aws:secretsmanager:<region>:<account>:secret:valystrum/ci/web-api-??????`), not `valystrum/ci/*`. With a
repository's secret, anyone can send scans and SBOM uploads for that repository, and a complete upload replaces its
components: if a build you did not trust ever ran with it, rotate the key.

**Rotating, listing and revoking a key.** Issuing the key again under **Settings, CI pipeline keys** rotates its
secret. Over the API, send `{"repo": "acme/web-api", "regenerateSecret": true}`; the same call without
`regenerateSecret` leaves the secret alone. The old secret is refused from that moment, so put the new one in Secrets
Manager straight away:

```bash
aws secretsmanager put-secret-value --secret-id valystrum/ci --secret-string '{"webhookSecret":"cs_wh_..."}'
```

Until you do, the scan passes with an `ERROR:` line (HTTP 401 `Invalid signature`; it fails when
`VALYSTRUM_FAIL_ON_ERROR` is true) and the SBOM build fails. Leave
Secrets Manager's automatic rotation off for this secret: only Valystrum can mint a new one.
`GET /api/v1/<org>/pipeline/config` and the same settings page list the active keys (never their secrets).
`PATCH /api/v1/<org>/pipeline/config/<keyId>` changes a key's repository or label without touching its secret, and
`DELETE` on the same path revokes it and returns its slot. Gold includes 100 repositories and Enterprise is custom.
From Gold, 25 more are an add-on at €100 a month or €1,000 a year. A new repository over the allowance is refused with
HTTP 409 `QuotaExceeded`; rotating or relabelling a key never spends a slot.

**3. Copy the files into your repository:** `valystrum-scan.sh` and `valystrum-sbom.sh` to `.valystrum/`, and the two
buildspecs from `examples/` to the root (or wherever the build projects name them). Replace `acme` in the URLs with
your organisation's slug, and `valystrum/ci` with the repository's own secret when you have several (step 2).
**Settings, CI pipeline keys** shows the scan's URL; the SBOM's is the same with `sbom/ingest` in place of
`pipeline/scan`. If your default branch is not `main`, change `VALYSTRUM_DEFAULT_BRANCH` in
`buildspec-security-scan.yml` and the `HEAD_REF` filters of the `PUSH` webhooks (steps 4 and 5) to its name. With the
wrong name no scan counts as the default branch's, so a fix closes a finding only when no branch still finds it, and
the SBOM build never runs.

**4. Create the scan's build project.**

- Source: your GitHub, GitLab or Bitbucket repository (through a CodeConnections connection), with **Git clone
  depth: Full**, so the build has the history it compares with.
- Primary source webhook events: `PULL_REQUEST_CREATED`, `PULL_REQUEST_UPDATED` and `PULL_REQUEST_REOPENED`, and
  `PUSH` for the default branch, so the default branch reads the code and decides its findings
  ([The branch](#the-branch)); without it a fix closes a finding only when no branch still finds it. Leave `PUSH` off
  when a pipeline scans the default branch through a project of its own ([The branch](#the-branch)). Turn on
  **Report build statuses to source provider**.
- Pull requests from outside your organisation: a pull request's build runs the buildspec and the `.valystrum/`
  scripts from the pull request itself, with the project's service role, so it can read the Valystrum secret, and any
  other secret that role may read (step 2). If people outside your organisation can open pull requests (a public
  repository, forks), do not let their changes build unreviewed: keep the webhook's pull request build policy, which
  CodeBuild turns on for every new project. A pull request from someone outside its approver roles (by default admin,
  maintain and write on GitHub; owner, maintainer and developer on GitLab; admin and write on Bitbucket) builds only
  after a trusted member reviews it, `.valystrum/` and the buildspec included, and comments
  `/codebuild_run(<the latest commit's SHA>)`. Narrow it to pull requests from forks if you must, but do not turn it
  off. Filtering the `PULL_REQUEST_*` events by `ACTOR_ACCOUNT_ID` with an anchored pattern of the account IDs you trust
  (`^(1234567|7654321)$`) keeps everyone else's pull requests from building at all.
- Environment: a CodeBuild-managed Linux image (`aws/codebuild/standard` or `aws/codebuild/amazonlinux`), which has
  bash, git, curl, openssl and jq. With another image, the script fetches jq when it has none (jq 1.8.2, checked
  against its SHA-256; x86_64 and arm64).
- Network: the build needs outbound HTTPS to `app.valystrum.io` for both scripts and, when the image has no jq, to
  `github.com` and the `githubusercontent.com` host GitHub redirects release downloads to. The SBOM build also needs
  the package sources its generator installs from: your npm registry, your Python package index, the Go module proxy
  and checksum database (`proxy.golang.org` and `sum.golang.org` unless `GOPROXY` and `GOSUMDB` say otherwise), or
  your Maven repositories. A build project in a VPC reaches them through a NAT gateway
  or your proxy, and reaches Secrets Manager the same way or through a VPC endpoint.
- Buildspec: `buildspec-security-scan.yml`.
- The service role also needs to write the report group: `codebuild:CreateReportGroup`, `codebuild:CreateReport`,
  `codebuild:UpdateReport` and `codebuild:BatchPutTestCases`.

**5. Create the SBOM's build project** the same way, with the buildspec `buildspec-sbom.yml`, a `PUSH` webhook
filtered to the default branch (`HEAD_REF` `^refs/heads/main$`), and an image with your project's toolchain (node and
npm, python3 with venv, go, or mvn). The example installs an npm project's dependencies first: the generator reads
the installed `node_modules`, not the lockfile alone, and an SBOM with no components fails the build. Set the
project's **concurrent build limit** to 1, so an older build that finishes late cannot put an older set back. CodeBuild
does not queue a build over that limit: a push that arrives while the SBOM build runs is not built at all, so Valystrum
keeps the older set until the next push to the default branch (start a build of the branch by hand when it must not
wait).

**6. Make the scan a merge gate.** With build statuses reported to the source provider, require the build's check in
the branch's protection rules (GitHub), with **Pipelines must succeed** among the merge checks (GitLab, which counts
the status an external CI reports), or with the merge check that requires passing builds (Bitbucket). The build fails
on a critical or high finding; only a required check stops the merge.

**7. Open a pull request.** A green first build reads `Scan complete: 0 finding(s), passed=true.`, and the SBOM build,
which runs at the next push to the default branch (step 5), logs
`Ingested N components into Valystrum as source 'acme/web-api'; removed 0 it no longer lists.` The scan then shows in
Valystrum's **Dev Pipeline** module, its findings in **Findings**, and the components under **SBOM & Licences**.

## Inside CodePipeline

A CodePipeline (V2) source action hands CodeBuild a copy of the repository, without the webhook variables the
buildspec reads. Give the pipeline what it needs:

- In the source action (`CodeStarSourceConnection`), set the output artifact format to **Full clone**
  (`OutputArtifactFormat: CODEBUILD_CLONE_REF`), so the build has git history. The CodeBuild role then needs
  `codeconnections:UseConnection` on the connection.
- In the build action, pass the source's variables as environment variables; they win over what the buildspec
  works out:

```yaml
# CloudFormation, the build stage of an AWS::CodePipeline::Pipeline (PipelineType: V2)
- Name: Valystrum
  Actions:
    - Name: SecurityScan
      ActionTypeId: { Category: Build, Owner: AWS, Provider: CodeBuild, Version: "1" }
      InputArtifacts: [{ Name: SourceOutput }]
      Configuration:
        ProjectName: !Ref ValystrumScanProject
        EnvironmentVariables: >-
          [{"name":"VALYSTRUM_REPO","value":"#{SourceVariables.FullRepositoryName}","type":"PLAINTEXT"},
           {"name":"VALYSTRUM_BRANCH","value":"#{SourceVariables.BranchName}","type":"PLAINTEXT"},
           {"name":"VALYSTRUM_COMMIT","value":"#{SourceVariables.CommitId}","type":"PLAINTEXT"}]
      RunOrder: 1
    - Name: Sbom
      ActionTypeId: { Category: Build, Owner: AWS, Provider: CodeBuild, Version: "1" }
      InputArtifacts: [{ Name: SourceOutput }]
      Configuration:
        ProjectName: !Ref ValystrumSbomProject
        EnvironmentVariables: >-
          [{"name":"VALYSTRUM_SOURCE","value":"#{SourceVariables.FullRepositoryName}","type":"PLAINTEXT"}]
      RunOrder: 1
```

The source action needs `Namespace: SourceVariables`. A pipeline started by a push scans the change against the
previous commit (CodePipeline does not hand the build the commit before the push; pass it as `VALYSTRUM_BASE` when you
have it), so a push of several commits is scanned as its last one. For a pipeline started by a pull request (a V2
trigger), also pass the pull request's target branch as `VALYSTRUM_BASE`, so the whole pull request is scanned; the
script fetches that branch when the clone lacks it.

With a CodeCommit source action, the repository's variable is `#{SourceVariables.RepositoryName}` (CodeCommit has no
`FullRepositoryName`), Full clone needs `codecommit:GitPull` on the repository for the CodeBuild role, and the key is
issued for that name. An S3 or ECR source action hands the build no git history, so the scan cannot run in that
pipeline: it fails the build with `This is not a git checkout`, a setup error. The SBOM build can run there, with
`VALYSTRUM_SOURCE` set.

A build CodePipeline starts takes its source from the pipeline, and CodeBuild reports a build status to GitHub, GitLab
or Bitbucket only for a build with that source of its own, so the pull request has no check from the pipeline to
require. For the merge gate, keep the scan's own build project with its pull request webhook (steps 4 and 6), and run
the pipeline on pushes to the default branch.

Run the `Sbom` action only in a pipeline that the default branch alone starts: the example buildspec uploads a complete
SBOM (`VALYSTRUM_COMPLETE: "true"`), so a run for a pull request or another branch would replace the source's
components in Valystrum with that branch's. Keep the pipeline's execution mode `SUPERSEDED` (the default) or `QUEUED`,
not `PARALLEL`, so two executions never upload at once.

In a pipeline, a failed action stops the execution before the stages after it. The scan fails on a critical or high
finding (with `VALYSTRUM_FAIL_ON_FINDINGS=true`), which makes it a gate before deployment. The SBOM build has no
fail-open rule, so a Valystrum outage or a refused upload stops the pipeline too: if a deployment must not wait on the
SBOM, put the `Sbom` action in a stage after it.

The buildspec exports nothing to CodePipeline. To hand the verdict to a later action, add
`exported-variables: [VALYSTRUM_PASSED, VALYSTRUM_FINDINGS, VALYSTRUM_SCAN_ID]` under the buildspec's `env`, read the
file in a `post_build` command (`set -a; . valystrum-reports/valystrum-scan.env; set +a`), and give the build action a
`Namespace`; later actions read `#{<namespace>.VALYSTRUM_SCAN_ID}`. The file is missing only after a setup error (an
empty setting, or no git checkout), which fails the build anyway.

## The security scan

### What it checks

| Check | Reads | Finds |
|---|---|---|
| Leaked secrets | the lines the change adds | AWS access keys, GitHub tokens, Stripe live secret keys and private keys (critical); Google API keys and Slack tokens (high); JWTs and generic API keys (medium: reported, never failing the build) |
| Code analysis (SAST) | each changed file, whole | weaknesses in JavaScript and TypeScript, Python, Java, Go, C#, Scala, Kotlin, Swift and Objective-C, with Opengrep and a licence-clean rule pack: an error-level rule is high, a warning medium, an info low, and command, SQL, code or eval injection, unsafe deserialisation, hard-coded credentials and SSRF are critical |
| IaC | each changed file, whole | Terraform, CloudFormation, Kubernetes, Helm and Dockerfiles, with Trivy's checks: critical, high and medium as Trivy grades them; low is not reported |

Code analysis and the IaC checks read each changed file whole, so a critical or high weakness already in a file the
change touches fails the build, even on a line the change did not touch. Secrets are looked for only in the lines the
change adds, so a secret already in the file is not reported. An AWS access key is found by its key ID (`AKIA...`):
temporary keys (`ASIA...`), a secret access key on its own and session tokens are not among the families. The secret
scan skips lock files (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `poetry.lock`, `Gemfile.lock`, `go.sum`,
`Cargo.lock`), the `node_modules`, `vendor`, `dist`, `build`, `out`, `coverage` and `.next` directories, minified,
snapshot and source-map files, and `.example`, `.sample`, `.template` and `.dist` files, and it ignores values that
read as placeholders (`example`, `xxxx`, `placeholder`, `changeme`, `your_key`, `dummy`, `redacted`, `<...>`). Ruby,
PHP, C and any language not listed get the secret check only. The two engines run side by side with 20 seconds each,
and Valystrum waits 25, so a large change can come back as `timeout` (a `WARNING:` line, see
[What blocks a merge](#what-blocks-a-merge-and-what-does-not)).

### Settings

The buildspec sets these as environment variables; `VALYSTRUM_SECRET` comes from Secrets Manager. An environment
variable set on the build project, or passed by a pipeline's build action, overrides the same name in the buildspec's
`env` section, so the scan's URL, the default branch or the secret (as a variable of type Secrets Manager) can be set
per project without editing the buildspec.

| Variable | Default | What it does |
|---|---|---|
| `VALYSTRUM_SCAN_URL` | required | `https://app.valystrum.io/api/v1/<org>/pipeline/scan` |
| `VALYSTRUM_SECRET` | required | The repository's CI secret |
| `VALYSTRUM_REPO` | the source URL's `owner/repo` | The repository as its CI key names it. It picks the key, so it must match exactly |
| `VALYSTRUM_BRANCH` | the webhook's head branch | The branch the change is on. Without it the scan gates the change but closes nothing in Valystrum |
| `VALYSTRUM_BASE` | the pull request's target branch, or the commit before a push | What the change is compared with; the previous commit when nothing names it |
| `VALYSTRUM_COMMIT` | the build's resolved commit | The commit scanned |
| `VALYSTRUM_PR_NUMBER` | from the webhook's `pr/<n>` trigger | The pull request's number |
| `VALYSTRUM_SEQUENCE` | the build number (`CODEBUILD_BUILD_NUMBER`) | A number that grows with each build: Valystrum orders a branch's scans by it. A retried build has a new number (see [The branch](#the-branch)) |
| `VALYSTRUM_DEFAULT_BRANCH` | `main` in the example; else what the clone's `origin` names | The repository's default branch: a scan of it decides a finding once it has read the file. Left unknown, the build warns |
| `VALYSTRUM_PATHS` | empty | Space-separated pathspecs that narrow the diff, and so the secret scan. The files sent for code analysis are every changed file all the same |
| `VALYSTRUM_SEND_FILES` | `true` | Send the changed files, which code analysis and the IaC checks need. Off, only secrets are scanned |
| `VALYSTRUM_FAIL_ON_FINDINGS` | `true` | Fail the build on a critical or high finding. Off, the build only reports |
| `VALYSTRUM_FAIL_ON_ERROR` | `false` | Also fail the build when the scan could not run (no answer, a refusal such as a wrong secret), the platform could not complete it, or its code and IaC scanner did not finish the files sent |
| `VALYSTRUM_JUNIT` | `valystrum-reports/junit.xml` in the example; else none | The JUnit report the report group reads. Unset, no report is written |
| `VALYSTRUM_OUT_DIR` | `valystrum-reports` | Where the outputs go: `summary.md` and `valystrum-scan.env` (`VALYSTRUM_PASSED`, `VALYSTRUM_FINDINGS`, `VALYSTRUM_SCAN_ID`) |

### The branch

The buildspec sends the pull request's source branch, or the pushed branch, and each scan that reads a file in full is
that branch's read of it. Once the repository's default branch (`VALYSTRUM_DEFAULT_BRANCH`) has read a file, its scans
decide that file's findings: a fix closes a finding when it reaches the default branch, and a pull request or an old
branch that still carries the finding neither reopens nor holds it. Until the default branch has read the file, every
branch that read it holds the finding while its latest scan still finds it, so a fix on a pull request closes a
finding only when the default branch has not read that file and no other branch still finds it. Scan the default
branch on push as well, so it reads the code.

What counts as read: a file the change deletes, for every kind of finding; a code or IaC file the script sent whole
whose engine finished and could parse it; and, for a secret, a file sent whole that the secret scan does not skip,
which Valystrum reads again in full. So a build with `VALYSTRUM_SEND_FILES` off, or whose request went without the
files (over 2 MB), closes only a deleted file's findings; a file the script did not send (over 512 KB, binary, or past
the 300-file or 1.5 MB budget) proves nothing; and an engine that timed out or failed closes nothing of its kind. A
scan reads only the files its change touches, so a finding in a file no later change touches stays open until you
resolve it.

A closed finding is resolved with the note `Fixed: not found by the code scan of <day>`, and its open remediation
items move to ready for verification (a linked Jira or ServiceNow ticket gets the note as a comment), for a person to
verify and close. A finding you accepted or marked a false positive is never closed or reopened by a scan; one you
reopened by hand stays open until a scan finds it again, and only a later scan that no longer finds it closes it.
Deleting a branch tells Valystrum nothing: a finding raised only on a branch that is never merged stays open until a
scan of the default branch reads that file, or until you resolve it. A finding recorded before Valystrum v0.21.0 has
no branch on record and closes only when a scan of the default branch reads its file. The **Dev Pipeline** module says
why each pipeline finding is still open.

A leaked secret is the exception: a scan that no longer finds it leaves its finding open, with a note asking you to
confirm you rotated it, since a secret taken out of the code is still known to whoever copied it. A renamed or moved
file takes its findings to its new path; if moving them fails, they stay open on the old path and the next scan of the
repository moves them. Resolving or accepting a finding in Valystrum does not change the gate: each build is judged on
its own findings, so the next change that still carries a critical or high one fails too.

Valystrum orders a branch's scans by the build number: a build that finishes after a newer one cannot reopen what the
newer scan closed or close what it saw, and a force-push back to an older commit is still the newer scan. A retried
build (**Retry build**) is a new build with a new number, so it counts as the branch's newest scan even when its commit
is older: retry only a branch's latest build, or start a new build of the branch, and leave the project's automatic
retry (its auto-retry limit) at 0, since an automatic retry of an older build is a new number too.

Build numbers are counted per build project. Scan each branch from one project: when two projects scan the same branch
(the pull request project with a `PUSH` event for the default branch, and a pipeline's project), Valystrum compares two
unrelated counters, and the project with the lower numbers can never close what the other saw. With a pipeline that
scans the default branch through a project of its own, leave `PUSH` off the webhook project.

Moving a repository's scans to CodeBuild from another CI, or to a new build project, starts the count again.
Valystrum compares a branch's run numbers, and CodeBuild counts from 1 in each project, while Valystrum's GitHub
Action sends its run id (an 11-digit number) from its release after v1.1, GitLab its pipeline id and Azure Pipelines
its build id. Until the new project's numbers pass the old ones, its scans of a branch the old CI or project scanned
neither close nor reopen that branch's findings (the gate still works). Renaming a build project creates a new one
too. Lift the new numbers above the old ones: in `buildspec-security-scan.yml`, change the sequence line to
`export VALYSTRUM_SEQUENCE="$((100000000000 + CODEBUILD_BUILD_NUMBER))"`, above any run number those CIs send today,
and never lower it afterwards.

### What blocks a merge, and what does not

The build fails when the scan reports any **critical or high** finding (with `VALYSTRUM_FAIL_ON_FINDINGS=true`). A
scanner outage never blocks you unless you ask it to: when the scan endpoint cannot be reached or refuses the
request, when the platform answers that it could not complete the scan, or when its code and IaC scanner did not
finish the files sent, the build passes with a message that says what did not run (an `ERROR:` line when the request
got no answer or was refused, a `WARNING:` line otherwise; in the last case what did run still fails the build on a
critical or high finding), and the report shows a test case in error naming it. `VALYSTRUM_FAIL_ON_ERROR=true` fails
the build in all three cases. Turn it on once the setup works, or a rotated secret or a plan change turns the gate off
without failing a build.

Two things fail the build whatever `VALYSTRUM_FAIL_ON_ERROR` says, because they are setup errors and not an outage: a
setting the script needs is empty (`VALYSTRUM_SCAN_URL`, `VALYSTRUM_SECRET`, `VALYSTRUM_REPO`), or the build directory
is not a git checkout. A third fails it before the script starts: CodeBuild reads the secret from Secrets Manager when
the build begins, so a secret that is missing, has no `webhookSecret` key, or that the service role may not read fails
the build (the scan and the SBOM alike, and in CodePipeline the execution stops), whatever `VALYSTRUM_FAIL_ON_ERROR`
says. A jq that is missing and cannot be fetched counts as an outage: the build passes with an error unless
`VALYSTRUM_FAIL_ON_ERROR` is true.

### Where the findings show

- The build's **Reports** tab: each finding as a failed test case (`SECRET-aws_access_key at deploy.env:1`), medium
  and low included, with its severity and message, in a report group named `<project>-valystrum-security-scan` (the
  project's name and the buildspec's report). So a build that passed can show a failed report: the build's status, not
  the report's, is the gate. The report also holds one test case, `Scan completed`, which passes when the scan ran and
  is in error when it did not run or did not finish.
- The build log: a summary table of the 20 most severe findings, and under it one line per finding, all of them.
- Valystrum: the **Dev Pipeline** module and **Findings**, one finding per repository, rule and file.
- A leaked secret also lands in **Data Security**, **Secrets**, as its masked preview, with a finding of its own in
  **Findings**.

Unlike the GitHub Action, these scripts post no comment on the pull request: the result reaches it as the build's
status (step 6), and the findings are in the Reports tab, the log and Valystrum. A detected secret is only ever shown
as a masked preview (`AKIA••••3456`), in the report, the log and Valystrum.

## The SBOM upload

### Settings

| Variable | Default | What it does |
|---|---|---|
| `VALYSTRUM_INGEST_URL` | required | `https://app.valystrum.io/api/v1/<org>/sbom/ingest` |
| `VALYSTRUM_SECRET` | required | The repository's CI secret |
| `VALYSTRUM_SOURCE` | the source URL's `owner/repo` | The label the components are grouped under. It also picks the CI key, so it must equal the repository the key was issued for |
| `VALYSTRUM_PATH` | `.` | The directory holding the manifests. Its subdirectories are not searched |
| `VALYSTRUM_ECOSYSTEM` | `auto` | Every generator whose manifest is present, or a space-separated subset of `npm python go maven` |
| `VALYSTRUM_ASSET` | empty | The name or hostname of the Valystrum asset these components belong to. Left empty, an asset whose name or hostname equals the source label is used; otherwise the components stay unlinked |
| `VALYSTRUM_COMPLETE` | `false` (the example sets `true`) | `true` when this build's SBOM is the whole of the source: the source's components it no longer lists are removed in Valystrum and their CVE findings resolve (one whose CVE still affects the version the SBOM now lists moves to that version and stays open), and a licence the SBOM no longer carries closes its finding. An SBOM with a component that has no version only adds. Only from the default branch; `false` when several builds upload parts of one source |
| `VALYSTRUM_OUT_DIR` | `valystrum-reports` | Where `sbom.cdx.json`, `valystrum-sbom.env` (`VALYSTRUM_COMPONENTS`, `VALYSTRUM_REMOVED`) and `summary.md` go |

The SBOM upload has no fail-open rule: an empty setting, a `VALYSTRUM_PATH` that is not a directory, a jq it cannot
fetch, an SBOM with no components, or an upload Valystrum refuses or does not answer after four attempts fails the
build, so a broken upload never passes as a success.

Manifests in further directories: run the script once per directory in the same build, after installing that
directory's dependencies, each run with its own `VALYSTRUM_PATH` and `VALYSTRUM_OUT_DIR` and with `VALYSTRUM_COMPLETE`
false, since every run uploads under the repository's one source and a complete one would remove the others'
components:

```yaml
      - (cd services/billing && npm ci --ignore-scripts --no-audit --no-fund)
      - VALYSTRUM_PATH=services/billing VALYSTRUM_OUT_DIR=valystrum-reports/billing VALYSTRUM_COMPLETE=false bash .valystrum/valystrum-sbom.sh
```

To remove what is gone, either generate once from a manifest that covers every module (a Maven aggregator `pom.xml`)
with `VALYSTRUM_COMPLETE` true, or give each module a source of its own: a key issued for exactly that label (each
takes a repository slot), with its secret in Secrets Manager passed to that run as `VALYSTRUM_SECRET`.

A source you no longer upload (a renamed repository, an archived one) keeps its components and their open findings,
and shows as stale under **Sources by age** on **SBOM & Licences** after 30 days. On its page, **Retire this source**
(confirmed in a second step) removes its components and resolves their open CVE and licence findings; accepted risks
and false positives stay. The same is `POST /api/v1/<org>/sbom/retire` with `{"source": "<the label>"}` from a
signed-in session. An upload under the same label later starts the source again.

A repository that still exists but rarely changes shows as stale too. To keep its SBOM fresh, start the SBOM build
project on a schedule (an Amazon EventBridge schedule whose target is the project); such a build takes the project's
own source and source version (the default branch unless the project names another), so `VALYSTRUM_SOURCE` is worked
out as on a push.

### The generators

| Manifest | Generator | How it and everything it installs are locked |
|---|---|---|
| `package-lock.json` | `@cyclonedx/cyclonedx-npm` 6.0.1 | its whole tree from a lock file carried in the script (`npm ci`), each package by its SHA-512, fetched through your npm registry; nothing is resolved |
| `poetry.lock`, `Pipfile.lock` or `requirements.txt` (the first found) | `cyclonedx-bom` 7.6.0 | its whole tree from a hashed requirements file carried in the script (`pip --require-hashes --no-deps`), each wheel by its SHA-256, in a virtual environment; nothing is resolved |
| `go.mod` | `cyclonedx-gomod` 1.12.0 | `go run` at that version builds from the module's own `go.sum`, every module checked against Go's checksum database (sum.golang.org) |
| `pom.xml` | `cyclonedx-maven-plugin` 2.9.3 | the plugin's version fixes every dependency's version, each checked against the repository's checksums (`--strict-checksums`) |

Each generator is pinned to its newest release as of 2026-10-28. The Python lock covers CPython 3.9 and later on Linux
(x86_64 and aarch64); the npm generator runs without its optional schema validators, which Valystrum does not need
because it reads the components itself, and needs Node.js 20.18 or later and npm 9 or later in the image. Several
ecosystems in one directory are merged into one component list.

The npm SBOM lists every installed package, devDependencies included unless `NODE_ENV` is `production` (the
generator's default; the script passes no `--omit`). The Python generator reads `poetry.lock`, `Pipfile.lock` or
`requirements.txt` as it stands and installs nothing: a `requirements.txt` lists only what it names, so its transitive
dependencies are in the SBOM only when the file pins them all (`pip freeze` or `pip-compile` output), and a line
without an exact version (`requests>=2`) gives a component Valystrum skips, which also makes a complete upload only
add.

## What is sent, and what never is

The scan sends the repository name, the branch, the commit, the pull request's number, the commit's committer time
(`committedAt`, never later than now), the repository's default branch (`defaultBranch`) and a number that grows with
each run (`sequence`), the diff (base64) and, with `VALYSTRUM_SEND_FILES`, the changed files' contents: at most 300
files, each at most 512 KB, about 1.5 MB in all, binary files never. A request over 2 MB goes without the files, so
only secrets are scanned, and one still over 2 MB without them is refused with HTTP 413. The SBOM upload sends the
generated bill as it is (`sbom.cdx.json`, also written to `VALYSTRUM_OUT_DIR`), of which Valystrum reads each
component's name, version, purl and licence, plus the source label, the asset name and `complete`. Each request is
signed with HMAC-SHA256 over `"<unix seconds>.<body>"` under your CI secret, in the header
`X-CS-Signature: t=<seconds>,v1=<hex>`; Valystrum accepts a signature once and within five minutes, so every retry is
signed afresh. The script retries on no answer, 408, 429 and 5xx (three
attempts for the scan, four for the SBOM). The scan waits up to two minutes for each attempt. The SBOM upload sets no
time limit of its own, so the build project's timeout (60 minutes unless you change it) bounds an upload that hangs.

The buildspecs name `valystrum-reports/` (scan) and `sbom.cdx.json` (SBOM) as the build's artifacts, but CodeBuild
keeps them only where the build has a place for them: an artifacts location in S3 on the build project, or an output
artifact on the pipeline's build action. Without one, the SBOM the build uploaded is in Valystrum only.

Never sent: the CI secret itself, other environment variables, AWS credentials, files the change did not touch, the
repository's history, or anything else in the build container. Valystrum does not keep the diff or the files: it
scans them and keeps the scan record, the findings (a secret only as its masked preview, and a one-way hash that
recognises it when it is seen again) and the SBOM's components.

## Troubleshooting

| Symptom | Cause |
|---|---|
| The pull request's build status reads `Pull request approval required for starting a build` | CodeBuild's pull request build policy (on for a new project): the author is outside its approver roles. A trusted member reviews the change, including `.valystrum/` and the buildspec, and comments `/codebuild_run(<the latest commit's SHA>)` (step 4) |
| The build fails before the scan with a Secrets Manager error | The service role cannot read the secret the buildspec names (`valystrum/ci` in the example), or the secret has no `webhookSecret` key |
| `VALYSTRUM_REPO is empty` or `VALYSTRUM_SOURCE is empty` | The build has no source URL to work it out from (a build CodePipeline started): set the variable, as under [Inside CodePipeline](#inside-codepipeline). With an S3 or CodeCommit source the URL gives a wrong name instead, and the scan and the SBOM upload are refused with HTTP 401 `Invalid signature: sign with this repository's CI key`: set the variable there too (step 1) |
| `This is not a git checkout` | A CodePipeline source without **Full clone** hands the build a zip with no history, and an S3 or ECR source action has none to hand ([Inside CodePipeline](#inside-codepipeline)) |
| `Shallow clone detected: fetching history so the diff has a base.` | The project's Git clone depth is not Full: the script fetches the history from `origin` itself, which needs the build to reach the repository. Set the depth to Full |
| `No changes to scan.` | The diff is empty: nothing changed under `VALYSTRUM_PATHS`, or the base was not reachable |
| `Base '...' is not in this clone; falling back to the previous commit` | Set the project's Git clone depth to Full; for a pull request the target branch must be fetchable from `origin` |
| `No branch named (VALYSTRUM_BRANCH)` | A build that no webhook or pipeline started: the scan gates the change but closes nothing in Valystrum |
| HTTP 401 `Invalid signature: sign with this repository's CI key` (the scan and the SBOM upload alike) | One answer for each of these, so it tells nobody which organisations exist or which repositories have a key. A wrong or rotated secret in the secret the buildspec names, or another repository's secret (each repository has its own, step 2). The organisation slug in `VALYSTRUM_SCAN_URL` or `VALYSTRUM_INGEST_URL` is wrong. Or there is no active key for this repository (there is no organisation-wide key to fall back on): step 1 was not run for it, the key was revoked, or the key names it differently: compare the key under **Settings, CI pipeline keys** with `VALYSTRUM_REPO` (for the SBOM, `VALYSTRUM_SOURCE`), which must match it exactly, case included. A renamed repository: `PATCH /api/v1/<org>/pipeline/config/<keyId>` with `{"repo": "<new name>"}`, and retire its old SBOM source ([The SBOM upload](#the-sbom-upload)). The scan's findings are kept under the repository's name: after a rename, scans under the new name do not read the findings recorded under the old one, so close those in **Findings**. Or keep the old name: leave the key as it is and set `VALYSTRUM_REPO` and `VALYSTRUM_SOURCE` on both build projects to the name it was issued for |
| HTTP 401 `Signature already used` | The same signed request reached Valystrum twice (a proxy or gateway that resends it). The scripts sign every attempt afresh, so this does not come from their own retries: run the build again |
| HTTP 401 `Signature timestamp outside the five-minute window` | The build container's clock is off by more than five minutes |
| HTTP 403 `FeatureLocked` | The organisation is below the Gold plan |
| HTTP 403 `Organization is suspended` (SBOM upload) | The organisation is suspended in Valystrum |
| HTTP 400 `No valid components found in SBOM payload` | No component in the SBOM has both a name and a version |
| HTTP 413 | The scan's request is over 2 MB even without file contents (narrow it with `VALYSTRUM_PATHS`), or the SBOM is over 8 MB |
| HTTP 429 | More than 60 scans, or 30 SBOM uploads, a minute for the organisation, counted only once a request's signature checks out; or more than 300 scan requests, or 150 SBOM requests, a minute from one address, signed or not (builds behind one NAT gateway share an address) |
| Every build warns `could not complete the scan (scan_error)` | The platform's scanner failed after it accepted the request |
| No report in the Reports tab | The service role cannot write report groups (step 4) |
| `No supported manifest found` or `0 components` | No manifest in `VALYSTRUM_PATH`, or the dependencies were not installed before the script |
| `$'\r': command not found` | The script was committed with Windows line endings: commit `.valystrum/*.sh` with LF (`*.sh text eol=lf` in `.gitattributes`) |

## Updating

Releases follow semantic versioning. To update, copy the scripts of a newer release into `.valystrum/`; the rules the
scan applies live in Valystrum and change with the platform, not with these scripts. The scripts are written from one
canonical copy in the Valystrum repository, which also serves the GitLab, Azure Pipelines and Google Cloud Build
integrations, so a fix reaches all of them at once.

Set everything through the variables and leave the copied scripts as they are, so an update is a plain copy: the note
at the top of each script about a canonical copy and a sync is for Valystrum's own repository.
`VALYSTRUM_ANNOTATIONS` (read by both scripts) and `VALYSTRUM_CODEQUALITY` (read by the scan) are for Azure Pipelines
and GitLab: leave them unset on CodeBuild. The scripts print no version, so note the release you copied them from, for
example in the commit that adds them.
