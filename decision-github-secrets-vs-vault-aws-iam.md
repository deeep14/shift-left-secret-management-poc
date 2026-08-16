# Decision Document --- GitHub Secrets vs Vault with AWS IAM

## Executive Summary

### Decision

**Recommended long-term approach: Vault + AWS IAM authentication.**

For an enterprise CI/CD platform, Vault + AWS IAM provides the stronger
architecture because it centralizes secret management, uses the AWS
workload identity of the self-hosted runner, supports least-privilege
policies, and avoids storing Vault authentication credentials in GitHub.

However, **GitHub Secrets is the better choice for small/simple
workloads** where introducing and operating Vault would add unnecessary
complexity.

### At a glance

  Area                            GitHub Secrets       Vault + AWS IAM
  ------------------------------- -------------------- ---------------------
  Initial setup                   Excellent            Moderate/Complex
  Simplicity                      Excellent            Moderate
  Centralized secret management   No                   Yes
  Secret rotation at scale        Moderate             Strong
  Workload identity               No                   Yes
  Least privilege                 Good                 Excellent
  Governance                      Good                 Strong
  Operational overhead            Low                  High
  Multi-repository scalability    Moderate             Excellent
  Best fit                        Small/simple CI/CD   Enterprise platform

------------------------------------------------------------------------

## 1. What We Tested

Both approaches were implemented against the same Databricks CI/CD
scenario.

### Approach 1 --- GitHub Secrets

Databricks service-principal credentials were stored as:

``` text
DATABRICKS_CLIENT_ID
DATABRICKS_CLIENT_SECRET
```

in GitHub repository secrets.

Flow:

``` text
GitHub
  |
  | GitHub Secrets
  v
GitHub Actions
  |
  v
AWS EC2 Self-hosted Runner
  |
  v
Databricks CLI
  |
  v
Databricks Workspace
```

We successfully verified:

-   PR validation
-   Push-to-main validation
-   Push-to-main deployment
-   Service-principal authentication
-   Bundle validation
-   Bundle deployment

### Approach 2 --- Vault + AWS IAM

The Databricks credentials were moved into Vault KV:

``` text
secret/databricks
```

The EC2 runner was assigned:

``` text
github-vault-poc-runner-role
```

Vault was configured with:

``` text
AWS IAM Role
github-vault-poc-runner-role
        |
        v
Vault Role
databricks-ci-aws
        |
        v
Vault Policy
databricks-ci
        |
        v
secret/data/databricks
```

We successfully verified:

``` bash
vault login -method=aws role=databricks-ci-aws
```

without using:

-   `VAULT_ROLE_ID`
-   `VAULT_SECRET_ID`
-   AWS access keys

The resulting token successfully retrieved the Databricks credentials
from Vault.

------------------------------------------------------------------------

# 2. Security

## GitHub Secrets

### Pros

-   Secrets are not committed to Git.
-   GitHub provides encrypted secret storage.
-   Secrets can be scoped to repositories/environments.
-   Very simple to implement.

### Cons

-   The Databricks client secret still exists in GitHub.
-   Secret ownership can become distributed across repositories.
-   Rotation can require updating multiple GitHub secrets.
-   GitHub remains part of the long-term credential-management boundary.

## Vault + AWS IAM

### Pros

-   Databricks credentials are centrally stored in Vault.
-   GitHub does not need Vault Role IDs/Secret IDs.
-   The EC2 runner authenticates using its AWS workload identity.
-   Vault policies can restrict access to specific secret paths.
-   Rotation can be managed centrally.
-   The runner's AWS IAM role provides a machine identity.

### Cons

-   Vault becomes another security-critical platform.
-   Vault must be properly operated and secured.
-   Vault availability becomes part of the deployment path.
-   Incorrect policies can either block deployments or grant excessive
    access.

### Winner: Vault + AWS IAM

------------------------------------------------------------------------

# 3. Secret Rotation

With GitHub Secrets, many repositories can end up with copies of the
same credential:

``` text
Repo 1 -> credential
Repo 2 -> credential
Repo 3 -> credential
...
Repo N -> credential
```

Rotation can therefore become distributed.

With Vault:

``` text
Vault
  |
  +-- secret/databricks
```

Consumers retrieve the current value from the central store.

### Winner: Vault + AWS IAM

------------------------------------------------------------------------

# 4. Scalability

GitHub Secrets is excellent when there are only a few repositories and
credentials.

As repositories, teams, environments, and secrets grow, centralized
management becomes more valuable.

Vault allows a model such as:

``` text
Vault
 |
 +-- team-a/
 |    +-- databricks
 |
 +-- team-b/
 |    +-- databricks
 |
 +-- team-c/
      +-- databricks
```

with policies controlling which workload can read which secret.

### Winner: Vault + AWS IAM

------------------------------------------------------------------------

# 5. Operational Complexity

This is where GitHub Secrets clearly wins.

### GitHub Secrets

``` text
GitHub
  |
  v
GitHub Secrets
  |
  v
Databricks
```

### Vault + AWS IAM

``` text
GitHub
  |
  v
Self-hosted Runner
  |
  v
AWS IAM
  |
  v
Vault AWS Auth
  |
  v
Vault Role
  |
  v
Vault Policy
  |
  v
Vault KV
  |
  v
Databricks
```

The Vault solution has substantially more components to configure,
monitor, troubleshoot, and secure.

### Winner: GitHub Secrets

------------------------------------------------------------------------

# 6. Availability and Dependencies

GitHub Secrets introduces relatively few components into the credential
path.

Vault adds another critical dependency:

``` text
GitHub
  |
  v
Runner
  |
  v
Vault
  |
  v
Databricks
```

If production Vault is unavailable, the workflow cannot retrieve the
Databricks credentials.

A production implementation therefore needs highly available Vault and
appropriate operational controls.

### Winner: GitHub Secrets

------------------------------------------------------------------------

# 7. Identity Model

This is one of the biggest architectural differences.

### GitHub Secrets

The workflow receives a credential:

``` text
DATABRICKS_CLIENT_SECRET
```

and uses it to authenticate to Databricks.

### Vault + AWS IAM

The runner has an AWS workload identity:

``` text
github-vault-poc-runner-role
```

Vault authenticates that identity using AWS IAM authentication:

``` text
AWS IAM Role
      |
      v
Vault AWS Auth
      |
      v
Vault Role
      |
      v
Vault Token
```

The POC proved that the EC2 could authenticate to Vault without storing
Vault credentials.

### Winner: Vault + AWS IAM

------------------------------------------------------------------------

# 8. Least Privilege

The GitHub approach can be scoped using GitHub repository/environment
controls, which is good.

Vault provides an explicit policy boundary.

Our POC policy was:

``` hcl
path "secret/data/databricks" {
  capabilities = ["read"]
}
```

The resulting token could read the Databricks secret while unrelated
secret paths were denied.

### Winner: Vault + AWS IAM

------------------------------------------------------------------------

# 9. Governance and Auditability

GitHub provides useful repository and workflow controls.

Vault adds a dedicated secret-management layer with centralized:

-   Secret policies
-   Authentication
-   Secret access
-   Secret lifecycle
-   Team/workload permissions
-   Audit capabilities

This becomes increasingly valuable when multiple teams and repositories
consume secrets.

### Winner: Vault + AWS IAM

------------------------------------------------------------------------

# 10. Developer Experience

GitHub Secrets is extremely simple:

``` yaml
env:
  DATABRICKS_CLIENT_ID: ${{ secrets.DATABRICKS_CLIENT_ID }}
  DATABRICKS_CLIENT_SECRET: ${{ secrets.DATABRICKS_CLIENT_SECRET }}
```

Vault requires developers/platform engineers to understand:

-   AWS IAM roles
-   Vault AWS auth
-   Vault roles
-   Vault policies
-   Vault secret paths
-   Runner identity

### Winner: GitHub Secrets

------------------------------------------------------------------------

# 11. POC Effort

Our own POCs demonstrated this difference.

### GitHub Secrets

``` text
Create SP
   |
Add GitHub secrets
   |
Create workflow
   |
Deploy
```

### Vault + AWS IAM

``` text
Create EC2
   |
Create IAM role
   |
Install Docker
   |
Run Vault
   |
Store secret
   |
Create Vault policy
   |
Enable AWS auth
   |
Map IAM role to Vault role
   |
Configure GitHub runner
   |
Authenticate from workflow
   |
Retrieve secret
   |
Deploy
```

### Winner: GitHub Secrets

------------------------------------------------------------------------

# 12. Production Consideration

Our Vault POC used development mode:

``` text
Storage: inmem
```

and:

``` text
VAULT_DEV_ROOT_TOKEN_ID
```

This was appropriate for proving the authentication model, but it is
**not a production deployment model**.

A production Vault implementation would need, at minimum:

-   Persistent storage
-   TLS
-   Highly available deployment
-   Secure bootstrap
-   Production policies
-   Audit logging
-   Monitoring
-   Backup/recovery
-   Controlled network access
-   Appropriate token TTLs
-   Environment separation

Therefore, the recommendation is about the **Vault + AWS IAM
architecture**, not about deploying the development-mode Vault container
to production.

------------------------------------------------------------------------

# 13. Decision Matrix

Score:

-   1 = Poor
-   3 = Acceptable
-   5 = Strong

  Criterion                          GitHub Secrets   Vault + AWS IAM
  -------------------------------- ---------------- -----------------
  Initial simplicity                              5                 2
  Developer experience                            5                 3
  Operational overhead                            5                 2
  Centralized secret management                   2                 5
  Secret rotation                                 3                 5
  Multi-repository scalability                    3                 5
  Workload identity                               2                 5
  Least privilege                                 4                 5
  Governance                                      3                 5
  Auditability                                    3                 5
  Infrastructure independence                     5                 2
  Enterprise extensibility                        3                 5
  POC suitability                                 5                 3
  Long-term platform suitability                  3                 5

------------------------------------------------------------------------

# 14. Recommendation

## Use GitHub Secrets when

-   The number of repositories is small.
-   The number of credentials is small.
-   The team wants the simplest possible CI/CD implementation.
-   Rotation requirements are straightforward.
-   There is no existing enterprise Vault platform.
-   The additional operational cost of Vault is not justified.

## Use Vault + AWS IAM when

-   Many repositories or teams need secrets.
-   Centralized secret management is required.
-   Secret rotation is important.
-   Least-privilege access needs to be centrally enforced.
-   AWS self-hosted runners are already part of the platform.
-   Workload identity is preferred over distributing additional
    credentials.
-   The organization already operates production-grade Vault.
-   The platform is expected to grow.

------------------------------------------------------------------------

# 15. Final Decision

## 🏆 Preferred strategic architecture: Vault + AWS IAM

The important reason is not simply:

> "Vault is more secure than GitHub Secrets."

The stronger architectural distinction is:

``` text
GitHub Secrets
    =
credential distribution
```

versus:

``` text
Vault + AWS IAM
    =
centralized secret management
+
workload identity
+
policy-based access
+
centralized rotation
```

GitHub Secrets remains an excellent solution for simple, low-scale
CI/CD.

Vault + AWS IAM is the stronger **enterprise platform pattern** when
centralized governance, workload identity, least privilege, rotation,
and multi-repository scalability are requirements.

------------------------------------------------------------------------

# 16. Recommended Adoption Path

A practical migration path is:

``` text
Phase 1
GitHub Secrets
    |
    v
Establish CI/CD baseline
```

``` text
Phase 2
Vault
    |
    v
Move secret storage to Vault
```

``` text
Phase 3
AWS IAM authentication
    |
    v
Remove Vault credentials from GitHub
```

``` text
Phase 4
Centralize policies + rotation
    |
    v
Scale across repositories and teams
```

This allows teams to prove CI/CD first and introduce the additional
Vault infrastructure when the organizational requirements justify it.

------------------------------------------------------------------------

# 17. Design Review Recommendation

> **Use GitHub Secrets for simple, low-scale CI/CD workloads; adopt
> Vault with AWS IAM authentication as the strategic enterprise pattern
> when centralized secret management, workload identity, least
> privilege, rotation, and multi-repository scalability become
> requirements.**

## Final verdict

**For our POCs: both work.**

**For a quick implementation: GitHub Secrets wins.**

**For the long-term enterprise architecture: Vault + AWS IAM wins.**

The trade-off is straightforward:

> **GitHub Secrets optimizes for simplicity. Vault + AWS IAM optimizes
> for scale, governance, and identity-driven security.**
