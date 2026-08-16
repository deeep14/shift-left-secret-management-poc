# Approach 2 --- Vault + AWS IAM Authentication

## Purpose

This POC validates a Vault-based secret-management model where GitHub
Actions runs on an AWS EC2 self-hosted runner and uses the EC2's IAM
role to authenticate to Vault.

The goal is to remove the need to store Vault Role IDs / Secret IDs or
AWS access keys in GitHub.

The Databricks service-principal credentials remain in Vault and are
retrieved only during deployment.

The reference design describes the target model as a Mariner runner IAM
role authenticating to Vault AWS auth, mapping to a team Vault role, and
receiving only the policy required to read that team's Databricks
secret. fileciteturn3file1L77-L95

------------------------------------------------------------------------

# 1. Target Architecture

``` text
GitHub Actions
       |
       | runs on
       v
AWS EC2 Self-hosted Runner
       |
       | IAM Role
       v
github-vault-poc-runner-role
       |
       | AWS IAM authentication
       v
Vault AWS Auth
       |
       | mapped role
       v
databricks-ci-aws
       |
       | policy
       v
databricks-ci
       |
       | read
       v
secret/data/databricks
       |
       +-- DATABRICKS_CLIENT_ID
       |
       +-- DATABRICKS_CLIENT_SECRET
       |
       v
Databricks CLI
       |
       v
Databricks Workspace
```

The reference architecture similarly defines the runner IAM role -\>
Vault team role -\> least-privilege secret policy flow.
fileciteturn3file1L77-L95

------------------------------------------------------------------------

# 2. Why AWS IAM Authentication?

The earlier approach used

``` text
DATABRICKS_CLIENT_ID
DATABRICKS_SECRET_ID
```

Those values would themselves have to be stored somewhere such as GitHub
Secrets.

AWS IAM authentication removes that requirement.

The EC2 already has an AWS identity:

``` text
github-vault-poc-runner-role
```

Vault can authenticate that AWS identity using an AWS-signed IAM
request.

Conceptually:

``` text
EC2 IAM Role
      |
      | signed AWS authentication request
      v
Vault AWS Auth
      |
      | validates AWS identity
      v
Vault Role
      |
      v
Vault Token
```

The key idea is:

> AppRole proves identity using Vault credentials; AWS IAM auth proves
> identity using the AWS workload identity.

------------------------------------------------------------------------

# 3. AWS EC2 Setup

An EC2 instance was created for the POC.

The instance uses the IAM role:

``` text
github-vault-poc-runner-role
```

AWS account:

``` text
<ACCOUNT_ID>
```

IAM role ARN:

``` text
arn:aws:iam::<ACCOUNT_ID>:role/github-vault-poc-runner-role
```

The EC2 identity was verified with:

``` bash
aws sts get-caller-identity
```

Result:

``` text
arn:aws:sts::<ACCOUNT_ID>:assumed-role/github-vault-poc-runner-role/<instance-id>
```

This confirmed that the EC2 instance was successfully assuming the IAM
role.

------------------------------------------------------------------------

# 4. EC2 IAM Permissions

The role initially had no broad AWS permissions.

When creating the Vault AWS role mapping, Vault attempted to resolve the
IAM role ARN and AWS returned:

``` text
not authorized to perform: iam:GetRole
```

Therefore, the role was given the narrowly scoped permission:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iam:GetRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/github-vault-poc-runner-role"
    }
  ]
}
```

This allows Vault's AWS auth configuration to resolve the specified IAM
role without giving the runner broad IAM administration permissions.

------------------------------------------------------------------------

# 5. Docker Vault on EC2

Docker was installed on Amazon Linux 2023:

``` bash
sudo yum install -y docker
```

The Docker daemon was started:

``` bash
sudo systemctl start docker
```

and enabled on boot:

``` bash
sudo systemctl enable docker
```

Vault was then started as a development-mode container:

``` bash
docker run -d \
  --cap-add=IPC_LOCK \
  --name dev-vault \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=dev-only-token \
  hashicorp/vault
```

Vault was accessed locally using:

``` text
http://127.0.0.1:8200
```

> This is a development POC only. Vault dev mode uses in-memory storage
> and a development root token and must not be used as a production
> deployment.

------------------------------------------------------------------------

# 6. Vault CLI

The Vault CLI was installed on Amazon Linux 2023 using the HashiCorp RPM
repository.

After installation:

``` bash
vault version
```

The CLI was configured to communicate with the local Vault server:

``` bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='dev-only-token'
```

------------------------------------------------------------------------

# 7. Databricks Secret in Vault

The Databricks service-principal credentials were stored in Vault KV v2.

Secret path:

``` text
secret/databricks
```

The secret contains:

``` text
DATABRICKS_CLIENT_ID
DATABRICKS_CLIENT_SECRET
```

The actual values must never be committed to Git or documented in
plaintext.

The secret was verified with:

``` bash
vault kv get secret/databricks
```

------------------------------------------------------------------------

# 8. Least-Privilege Vault Policy

A Vault policy named:

``` text
databricks-ci
```

was created.

Policy:

``` hcl
path "secret/data/databricks" {
  capabilities = ["read"]
}
```

This gives the authenticated workload only the ability to read the
Databricks secret.

It does not grant:

``` text
create
update
delete
list
patch
sudo
```

The resulting authorization model is:

``` text
databricks-ci
     |
     +-- read secret/data/databricks
```

The reference design also calls for least-privilege Vault policies
scoped only to the team's Databricks secrets.
fileciteturn3file1L90-L95

------------------------------------------------------------------------

# 9. Enable AWS Authentication

Vault's AWS authentication method was enabled:

``` bash
vault auth enable aws
```

This creates:

``` text
auth/aws/
```

The AWS auth method is responsible for validating AWS IAM-based
authentication requests.

------------------------------------------------------------------------

# 10. AWS IAM -\> Vault Role Mapping

A Vault role was created:

``` text
databricks-ci-aws
```

The role maps the AWS IAM role:

``` text
arn:aws:iam::<ACCOUNT_ID>:role/github-vault-poc-runner-role
```

to the Vault policy:

``` text
databricks-ci
```

Command:

``` bash
vault write auth/aws/role/databricks-ci-aws \
  auth_type="iam" \
  bound_iam_principal_arn="arn:aws:iam::<ACCOUNT_ID>:role/github-vault-poc-runner-role" \
  token_policies="databricks-ci"
```

Verify:

``` bash
vault read auth/aws/role/databricks-ci-aws
```

Conceptually:

``` text
AWS IAM Role
github-vault-poc-runner-role
          |
          | mapped to
          v
Vault Role
databricks-ci-aws
          |
          | attached policy
          v
Vault Policy
databricks-ci
          |
          | read
          v
secret/data/databricks
```

This corresponds to the reference model where the runner IAM role is
mapped to a team-specific Vault role. fileciteturn3file1L77-L95

------------------------------------------------------------------------

# 11. AWS IAM Authentication Test

The EC2's AWS identity was already available through the instance role.

The AWS CLI confirmed:

``` bash
aws sts get-caller-identity
```

The instance was using:

``` text
github-vault-poc-runner-role
```

The EC2 also uses IMDSv2 for temporary credentials.

The important point is that these AWS credentials are temporary and are
supplied by the EC2 instance role rather than being stored as static AWS
access keys.

------------------------------------------------------------------------

# 12. Vault Login Using AWS IAM

The critical test was:

``` bash
vault login -method=aws role=databricks-ci-aws
```

This succeeded.

Vault returned a token with:

``` text
token_meta_account_id    <ACCOUNT_ID>
token_meta_auth_type     iam
token_policies           ["databricks-ci" "default"]
```

This proves that Vault authenticated the EC2 using AWS IAM and mapped
the AWS identity to the Vault role.

------------------------------------------------------------------------

# 13. Secret Retrieval Test

After successful AWS IAM authentication:

``` bash
vault kv get secret/databricks
```

successfully returned:

``` text
DATABRICKS_CLIENT_ID
DATABRICKS_CLIENT_SECRET
```

Therefore the complete chain was successfully proven:

``` text
EC2
 |
 | AWS IAM identity
 v
AWS
 |
 | IAM authentication
 v
Vault AWS Auth
 |
 | AWS role mapping
 v
databricks-ci-aws
 |
 | policy
 v
databricks-ci
 |
 | read
 v
secret/databricks
```

------------------------------------------------------------------------

# 14. GitHub Secrets vs AWS IAM

The earlier AppRole experiment used:

``` text
DATABRICKS_CLIENT_ID
DATABRICKS_SECRET_ID
```

The flow was:

``` text
GitHub Actions
      |
      | Client ID + Secret ID
      v
Databricks secret
```

The AWS IAM approach is:

``` text
GitHub Actions
      |
      v
AWS EC2 Runner
      |
      | IAM role
      v
Vault AWS Auth
      |
      v
Vault role
      |
      v
Vault token
      |
      v
Databricks secret
```

The AWS IAM approach avoids storing Vault AppRole credentials in GitHub.

------------------------------------------------------------------------

# 15. Intended GitHub Actions Integration

The next step is to run GitHub Actions on the same AWS EC2 as a
self-hosted runner.

The workflow will conceptually do:

``` bash
vault login -method=aws role=databricks-ci-aws
```

then retrieve:

``` bash
vault kv get -field=DATABRICKS_CLIENT_ID secret/databricks
vault kv get -field=DATABRICKS_CLIENT_SECRET secret/databricks
```

and export the values as:

``` text
DATABRICKS_CLIENT_ID
DATABRICKS_CLIENT_SECRET
```

The Databricks CLI can then execute:

``` bash
databricks bundle validate -t dev
```

and:

``` bash
databricks bundle deploy -t dev
```

The environment-variable contract remains compatible with the Databricks
CLI. The reference design likewise describes resolving Vault secrets
into `DATABRICKS_*` environment variables before bundle deployment.
fileciteturn3file8L387-L394

------------------------------------------------------------------------

# 16. Intended Final CI/CD Flow

``` text
GitHub
   |
   | Pull Request / Push
   v
GitHub Actions
   |
   | Self-hosted runner
   v
AWS EC2
   |
   | IAM role
   v
Vault AWS Auth
   |
   | mapped AWS role
   v
Vault Role: databricks-ci-aws
   |
   | policy: databricks-ci
   v
Vault KV
   |
   | Databricks credentials
   v
DATABRICKS_* environment variables
   |
   v
Databricks CLI
   |
   v
Bundle Validate / Deploy
```

------------------------------------------------------------------------

# 17. Security Considerations

Do not log:

``` text
DATABRICKS_CLIENT_SECRET
```

Do not commit:

``` text
DATABRICKS_CLIENT_SECRET
VAULT_TOKEN
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
VAULT_ROLE_ID
VAULT_SECRET_ID
```

The reference design explicitly recommends not logging client secrets or
full Vault payloads and recommends masking secret values when exporting
them into GitHub Actions. fileciteturn3file4L203-L214

Vault policies should follow least privilege.

The runner IAM role should not be allowed to access other teams'
secrets. fileciteturn3file4L205-L214

Production should also retain GitHub Environment approval controls for
production deployments; moving credential storage into Vault does not
replace deployment approval gates. fileciteturn3file4L205-L214

------------------------------------------------------------------------

# 18. Final Result

The POC demonstrates two credential-management models:

``` text
Approach 1
GitHub Secrets
      |
      v
Databricks CLI
```

and:

``` text
Approach 2
AWS IAM
   |
   v
Vault AWS Auth
   |
   v
Vault Policy
   |
   v
Databricks Secret
   |
   v
Databricks CLI
```

Approach 2 removes the need for credentials in GitHub and
uses the AWS runner's workload identity as the authentication mechanism.
