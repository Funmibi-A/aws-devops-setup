# CI Pipeline on AWS: CodeCommit → CodeBuild → CodePipeline

A continuous integration pipeline that turns every push to a CodeCommit repository into a tested, packaged build artifact in S3, with no manual steps.

- **Region:** `eu-west-1` (Ireland)
- **Application:** small Node.js module with a unit test
- **Repository:** `ci-demo-repo` in AWS CodeCommit (eu-west-1); exported ZIP (`ci-demo.zip`) submitted with this README

---

## 1. Task checklist

| # | Task | Done | Evidence |
|---|---|---|---|
| 1 | Environment setup: IAM roles for CodeBuild and CodePipeline | Yes | Section 4; `screenshots/01-*.png` |
| 2 | Create a CodeCommit repository and clone it | Yes | `screenshots/02-codecommit-repo-files.png` |
| 3 | Application code plus `buildspec.yml` | Yes | Section 2; `screenshots/03-local-test-pass.png` |
| 4 | Commit and push to CodeCommit | Yes | Repository link above |
| 5 | CodeBuild project with runtime, linked to the repo | Yes | Section 3; `screenshots/05-*.png` |
| 6 | CodePipeline with CodeCommit source and CodeBuild build | Yes | `screenshots/06-pipeline-settings.png` |
| 7 | Automatic trigger, build commands run, artifact produced | Yes | Section 7; `screenshots/07-*.png` |

---

## 2. Architecture and repository contents

```
git push (main)
     │
     ▼
CodeCommit repo ──► EventBridge rule ──► CodePipeline (ci-demo-pipeline)
 (ci-demo-repo)    (detects changes        │
                    on main)               ├─ Source stage: fetch code from CodeCommit
                                           └─ Build stage:  CodeBuild (ci-demo-build)
                                                   │  runs buildspec.yml
                                                   ▼
                                        S3 bucket: build artifact
```

| Resource | Name |
|---|---|
| CodeCommit repository | `ci-demo-repo` (branch `main`) |
| CodeBuild project | `ci-demo-build` |
| CodePipeline | `ci-demo-pipeline` |
| Artifact bucket | `ci-demo-artifacts-<ACCOUNT_ID>` (public access blocked) |
| EventBridge rule | `codepipeline-cidemo-main-118185-rule` |

| File | Purpose |
|---|---|
| `index.js` | Application code: an `add(a, b)` function |
| `index.test.js` | Unit test: asserts `add(2, 3)` equals `5` |
| `package.json` | Defines the `test` and `build` scripts |
| `buildspec.yml` | Tells CodeBuild what to run, phase by phase |

```yaml
version: 0.2

env:
  variables:
    APP_ENV: "ci"
    NODE_ENV: "test"

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - echo "Installing dependencies"
      - npm install
  pre_build:
    commands:
      - echo "Running tests"
      - npm test
  build:
    commands:
      - echo "Build started"
      - npm run build
  post_build:
    commands:
      - echo "Build finished"

artifacts:
  files:
    - '**/*'
  exclude-paths:
    - '.git/**/*'
  name: ci-demo-build
```

Tests run in `pre_build`. If `npm test` fails, the build stops there and no artifact is produced.

---

## 3. Build environment and variables

| Variable | Value | Purpose |
|---|---|---|
| `APP_ENV` | `ci` | Labels the environment the build runs in |
| `NODE_ENV` | `test` | Node.js runtime mode |

Both are defined in `buildspec.yml`. No secrets are used in the build.

| CodeBuild setting | Value |
|---|---|
| Source | AWS CodeCommit, `ci-demo-repo`, branch `main` |
| Environment | Managed image, Amazon Linux, Standard runtime (`aws/codebuild/amazonlinux-x86_64-standard:6.0`) |
| Node.js runtime | 20 (declared in `buildspec.yml`) |
| Service role | `ci-demo-codebuild-role` |
| Logs | CloudWatch Logs, group `/aws/codebuild/ci-demo-build` |

---

## 4. IAM roles (least privilege)

Each service has its own role. A role has a **trust policy** (which service may assume it) and a **permissions policy** (what it may do). Permissions are scoped to specific resource ARNs rather than `*`.

### `ci-demo-codebuild-role`

Trust policy: only `codebuild.amazonaws.com` can assume this role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:eu-west-1:<ACCOUNT_ID>:log-group:/aws/codebuild/ci-demo-build*"
    },
    {
      "Effect": "Allow",
      "Action": ["codecommit:GitPull"],
      "Resource": "arn:aws:codecommit:eu-west-1:<ACCOUNT_ID>:ci-demo-repo"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion", "s3:PutObject", "s3:GetBucketLocation"],
      "Resource": [
        "arn:aws:s3:::ci-demo-artifacts-<ACCOUNT_ID>",
        "arn:aws:s3:::ci-demo-artifacts-<ACCOUNT_ID>/*"
      ]
    }
  ]
}
```

| Permission | Why |
|---|---|
| CloudWatch Logs actions | Write build logs |
| `codecommit:GitPull` on `ci-demo-repo` | Download source (used by the standalone test build) |
| S3 actions on the artifact bucket | Read inputs and write output |

### `ci-demo-codepipeline-role`

Trust policy: only `codepipeline.amazonaws.com` can assume this role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codecommit:GetRepository",
        "codecommit:GetBranch",
        "codecommit:GetCommit",
        "codecommit:UploadArchive",
        "codecommit:GetUploadArchiveStatus",
        "codecommit:CancelUploadArchive"
      ],
      "Resource": "arn:aws:codecommit:eu-west-1:<ACCOUNT_ID>:ci-demo-repo"
    },
    {
      "Effect": "Allow",
      "Action": ["codebuild:StartBuild", "codebuild:BatchGetBuilds"],
      "Resource": "arn:aws:codebuild:eu-west-1:<ACCOUNT_ID>:project/ci-demo-build"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion", "s3:PutObject", "s3:GetBucketLocation"],
      "Resource": [
        "arn:aws:s3:::ci-demo-artifacts-<ACCOUNT_ID>",
        "arn:aws:s3:::ci-demo-artifacts-<ACCOUNT_ID>/*"
      ]
    }
  ]
}
```

| Permission | Why |
|---|---|
| CodeCommit actions on `ci-demo-repo` | Source stage reads the branch and packages the code |
| `codebuild:StartBuild`, `BatchGetBuilds` on `ci-demo-build` | Build stage starts the build and tracks it |
| S3 actions on the artifact bucket | Passes artifacts between stages |

---

## 5. Pipeline configuration

- **Source:** AWS CodeCommit, `ci-demo-repo`, branch `main`, artifact format `CODE_ZIP`
- **Change detection:** EventBridge rule `codepipeline-cidemo-main-118185-rule`, which listens for `referenceCreated` and `referenceUpdated` on branch `main` and targets `ci-demo-pipeline`
- **Build:** AWS CodeBuild, project `ci-demo-build`
- **Test and Deploy stages:** skipped
- **Service role:** `ci-demo-codepipeline-role`

---

## 6. Evidence

| Evidence | File |
|---|---|
| **Pipeline: Source and Build both Succeeded** | `screenshots/07-pipeline-succeeded.png` |
| **Build artifact in S3** | `screenshots/07-s3-artifact.png` |
| S3 pipeline folders (Source and Build outputs) | `screenshots/07-s3-pipeline-folders.png` |
| CodeCommit repository with the application files | `screenshots/02-codecommit-repo-files.png` |
| Local test passing before the first push | `screenshots/03-local-test-pass.png` |
| CLI identity and region | `screenshots/00-caller-identity.png` |
| IAM roles created and verified | `screenshots/01-*.png` |
| CodeBuild project configuration | `screenshots/05-codebuild-name-source.png`, `screenshots/05-codebuild-project-created.png` |
| Standalone build succeeded | `screenshots/05-build-succeeded.png` |
| Pipeline settings | `screenshots/06-pipeline-settings.png` |
| Full run history with triggers | `screenshots/07-executions-history.png` |
| Deliberate failure: Build stage failed | `screenshots/07-pipeline-build-failed.png` |
| Failure reason in the build log (`5 !== 6`) | `screenshots/07-build-log-assertion-error.png` |
| Recovery after the fix: both stages succeeded | `screenshots/07-pipeline-recovered-after-fix.png` |

Artifacts are stored in `s3://ci-demo-artifacts-<ACCOUNT_ID>/ci-demo-pipeline/`: `SourceArti/` holds the zipped source and `BuildArtif/` holds the build output. CodePipeline saves artifacts as ZIP archives under generated names, so the objects have no `.zip` extension. Each successful run adds one object to `BuildArtif/` and failed runs add none; `07-s3-artifact.png` was taken when the folder held three.

---

## 7. Automation test results

Times are UTC+1, 25 Oct 2026. Full history: `screenshots/07-executions-history.png`.

| Time | Trigger | Result | Notes |
|---|---|---|---|
| 16:48 | Pipeline creation | Failed | First run; Source stage failed within 2 seconds |
| 16:57 | Manual retry | Failed | Source stage: role missing `codecommit:GetRepository` (section 8) |
| 17:27 | Manual | Failed | Invalid Source output format (section 8) |
| 17:50 | Manual | Succeeded | First green run; first artifact |
| 18:04 | **Automatic (`CloudWatchEvent`)** | Succeeded | Started by a `git push` (commit "Trigger pipeline with a push") |
| 18:22 | Automatic | Build failed | Commit "Break test deliberately"; failed in `pre_build`, **no artifact created** |
| 18:39 | Automatic | Succeeded | Commit "Fix test"; new artifact created |
| 20:47 | Automatic | Succeeded | Commit adding this README |

The run at 18:22 shows the pipeline acting as a quality gate: Source succeeded, the test failed in the Build stage, and nothing was written to `BuildArtif/`.

---

## 8. Problems encountered and fixes

| Problem | Cause | Fix |
|---|---|---|
| Source stage: not authorized for `codecommit:GetRepository` | Action missing from the CodePipeline policy | Added it, scoped to the one repository |
| Console set the Source output format to full clone (`CODEBUILD_CLONE_REF`) | Full clone needs permissions the role deliberately lacks | Changed the format to zip |
| Console edit did not save the change | Edit was not applied | Exported the pipeline with `get-pipeline`, edited the JSON with `jq`, applied it with `update-pipeline` |
| Run failed: `CODEPIPELINE_DEFAULT` is an invalid output format | Console label "CodePipeline default" is stored as `CODE_ZIP` | Used `CODE_ZIP`, as the error message listed |
| Literal `REGION` and `ACCOUNT_ID` in policy files | Placeholders in several places, including the bucket name | Generated files with shell variables and checked with `grep` |

## 9. What I learned

- A service role has a trust policy (who can use it) and a permissions policy (what it can do); least privilege means scoping the second to specific ARNs.
- IAM error messages name the exact missing action, so reading them beats guessing.
- Putting tests in `pre_build` makes the pipeline a quality gate: a failing test blocks the artifact.
- Console defaults need reviewing against a least-privilege design.
