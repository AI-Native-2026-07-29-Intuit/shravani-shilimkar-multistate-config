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
| 4 | `multistate-app-dev` | [`cfn/multistate-app-dev.yaml`](../cfn/multistate-app-dev.yaml) | `multistate-network-dev` (`!ImportValue`), `multistate-artifacts-dev` (`!ImportValue`), the `multistate/dev/db-master` secret |

Stacks 2 and 3 have no cross-dependency and can deploy in either order (or
in parallel); stack 4 must come after both because it imports each of
their outputs by name.

## One-time prerequisite: the DB secret

`multistate-app-dev` resolves **both** the RDS master username and password
from Secrets Manager via `{{resolve:secretsmanager:...}}` dynamic
references — never a `NoEcho: true` Parameter (see the template's header
comment and the audit notes below). The secret must therefore hold both
keys:

```bash
aws secretsmanager create-secret \
  --name multistate/dev/db-master \
  --secret-string '{"username": "multistate_admin", "password": "REPLACE_ME_LOCAL_DEV_ONLY"}' \
  --region us-east-1
```

This is a slightly wider secret payload than the workshop's example
(`{"password": "..."}` only) — pulling the username from the same secret
closes a cfn-nag F24 finding (RDS master username must not be a plaintext
string or a defaulted Parameter) the narrower payload would leave open.

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
`multistate-bootstrap-dev` (creates `multistate-api-cfn-deploy`) and
`multistate-app-dev` (creates `multistate-app-irsa-<env>`) since both name
their IAM roles explicitly; it's a no-op for the network and artifact
stacks and is safe to pass on every deploy.

The `Replacement` column in the describe-change-set output is the one
line item to scrutinize on every PR — anything that resolves to `True` on
`DbInstance`, `ArtifactBucket`, or `BootstrapBucket` is a signal to stop
and confirm `DeletionPolicy`/`UpdateReplacePolicy` will actually protect
the data before executing.

## CI gate

[`.github/workflows/cfn-validate.yml`](../.github/workflows/cfn-validate.yml)
runs on every PR touching `cfn/`:

- `cfn-lint` (>=1.20) — schema/intrinsic-function correctness. `W3691`
  (RDS engine-version deprecation) is explicitly ignored: AWS retires
  Postgres minor versions on its own cadence, faster than a CI pin can
  track, and the real check for "is this version installable" is
  `aws rds describe-db-engine-versions` at deploy time.
- `cfn-nag` (>=0.8.10) — security-posture scan. Zero `FAIL`s are required
  to merge; the accepted `WARN`s are listed below with rationale.
- `aws cloudformation validate-template` — per-template syntax validation
  using the same `multistate-api-cfn-deploy` OIDC role the deploy job
  uses, scoped read-only by the actions it's granted.

This workflow only lints/validates. The actual `create-change-set` →
`execute-change-set` flow is a separate, manually-triggered job so every
template change is reviewed as a diff before it touches a real stack.

### Accepted cfn-nag WARNs

| Rule | Resource | Why it's accepted |
|---|---|---|
| W33 | `PublicSubnet{A,B,C}` | `MapPublicIpOnLaunch: true` is the point of a public subnet; the private subnets (where the app and DB actually run) don't have it. |
| W40 / W5 | `ApplicationSecurityGroup` egress | Egress is intentionally open; ingress is the restricted side (VPC-CIDR-only on port 8080). Private-subnet workloads still only reach the internet through the env's NAT gateway(s). |
| W28 | `CfnDeployRole`, `AppIrsaRole`, `ApplicationSecurityGroup` | Names are pinned deliberately — the GitHub Actions trust policy and the Kubernetes IRSA ServiceAccount annotation both reference these names directly, so a CFN-generated random suffix would break the OIDC trust wiring on every stack recreate. |
| W35 | `BootstrapBucket`, `ArtifactBucket` | Access logging deferred: needs a dedicated log-target bucket this deliverable doesn't otherwise require. Tracked as a fast-follow, not shipped blocking this PR. |
| W60 | `Vpc` | Flow logs deferred for the same reason — out of scope for the network topology this deliverable asks for. |

`FAIL`s are not accepted under any rationale — two were found and fixed
during authoring (see audit notes below).

## Drift detection

```bash
aws cloudformation detect-stack-drift --stack-name multistate-network-dev
# poll:
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <id-from-above>
# once StackDriftStatus is IN_SYNC or DRIFTED:
aws cloudformation describe-stack-resource-drifts \
  --stack-name multistate-network-dev
```

Drill: a console edit to `ApplicationSecurityGroup` (e.g. adding an ad-hoc
ingress rule to unblock a debugging session) shows up as
`StackResourceDriftStatus: MODIFIED` with the added rule listed under
`PropertyDifferences`. The fix is never "update the template to match
the drift" — it's `execute-change-set` on the existing template, which
reverts the console edit and closes the gap it opened.

## Audit: cfn-author Skill output vs. the cohort checklist

The `cfn-author` Claude Skill was not available in this authoring
environment, so these four templates were hand-authored directly against
the same checklist the Skill's output is normally audited against. Recording
the checklist and the concrete failure modes it guards against here, since
that's the artefact Task 4 actually asks for regardless of which path
produced the first draft:

- **IRSA trust policy: `StringEquals`, never `StringLike`.**
  `multistate-app-dev.yaml`'s `AppIrsaRole` trust condition matches the
  exact `system:serviceaccount:<namespace>:<name>` subject with
  `StringEquals`. A cohort-common mistake is `StringLike` with a
  `system:serviceaccount:<namespace>:*` pattern "to save a redeploy when
  the SA name changes" — that also lets any other service account in the
  same namespace assume the role.
- **DB credentials: Secrets Manager dynamic reference, never
  `NoEcho: true` Parameter.** Both `MasterUsername` and
  `MasterUserPassword` on `DbInstance` resolve via
  `{{resolve:secretsmanager:multistate/dev/db-master:SecretString:...}}`.
  A `NoEcho` Parameter still places the value in the change-set payload
  and stack parameter history; the dynamic reference never puts the
  secret value in the template, the change-set, or CloudTrail.
- **Bucket deletion: both `DeletionPolicy: Retain` AND
  `UpdateReplacePolicy: Retain`.** Both `ArtifactBucket`
  (`multistate-artifacts-dev.yaml`) and `BootstrapBucket`
  (`multistate-bootstrap-dev.yaml`) set both policies. `DeletionPolicy`
  alone only protects against a stack delete; a property change that
  forces replacement (e.g. `BucketName`) is a separate code path that
  `UpdateReplacePolicy` is the one guarding.
- **IAM `Action`/`Resource` scoping.** Every policy statement across all
  four templates enumerates explicit actions (no `*` service-wide
  wildcards) and scopes `Resource` to a specific ARN or ARN prefix — e.g.
  `CfnDeployRole`'s `iam:PassRole` is scoped to `role/multistate-*` AND
  gated with `iam:PassedToService: cloudformation.amazonaws.com`, and
  `AppIrsaRole`'s secret read is scoped to the literal
  `multistate/<env>/db-master-??????` ARN, not `secret:*`.

Two real `cfn-nag` `FAIL`s were caught and fixed during local scanning
(see [Accepted cfn-nag WARNs](#accepted-cfn-nag-warns) above for what
was *not* fixed and why): `DbInstance` originally had a defaulted
`DbMasterUsername` Parameter (F24) and `DeletionProtection: false` (F80).
Both are exactly the class of finding this audit step exists to catch —
neither would have shown up from reading the template casually, only from
running the scanner.

## Known limitation of this pass

This authoring environment has no AWS credentials
(`aws sts get-caller-identity` returns `NoCredentials`), so the
`create-change-set` → `execute-change-set` flow, the live drift drill, and
`aws cloudformation validate-template` were not run against a real
account here. `cfn-lint` and `cfn-nag` were run locally against all four
templates (0 lint errors, 0 nag `FAIL`s) and are documented above with
their output; the commands in this doc are the exact ones to run once
this PR lands and the deploy job has real credentials.
