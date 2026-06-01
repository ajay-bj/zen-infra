# How Pods Access AWS Resources — IAM Roles for Service Accounts (IRSA)

A hands-on guide for the team. By the end of this you will understand **why**
organizations use IAM Roles + Service Accounts instead of access keys, **what**
OIDC is and who issues it, and you will have a pod that can list S3 buckets
without any AWS credentials baked into it.

> **This guide uses YOUR own cluster.** Everyone forks this repo and deploys into
> their own AWS account, so there are no hardcoded account IDs or cluster names
> below. Wherever you see a value in `<ANGLE_BRACKETS>`, replace it with your own.
> Every step is shown through the **AWS Console (UI)** first, with the CLI
> equivalent given as a reference.

---

## Part 1 — The Problem

Your application runs as a pod inside EKS. It needs to read a file from S3, or
read a secret from Secrets Manager. How does the pod prove to AWS that it is
allowed to do that?

There are a few ways people try this. Most of them are bad.

### Bad option 1 — Hardcode AWS access keys

Put `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` into the pod (env vars, a
config file, a Kubernetes Secret).

Problems:
- **Long-lived credentials.** They never expire on their own. If they leak, an
  attacker has access until someone manually rotates them.
- **They leak easily.** They end up in Git, in container images, in logs, in
  `kubectl describe`.
- **Hard to rotate.** Rotating means rebuilding and redeploying every workload
  that uses them.
- **No per-app isolation.** Everyone tends to share one powerful key.

### Bad option 2 — Give the EKS node's IAM role broad permissions

Every node already has an IAM role (the node group role). You could attach
`AmazonS3FullAccess` to it.

Problems:
- **Every pod on that node inherits the permission.** A compromised pod that
  was only supposed to read one bucket can now touch everything in S3.
- **No isolation between apps.** Two different services share the exact same AWS
  permissions just because they landed on the same node.
- **Violates least privilege.** You cannot say "only THIS app may read THIS bucket".

### The goal

What we actually want:
- Each app gets **only the permissions it needs** (least privilege).
- **No long-lived secrets** stored anywhere.
- Credentials are **short-lived** and **rotated automatically**.
- Permissions are **tied to the identity of the app**, not to the node.

This is exactly what **IRSA (IAM Roles for Service Accounts)** gives us.

---

## Part 2 — What is OIDC and Who Issues It

### OIDC in one sentence

**OIDC (OpenID Connect)** is a standard way for one system to prove identity to
another system using short-lived, cryptographically signed tokens (JWTs) instead
of passwords or static keys.

### Who issues it here

When you create an EKS cluster, AWS automatically runs an **OIDC identity
provider** for that cluster. It exposes a public URL called the **issuer**. It
looks like this (yours will have a different ID at the end):

```
https://oidc.eks.<REGION>.amazonaws.com/id/<YOUR_OIDC_ID>
```

That endpoint publishes public keys. Anyone (including AWS IAM) can use those
public keys to verify "yes, this token really was issued by this specific EKS
cluster, and it has not been tampered with."

### The three parties

1. **EKS OIDC provider** — issues signed tokens to pods. (The "who are you")
2. **IAM** — has been told to *trust* that specific OIDC provider, and maps
   identities to IAM roles. (The "what are you allowed to do")
3. **The pod** — receives a token automatically and exchanges it with AWS STS
   for temporary AWS credentials.

> Important: creating an EKS cluster gives you the OIDC issuer, but you still
> have to **register that issuer in IAM as an OIDC provider** before IAM will
> trust its tokens. In this repo, Terraform (`modules/eks`) already creates that
> registration for you. In Part 3 you will confirm it exists in the Console.

### How the flow works (step by step)

```
┌──────────────┐   1. Pod starts with a ServiceAccount that is
│     Pod      │      annotated with an IAM role ARN.
│ (your app)   │
└──────┬───────┘
       │ 2. EKS injects a short-lived OIDC token (a signed JWT)
       │    into the pod at /var/run/secrets/eks.amazonaws.com/...
       ▼
┌──────────────┐   3. AWS SDK in the pod calls STS:
│  AWS STS     │      "AssumeRoleWithWebIdentity" + the OIDC token.
└──────┬───────┘
       │ 4. STS checks the token signature against the EKS OIDC
       │    provider's public keys, and checks the IAM role's
       │    trust policy (does this role trust this provider +
       │    this exact ServiceAccount?).
       ▼
┌──────────────┐   5. If everything matches, STS returns
│ Temporary    │      TEMPORARY credentials (valid ~1 hour,
│ Credentials  │      auto-rotated). No static keys anywhere.
└──────────────┘
```

The key insight: the pod never holds a permanent secret. It holds a token that
proves "I am the s3-reader pod in namespace X", and AWS hands back temporary
credentials based on that proven identity.

---

## Part 3 — Hands-On Lab (AWS Console first)

We will deliberately do this the slow, manual way so you see each piece and what
problem it solves. In real life Terraform creates all of this for you — that is
what the `modules/iam` code in this repo does — but doing it by hand once makes
the automation make sense.

### Values you will collect along the way

Keep a scratch note. You will fill these in from YOUR account as you go:

| Placeholder | Where you get it | Example shape |
|---|---|---|
| `<ACCOUNT_ID>` | top-right of AWS Console, or Step 0 | `123456789012` |
| `<REGION>` | the region you deployed into | `us-east-1` |
| `<CLUSTER_NAME>` | your EKS cluster name | `pharma-prod-cluster` |
| `<OIDC_ID>` | Step 3 (the long ID at the end of the issuer URL) | `A1B2C3D4...` |
| `<OIDC_PROVIDER>` | Step 3 (issuer without `https://`) | `oidc.eks.us-east-1.amazonaws.com/id/A1B2C3D4...` |

### Prerequisites

Connect `kubectl` to YOUR cluster (replace the name and region):
```bash
aws eks update-kubeconfig --name <CLUSTER_NAME> --region <REGION>
kubectl get nodes
```

---

### Step 0 — Find your Account ID (Console)

Top-right of the AWS Console, click your account name. The 12-digit number is
your `<ACCOUNT_ID>`. Write it down.

---

### Step 1 — Deploy a pod WITHOUT any AWS permissions

Create a namespace:
```bash
kubectl create namespace irsa-demo
```

Create `irsa-demo-deploy.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: irsa-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aws-cli-demo
  namespace: irsa-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aws-cli-demo
  template:
    metadata:
      labels:
        app: aws-cli-demo
    spec:
      serviceAccountName: s3-reader-sa
      containers:
        - name: aws-cli
          image: amazon/aws-cli:latest
          command: ["sleep", "3600"]   # keep container alive so we can exec in
```

Apply it:
```bash
kubectl apply -f irsa-demo-deploy.yaml
kubectl get pods -n irsa-demo
```

---

### Step 2 — Exec into the pod and try to list S3 (this WILL fail)

```bash
kubectl exec -it -n irsa-demo deploy/aws-cli-demo -- /bin/bash
```

Inside the pod:
```bash
aws sts get-caller-identity
aws s3 ls
```

**Expected result:** an error like `Unable to locate credentials` or
`AccessDenied`.

**Why:** the pod has an identity inside Kubernetes (the ServiceAccount), but that
identity means **nothing** to AWS yet. There is no IAM role, no trust, no token
mapping. This is the "Part 1 problem" in action — the pod cannot prove anything
to AWS.

Type `exit` to leave the pod.

---

### Step 3 — Find YOUR cluster's OIDC provider (Console)

**In the AWS Console:**
1. Go to **EKS → Clusters → `<CLUSTER_NAME>`**.
2. Open the **Overview** tab (sometimes labelled **Details/Configuration**).
3. Find the field **OpenID Connect provider URL**. It looks like:
   ```
   https://oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>
   ```
4. Copy it. The part after `https://` is your `<OIDC_PROVIDER>`, and the long
   string at the very end is your `<OIDC_ID>`. Write both down.

**Now confirm IAM trusts it:**
1. Go to **IAM → Identity providers** (left menu, under *Access management*).
2. You should see an entry named
   `oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>`.
3. Click it. Note the **ARN** at the top — this is your `<OIDC_PROVIDER_ARN>`:
   ```
   arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>
   ```

> **If the identity provider is NOT listed**, your cluster's OIDC issuer is not
> registered in IAM yet. In this repo Terraform creates it, but if you ever need
> to do it by hand:
> ```bash
> eksctl utils associate-iam-oidc-provider --cluster <CLUSTER_NAME> --region <REGION> --approve
> ```

**CLI reference (to verify the same thing):**
```bash
aws eks describe-cluster --name <CLUSTER_NAME> --region <REGION> --query "cluster.identity.oidc.issuer" --output text
aws iam list-open-id-connect-providers
```

---

### Step 4 — Create the IAM Role with a trust policy (Console, click-by-click)

The **trust policy** answers: "Which identity is allowed to assume this role?"
We will restrict it to exactly our OIDC provider AND our specific ServiceAccount
(`irsa-demo:s3-reader-sa`).

This is the step most people get stuck on, so we go screen by screen. Take your
time and match each field exactly.

#### 4.1 — Open the Create Role wizard

1. In the AWS Console search bar, type **IAM** and open the **IAM** service.
2. In the left menu, click **Roles** (under *Access management*).
3. Click the orange **Create role** button (top right).

You are now on the **Step 1: Select trusted entity** screen.

#### 4.2 — Step 1 screen: Select trusted entity

You will see a row of boxes under **Trusted entity type**:
`AWS service` | `AWS account` | `Web identity` | `SAML 2.0 federation` | `Custom trust policy`.

> ⚠️ The first box, **AWS service**, is selected by default. That is the WRONG
> one for us — that is for EC2/Lambda. Do **not** leave it on that.

1. Click the **Web identity** box (third one). It will highlight blue.
2. A new section titled **Web identity** appears below with two dropdowns:
   - **Identity provider:** click the dropdown and choose your cluster's OIDC
     provider:
     ```
     oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>
     ```
     (This is the value you found in Step 3. If the dropdown is empty, the OIDC
     provider is not registered in IAM yet — go back to Step 3.)
   - **Audience:** click the dropdown and choose **`sts.amazonaws.com`**.
     (It is usually the only option.)
3. Leave everything else as-is and click **Next** (bottom right).

#### 4.3 — Step 2 screen: Add permissions

This is where you choose WHAT the role is allowed to do.

1. In the **Permissions policies** search box, type: `AmazonS3ReadOnlyAccess`.
2. In the results, tick the **checkbox** to the left of
   **AmazonS3ReadOnlyAccess**.
3. Click **Next** (bottom right).

> For a real app you would use a custom, tightly-scoped policy (e.g. read only
> one specific bucket). `AmazonS3ReadOnlyAccess` is fine for this learning demo.

#### 4.4 — Step 3 screen: Name, review, and create

1. **Role name:** enter exactly:
   ```
   irsa-demo-s3-reader-role
   ```
2. (Optional) **Description:** "Demo role for IRSA hands-on — S3 read only".
3. Scroll down to the **Step 1: Select trusted entities** review box. It shows
   the trust policy the wizard generated for you. Right now it only checks the
   audience (`aud`). We will tighten it in the next sub-step.
4. Click **Create role** (bottom right).

You will be returned to the Roles list with a green success banner.

#### 4.5 — Lock the trust policy to your exact ServiceAccount (important)

The wizard's trust policy lets **any** ServiceAccount in the cluster assume this
role. We want ONLY `irsa-demo:s3-reader-sa`. We add a `sub` condition.

1. In the Roles list, click your new role **`irsa-demo-s3-reader-role`**.
2. Click the **Trust relationships** tab.
3. Click **Edit trust policy**.
4. Replace the entire JSON with the following. **Replace `<ACCOUNT_ID>`,
   `<REGION>`, and `<OIDC_ID>` with your own values** (the same ones from Step 3):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com",
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:irsa-demo:s3-reader-sa"
        }
      }
    }
  ]
}
```

5. Click **Update policy**.

> 💡 Watch the punctuation: the keys are the OIDC provider string **without**
> `https://`, followed by `:aud` and `:sub`. A common mistake is leaving
> `https://` in, or using the wrong region/ID — the role then silently fails to
> assume and you get `AccessDenied` in Step 7.

**Read the two conditions out loud — this is the whole security model:**
- `:aud = sts.amazonaws.com` → the token must be intended for AWS STS.
- `:sub = system:serviceaccount:irsa-demo:s3-reader-sa` → ONLY the
  `s3-reader-sa` ServiceAccount in the `irsa-demo` namespace may assume this
  role. A pod in any other namespace, or using any other ServiceAccount, is
  rejected. **This is the per-app isolation we wanted.**

#### 4.6 — Copy the role ARN

Still on the role page, at the very top you will see **ARN** with a copy icon.
Copy it — you need it in Step 5:
```
arn:aws:iam::<ACCOUNT_ID>:role/irsa-demo-s3-reader-role
```

> Why a separate role + policy? The **role** is the identity ("the S3 reader").
> The **policy** (AmazonS3ReadOnlyAccess) is the list of allowed actions. Keeping
> them separate means you can reuse policies and audit permissions cleanly. For
> real least privilege you would attach a custom policy scoped to a single
> bucket instead of the managed read-only policy.

**CLI reference (same result without the UI):**
```bash
# trust-policy.json must contain the JSON above with your real values
aws iam create-role --role-name irsa-demo-s3-reader-role --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name irsa-demo-s3-reader-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

---

### Step 5 — Link the ServiceAccount to the IAM role (annotation)

This annotation is the bridge between Kubernetes and IAM. It tells EKS: "when a
pod uses this ServiceAccount, inject a token and target this role."

You can do this **either** with a one-line command **or** by editing the YAML.
Pick ONE of the two options below.

#### Option A — Quick command (no file editing)

```bash
kubectl annotate serviceaccount s3-reader-sa -n irsa-demo \
  eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/irsa-demo-s3-reader-role --overwrite
```

#### Option B — Declarative (edit the YAML and re-apply)

1. Open `irsa-demo-deploy.yaml` and add the `annotations` block to the
   ServiceAccount so it looks like this:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: irsa-demo
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/irsa-demo-s3-reader-role
```

2. Re-apply the file. `kubectl apply` is idempotent — it only updates what
   changed (here, it adds the annotation to the existing ServiceAccount):

```bash
kubectl apply -f irsa-demo-deploy.yaml
```

#### Verify the annotation was set (either option)

```bash
kubectl get serviceaccount s3-reader-sa -n irsa-demo -o yaml
```

You should see your `eks.amazonaws.com/role-arn` under `annotations`. If it is
there, the link is in place.

> Note: changing the ServiceAccount annotation does NOT automatically update
> pods that are already running. The token is only injected when a pod starts.
> That is why Step 6 restarts the deployment.

---

### Step 6 — Restart the pod so EKS injects the token

The token injection happens when the pod starts. Restart so it picks up the
annotation:

```bash
kubectl rollout restart deployment/aws-cli-demo -n irsa-demo
kubectl get pods -n irsa-demo
```

---

### Step 7 — Exec in again and list S3 (this WILL now work)

```bash
kubectl exec -it -n irsa-demo deploy/aws-cli-demo -- /bin/bash
```

Inside the pod:
```bash
aws sts get-caller-identity
```
You should now see the **assumed role** ARN
(`.../irsa-demo-s3-reader-role/...`), NOT a user and NOT an error.

```bash
aws s3 ls
```
This now returns your bucket list. 🎉

**What changed?** Nothing inside the container. No keys were added. Same image,
same command. The only difference is the pod now runs under a ServiceAccount
that AWS trusts via OIDC, so the AWS SDK automatically:
1. Read the injected OIDC token from
   `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`,
2. Called `sts:AssumeRoleWithWebIdentity`,
3. Got temporary credentials,
4. Used them for `aws s3 ls`.

Prove there are no static keys:
```bash
env | grep AWS
```
You will see `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` — **but no access
key or secret key anywhere.**

Type `exit`.

> **Want to SEE it in the Console?** Go to **CloudTrail → Event history** and
> filter by event name `AssumeRoleWithWebIdentity`. You will see the call your
> pod made, including which role it assumed. This is the per-app audit trail you
> get for free with IRSA.

---

## Part 4 — Why Organizations Prefer This

| Concern | Access keys in pod | Node IAM role | **IRSA (this)** |
|---|---|---|---|
| Long-lived secret? | Yes (bad) | No | **No** |
| Per-app least privilege? | No | No | **Yes** |
| Auto-rotated credentials? | No | Yes | **Yes** |
| Secret can leak in Git/logs? | Yes (bad) | N/A | **No** |
| Identity tied to the app? | No | No (tied to node) | **Yes** |
| Auditable in CloudTrail per app? | Hard | No | **Yes** |

In short: IRSA gives every workload its own least-privilege AWS identity, with
short-lived auto-rotated credentials, and nothing secret stored anywhere. That
is why it is the standard pattern in production EKS.

This is exactly what the `modules/iam` Terraform in this repo automates — for
example the External Secrets Operator role (`pharma-<env>-eso-role`) is created
with a trust policy bound to
`system:serviceaccount:external-secrets:external-secrets`, the same mechanism
you just did by hand.

---

## Part 5 — Cleanup

Tear down the demo so it does not linger.

**Kubernetes:**
```bash
kubectl delete namespace irsa-demo
```

**IAM (Console):** go to **IAM → Roles**, search `irsa-demo-s3-reader-role`,
select it, **Delete**. (The attached managed policy detaches automatically when
the role is deleted.)

**IAM (CLI alternative):**
```bash
aws iam detach-role-policy --role-name irsa-demo-s3-reader-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name irsa-demo-s3-reader-role
```

---

## Quick Glossary

- **OIDC (OpenID Connect):** identity standard using signed short-lived tokens.
- **OIDC issuer/provider:** the EKS-run endpoint that issues and signs those
  tokens; must be registered in IAM to be trusted.
- **IRSA:** IAM Roles for Service Accounts — binding a Kubernetes ServiceAccount
  to an IAM role via OIDC.
- **Trust policy:** the part of an IAM role that says *who* may assume it.
- **Permission policy:** the part that says *what* the role may do.
- **STS AssumeRoleWithWebIdentity:** the API call that swaps an OIDC token for
  temporary AWS credentials.
- **`aud` (audience):** who the token is for (`sts.amazonaws.com`).
- **`sub` (subject):** which ServiceAccount the token represents
  (`system:serviceaccount:<namespace>:<sa-name>`).
