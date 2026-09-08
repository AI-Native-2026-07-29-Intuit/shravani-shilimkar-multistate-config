# Multistate AWS substrate (W6 D3)

Four CloudFormation stacks, deployed in order, that provision the AWS
substrate the W6 D1 (CI on OIDC) and W6 D2 (Argo CD GitOps) work assumed
already existed. All templates live under [`cfn/`](../cfn/), all as raw
YAML (no CDK) per the curriculum's authoring anchor.

## Stacks and deploy order

| # | Stack name | Template | Depends on |
|---|---|---|---|
| 1 | `multistate-bootstrap-dev` | [`cfn/multistate-bootstrap-dev.yaml`](../cfn/multistate-bootstrap-dev.yaml) | Existing GitHub OIDC provider (from the W6 D1 PR) |
| 2 | `multistate-network-dev` | [`cfn/multistate-network-dev.yaml`](../cfn/multistate-network-dev.yaml) | — |
| 3 | `multistate-artifacts-dev` | [`cfn/multistate-artifacts-dev.yaml`](../cfn/multistate-artifacts-dev.yaml) | — |
| 4 | `multistate-app-dev` | [`cfn/multistate-app-dev.yaml`](../cfn/multistate-app-dev.yaml) | `multistate-network-dev` (`!ImportValue` on `VpcId`, `PrivateSubnets`, `AppSgId`) |

Stacks 2, 3, and 4 have no dependency on stack 1 beyond it having deployed
first (bootstrap only provisions the CI identity). Stacks 2 and 3 have no
cross-dependency on each other and can deploy in either order (or in
parallel); stack 4 must come after stack 2 because it imports three of
its outputs by name. `multistate-artifacts-dev` isn't consumed by any of
today's other three stacks — it exists for W5 D4's SAM artefacts and W6
D4's Lambda package, both landing later.

## The DB secret: stack-managed, not out-of-band

`DbMasterSecret` in `multistate-app-dev.yaml` is an
`AWS::SecretsManager::Secret` resource with `GenerateSecretString` — the
stack creates and randomly generates the credential itself (32 chars,
`multistate_master` username baked into the template), CFN never sees the
plaintext, and the secret carries `DeletionPolicy: Retain` +
`UpdateReplacePolicy: Retain` alongside the RDS instance it belongs to.
There is **no** separate `aws secretsmanager create-secret` step to run
before deploying this stack — an earlier draft of this doc had one; the
canonical Task 3 template supersedes it. `DbInstance` still resolves both
`MasterUsername` and `MasterUserPassword` via the
`{{resolve:secretsmanager:multistate/dev/db-master:SecretString:...}}`
dynamic reference (never a `NoEcho` Parameter — see the audit notes
below); `DbInstance` also carries `DependsOn: DbMasterSecret` so the
secret is guaranteed to exist in Secrets Manager before CFN tries to
resolve that reference within the same stack deployment.

## Deploy flow: ChangeSet, every time

No stack is ever created or updated with `aws cloudformation deploy` or a
direct `create-stack`/`update-stack` call. Every deploy — first create and
every subsequent update — goes through:

```bash
STACK=multistate-network-dev
TEMPLATE=cfn/multistate-network-dev.yaml
CHANGESET="${STACK}-$(date +%Y%m%d%H%M%S)"

aws cloudformation create-change-set \
  --stack-name "$STACK" \
  --change-set-name "$CHANGESET" \
  --template-body "file://$TEMPLATE" \
  --capabilities CAPABILITY_NAMED_IAM \
  --change-set-type $(aws cloudformation describe-stacks --stack-name "$STACK" >/dev/null 2>&1 && echo UPDATE || echo CREATE)

aws cloudformation wait change-set-create-complete \
  --stack-name "$STACK" --change-set-name "$CHANGESET"

aws cloudformation describe-change-set \
  --stack-name "$STACK" --change-set-name "$CHANGESET" \
  --query 'Changes[].ResourceChange.{Action:Action,Resource:LogicalResourceId,Type:ResourceType,Replacement:Replacement}' \
  --output table
# ^ this table is what goes in the PR body as the resource-level diff.

aws cloudformation execute-change-set \
  --stack-name "$STACK" --change-set-name "$CHANGESET"
```

`--capabilities CAPABILITY_NAMED_IAM` is required for
`multistate-bootstrap-dev` (creates `multistate-api-cfn-deploy`) since it
names its IAM role explicitly; it's a no-op for the other three stacks
(none of them create IAM resources) and is safe to pass on every deploy
regardless.

The `Replacement` column in the describe-change-set output is the one
line item to scrutinize on every PR — anything that resolves to `True` on
`DbInstance`, `DbMasterSecret`, `MultistateArtifactsBucket`, or
`BootstrapBucket` is a signal to stop and confirm
`DeletionPolicy`/`UpdateReplacePolicy` will actually protect the data
before executing.

## In-place UPDATE drill (Task 4)

Proves the ChangeSet flow's `Replacement` column actually distinguishes
"modify in place" from "tear down and recreate." Pick a change to
`multistate-network-dev.yaml` that CloudFormation can apply without
replacing anything — e.g. renaming the `Project` tag value, or widening
`VpcCidr`'s `AllowedPattern` (not the CIDR itself, which *would* force VPC
replacement) — then run the same flow as any other update:

```bash
aws cloudformation create-change-set \
  --stack-name multistate-network-dev \
  --change-set-name network-tag-update \
  --change-set-type UPDATE \
  --template-body file://cfn/multistate-network-dev.yaml \
  --region us-east-1

aws cloudformation describe-change-set \
  --stack-name multistate-network-dev --change-set-name network-tag-update \
  --query 'Changes[].ResourceChange.{Resource:LogicalResourceId,Action:Action,Replacement:Replacement}' \
  --output table
# paste this table into the PR body

aws cloudformation execute-change-set \
  --stack-name multistate-network-dev --change-set-name network-tag-update
```

Expected: every row is `Action: Modify` with `Replacement: False`. Any
row that comes back `Replacement: True` (or `Conditional`) is a stop sign
— on this stack that would mean something like `VpcCidr` itself changing,
which replaces `Vpc` and cascades into every subnet, route table, and NAT
gateway depending on it.

## Cross-stack reference health check (Task 3)

After `multistate-network-dev` and `multistate-app-dev` are both up,
prove the `!ImportValue` safety net actually works by trying to delete
the stack the app stack depends on:

```bash
aws cloudformation delete-stack --stack-name multistate-network-dev
aws cloudformation describe-stack-events --stack-name multistate-network-dev \
  --query 'StackEvents[0].[ResourceStatus,ResourceStatusReason]'
```

Expected: `DELETE_FAILED`, with a reason like
`Export multistate-network-dev-PrivateSubnets cannot be deleted as it is
in use by multistate-app-dev`. CloudFormation refuses the delete outright
— no resources are torn down. Roll back the attempt with:

```bash
aws cloudformation delete-stack --stack-name multistate-network-dev --retain-resources ""
# or, more simply, just leave it: a DELETE_FAILED stack with nothing
# actually deleted needs no rollback action beyond not retrying the delete.
```

This is the concrete payoff of exporting by name instead of hardcoding
subnet/SG IDs in `multistate-app-dev.yaml`: a network rebuild can't
silently orphan the app stack's references, because CFN won't let the
network stack disappear while something still imports its outputs.

## CI gate

[`.github/workflows/cfn-validate.yml`](../.github/workflows/cfn-validate.yml)
runs a single `validate` job, sequentially, on every PR touching `cfn/`
(actions pinned by commit SHA, not tag, to close the supply-chain gap a
mutable tag leaves open):

- `cfn-lint==1.10.3` — schema/intrinsic-function correctness against all
  four templates in one invocation. Three checks are explicitly ignored:
  `W3691` (RDS engine-version deprecation) since AWS retires Postgres
  minor versions on its own cadence, faster than a CI pin can track, and
  the real check for "is this version installable" is
  `aws rds describe-db-engine-versions` at deploy time; `W7001` (unused
  Mapping) for `EnvToReplicas` in `multistate-network-dev.yaml`, which
  documents per-env scale hints for operators/other IaC rather than being
  consumed by an intrinsic function inside this template; and `W1020`
  (unnecessary `Fn::Sub`) on `DbInstance`'s dynamic-reference credentials.
- `cfn-nag 0.8.10` — security-posture scan across `cfn/`. No
  `--fail-on-warnings`: several `WARN`s here are accepted, reasoned-through
  risk (the table below), not oversights — failing the build on every
  `WARN` would force silencing the scanner instead of reading its output.
  Zero `FAIL`s is still a hard gate; `cfn_nag_scan` exits non-zero on any
  `FAIL` regardless of the warning-tolerance setting.
- `aws cloudformation validate-template` — one step per template (four
  total), using the `multistate-api-cfn-deploy` role via OIDC — the same
  role a real deploy job would use, but this workflow only ever calls the
  read-only `validate-template` action, never `create-change-set` or
  `execute-change-set`.

This workflow only lints/validates. The actual `create-change-set` →
`execute-change-set` flow is a separate, manually-triggered step (see
[Deploy flow](#deploy-flow-changeset-every-time) above) so every template
change is reviewed as a diff before it touches a real stack.

`cfn-validate`'s `cfn-lint + cfn-nag + aws validate-template` check is
configured as a **required status check** on `w6d2-implementation` — the
repo's actual default branch, and the base every PR in this repo targets
(the deliverable's "main" is this branch in practice). Set via:

```bash
cat <<'JSON' | gh api -X PUT \
  repos/AI-Native-2026-07-29-Intuit/shravani-shilimkar-multistate-config/branches/w6d2-implementation/protection \
  --input -
{
  "required_status_checks": {
    "strict": false,
    "contexts": ["cfn-lint + cfn-nag + aws validate-template"]
  },
  "enforce_admins": false,
  "required_pull_request_reviews": null,
  "restrictions": null
}
JSON
```

This is a repo-settings change (GitHub's branch-protection API), not
something expressible in the workflow YAML itself — it was applied
directly via the GitHub API rather than shipped in this PR's diff. PRs
into `w6d2-implementation` now show the check as required and block
merge until it reports success (it will read "Expected — Waiting for
status to be reported" until `cfn-validate` has run at least once).

### Accepted cfn-nag WARNs

| Rule | Resource | Why it's accepted |
|---|---|---|
| W33 | `PublicSubnet{A,B,C}` | `MapPublicIpOnLaunch: true` is the point of a public subnet; the private subnets (where the app and DB actually run) don't have it. |
| W5 | `MultistateAppSecurityGroup` egress | Egress to `0.0.0.0/0` is restricted to port 443 only (HTTPS to ECR/STS/Secrets Manager) — ingress is the side that's actually locked down (VPC-CIDR-only on port 8080, no `0.0.0.0/0` ingress anywhere). |
| W28 | `CfnDeployRole`, `MultistateAppSecurityGroup`, `DbInstance` | Names are pinned deliberately — the GitHub Actions trust policy references `CfnDeployRole`'s name directly, the app SG's name is a stable operational label, and `DbInstance`'s identifier is what appears in the RDS console/CLI day to day. A CFN-generated random suffix on any of these would break something a human or another system depends on by name. |
| W35 | `BootstrapBucket`, `MultistateArtifactsBucket` | Access logging deferred: needs a dedicated log-target bucket this deliverable doesn't otherwise require. Tracked as a fast-follow, not shipped blocking this PR. |
| W60 | `Vpc` | Flow logs deferred for the same reason — out of scope for the network topology this deliverable asks for. |
| W77 | `DbMasterSecret` | No explicit `KmsKeyId` — encrypts with the AWS-managed `aws/secretsmanager` key. Sufficient for a dev-tier secret with no cross-account sharing requirement; a customer-managed KMS key is the staging/prod upgrade path. |

`FAIL`s are not accepted under any rationale — three were found and fixed
during authoring (see audit notes below).

## Drift detection

```bash
aws cloudformation detect-stack-drift --stack-name multistate-artifacts-dev
# poll:
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <id-from-above>
# once StackDriftStatus is IN_SYNC or DRIFTED:
aws cloudformation describe-stack-resource-drifts \
  --stack-name multistate-artifacts-dev \
  --query "StackResourceDrifts[?StackResourceDriftStatus!='IN_SYNC']"
```

**Task 4 drill:** add a tag to `MultistateArtifactsBucket` directly in the
S3 console (a stand-in for the "someone hotfixed prod through the
console" scenario). Expected `describe-stack-resource-drifts` output —
the 4-6 lines that go in the PR body:

```json
[
  {
    "StackResourceDriftStatus": "MODIFIED",
    "LogicalResourceId": "MultistateArtifactsBucket",
    "PropertyDifferences": [
      {"PropertyPath": "/Tags/2", "ExpectedValue": null, "ActualValue": "{\"Key\":\"...\",\"Value\":\"...\"}", "DifferenceType": "ADD"}
    ]
  }
]
```

The fix is never "update the template to match the drift" — revert the
console edit (remove the tag) and re-run `detect-stack-drift`; it should
report back `StackDriftStatus: IN_SYNC`. If the drift were something
worth keeping, the correct move is still to add it to the template and
run it through a ChangeSet, not to leave the console the source of truth.

## Audit: cfn-author Skill output vs. the cohort checklist

The `cfn-author` Claude Skill was not available in this authoring
environment, so these four templates were hand-authored directly against
the same checklist the Skill's output is normally audited against. Recording
the checklist and the concrete failure modes it guards against here, since
that's the artefact Task 4 actually asks for regardless of which path
produced the first draft:

- **OIDC trust policy: `StringEquals` on `aud` — never `StringLike`,
  even though `sub` legitimately needs it.** `CfnDeployRole`'s trust
  policy (`multistate-bootstrap-dev.yaml`) uses `StringEquals` on
  `token.actions.githubusercontent.com:aud: sts.amazonaws.com` — that
  claim is a single fixed value, so `StringLike` there buys nothing and
  is the exact "sometimes ships `StringLike` on the aud claim" quirk this
  audit step exists to catch. `sub`, by contrast, correctly uses
  `StringLike` because it legitimately needs to match two patterns
  (`repo:uptimecrew/multistate-config:ref:refs/heads/main` and
  `:pull_request`) — the audit distinction isn't "StringLike is always
  wrong," it's "StringLike only where the claim actually varies, and
  scoped to the exact repo, never a wildcard org/repo like
  `repo:uptimecrew/*`." (No IRSA role exists in this stack set yet — that
  lands on W6 D5 when the app pod's ServiceAccount is wired up; the same
  StringEquals-on-fixed-claims discipline applies there, on both `aud`
  and the exact namespace:service-account `sub` subject.)
- **DB credentials: Secrets Manager, never `NoEcho: true` Parameter.**
  `DbMasterSecret` (`multistate-app-dev.yaml`) is a
  `AWS::SecretsManager::Secret` with `GenerateSecretString` — CFN never
  handles the plaintext value at all, only the secret's ARN — and
  `DbInstance` resolves both `MasterUsername` and `MasterUserPassword`
  via `{{resolve:secretsmanager:multistate/dev/db-master:SecretString:...}}`.
  A `NoEcho` Parameter still places the value in the change-set payload
  and stack parameter history; neither the generated-secret pattern nor
  the dynamic reference ever puts the credential in the template, the
  change-set, or CloudTrail.
- **Stateful-resource deletion: both `DeletionPolicy: Retain` AND
  `UpdateReplacePolicy: Retain`.** `MultistateArtifactsBucket`
  (`multistate-artifacts-dev.yaml`), `BootstrapBucket`
  (`multistate-bootstrap-dev.yaml`), and `DbInstance` + `DbMasterSecret`
  (`multistate-app-dev.yaml`) all set both policies. `DeletionPolicy`
  alone only protects against a stack delete; a property change that
  forces replacement (e.g. `BucketName`, or an RDS engine-version bump
  that AWS treats as requiring replacement) is a separate code path that
  `UpdateReplacePolicy` is the one guarding.
- **IAM `Action`/`Resource` scoping.** Every policy statement across all
  four templates enumerates explicit actions (no `*` service-wide
  wildcards) and scopes `Resource` to a specific ARN or ARN prefix — e.g.
  `CfnDeployRole`'s `iam:PassRole` is scoped to `role/multistate-*` AND
  gated with `iam:PassedToService: cloudformation.amazonaws.com`, and its
  CloudFormation actions are scoped to `stack/multistate-*` and
  `changeSet/multistate-*`, not `*`.

Three real `cfn-nag` `FAIL`s were caught and fixed during local scanning
(see [Accepted cfn-nag WARNs](#accepted-cfn-nag-warns) above for what
was *not* fixed and why):

1. `DbInstance` originally had a defaulted `DbMasterUsername` Parameter
   (F24) — fixed by resolving the username from the same secret as the
   password.
2. `DbInstance` originally had `DeletionProtection: false` (F80) — fixed
   by setting it `true`.
3. `DbSecurityGroup` had no `SecurityGroupEgress` block at all (F1000 —
   an SG with no explicit egress defaults to allow-all outbound) — fixed
   with an explicit deny-everywhere egress rule, since an RDS instance
   never needs to initiate outbound connections.

All three are exactly the class of finding this audit step exists to
catch — none would have shown up from reading the template casually,
only from running the scanner.

## Known limitation of this pass: no AWS account access

This authoring environment has no AWS credentials
(`aws sts get-caller-identity` returns `NoCredentials`). Everything that
requires an AWS API call against a real account was **not** run here and
is marked complete on the strength of the template/CI authoring plus
local static analysis (`cfn-lint`, `cfn-nag`) only. Concretely, per task:

- **Task 1 (bootstrap stack).** `cfn/multistate-bootstrap-dev.yaml` is
  authored to the letter of the reference template, and passes `cfn-lint`
  (0 errors) and `cfn-nag` (0 `FAIL`s). Not done: the actual
  `create-change-set` → `describe-change-set` → `execute-change-set` →
  `wait stack-create-complete` sequence, and confirming via
  `describe-stacks` that the stack reaches `CREATE_COMPLETE` with
  `BootstrapBucketName` + `CfnDeployRoleArn` present in `Outputs`. The
  exact commands are in [Deploy flow](#deploy-flow-changeset-every-time)
  above — run those against a real account to close this out.
- **Task 2 (network stack).** `cfn/multistate-network-dev.yaml` is
  authored to the letter of the reference template and passes `cfn-lint`
  (0 errors, `W7001` on the intentionally-unused `EnvToReplicas` Mapping
  excluded) and `cfn-nag` (0 `FAIL`s). Not done: the
  `create-change-set --change-set-type CREATE` → `execute-change-set`
  sequence; confirming `describe-stacks --stack-name
  multistate-network-dev` reaches `CREATE_COMPLETE`; confirming
  `aws ec2 describe-vpcs` shows the `10.43.0.0/16` VPC; and confirming
  `aws cloudformation list-exports` lists the four
  `multistate-network-dev-*` exports (`VpcId`, `PublicSubnets`,
  `PrivateSubnets`, `AppSgId`).
- **Task 3 (app + artifacts stacks).** `cfn/multistate-app-dev.yaml` and
  `cfn/multistate-artifacts-dev.yaml` are authored to the letter of the
  reference templates and pass `cfn-lint` (0 errors, `W1020` on the
  stylistic-only unnecessary-`Fn::Sub` finding excluded) and `cfn-nag`
  (0 `FAIL`s after the three fixes above). Not done: the ChangeSet deploy
  for both stacks; confirming `describe-stacks --stack-name
  multistate-app-dev` reaches `CREATE_COMPLETE`; confirming
  `aws s3api get-public-access-block --bucket
  uptimecrew-multistate-artifacts-dev-<account-id>` shows all four
  toggles `true`; confirming `aws s3api get-bucket-policy` on that bucket
  contains the `aws:SecureTransport: false` → `Deny` statement; and the
  [cross-stack reference health check](#cross-stack-reference-health-check-task-3)
  above — attempting `delete-stack` on `multistate-network-dev` while
  `multistate-app-dev` exists and confirming CFN refuses with
  `DELETE_FAILED` / "Export ... is in use by stack". The `!ImportValue`
  wiring between `multistate-network-dev` → `multistate-app-dev` (on
  `VpcId`, `PrivateSubnets`, `AppSgId`) has therefore not been exercised
  against real exports either.
- **Task 4 (CI + drift + ChangeSet-UPDATE + Skill audit).** Branch
  protection on `w6d2-implementation` now requires the `cfn-lint +
  cfn-nag + aws validate-template` status check (applied via the GitHub
  API — see [CI gate](#ci-gate) above), so that piece is done. Still
  not done, all for the same reason (no AWS account access):
  `cfn-validate.yml` itself has not run in GitHub Actions yet (needs
  `secrets.AWS_ACCOUNT_ID` set and the OIDC role from Task 1 to exist
  first) — locally, the exact pinned versions it uses
  (`cfn-lint==1.10.3`, `cfn-nag 0.8.10`) were run against all four
  templates: 0 lint errors, 0 nag `FAIL`s. Also not done: the drift
  drill in [Drift detection](#drift-detection) above — adding a real tag
  via the S3 console, confirming `DRIFTED`, and confirming the revert
  returns `IN_SYNC`; and the
  [in-place UPDATE drill](#in-place-update-drill-task-4) above —
  confirming a real `create-change-set --change-set-type UPDATE` on
  `multistate-network-dev` comes back with `Replacement: False` on every
  row. The Skill-output audit itself is unaffected by AWS access — the
  `cfn-author` Skill was not available in this environment (see below),
  so the audit was performed by hand-authoring against the same
  checklist instead.

Once a real account is available: deploy the bootstrap stack first (it's
the trust anchor everything else assumes), confirm its `Outputs`, then
run the same ChangeSet flow for the network, artifacts, and app stacks in
that order, and finally let `cfn-validate.yml` run for real on the PR.
