# Alternative Secret & Authentication Approaches for Databricks CI/CD

## Purpose

We have already implemented two working POCs:

1.  **GitHub Repository Secrets**
2.  **Vault + AWS IAM Authentication**

This document evaluates **five additional approaches** that could be
considered for the same Databricks CI/CD problem.

The source design explicitly considers repository secrets, organization
secrets, Vault per team, and a longer-term move to OIDC-based Databricks
authentication. It also notes that organization secrets become more
attractive as repository count and credential-rotation frequency
increase. fileciteturn4file5L1246-L1278

The five alternatives covered here are:

1.  GitHub Organization Secrets
2.  Vault AppRole
3.  AWS Secrets Manager + EC2 IAM Role
4.  GitHub Actions OIDC → Databricks Workload Identity Federation
5.  AWS IAM → Databricks Workload Identity Federation

------------------------------------------------------------------------

# 1. GitHub Organization Secrets

## Concept

Instead of storing Databricks credentials separately in every
repository, store them as **organization-level GitHub secrets** and
restrict which repositories can use them.

``` text
GitHub Organization
       |
       | Organization Secret
       v
GitHub Actions
       |
       v
Self-hosted Runner
       |
       v
Databricks CLI
       |
       v
Databricks
```

Example:

``` text
DATABRICKS_CLIENT_ID
DATABRICKS_CLIENT_SECRET
```

are stored at organization level.

The organization can then allow only selected repositories to access the
secrets.

------------------------------------------------------------------------

## Pros

-   Very easy migration from repository secrets.
-   Centralizes the credential at the GitHub organization level.
-   Reduces duplication across repositories.
-   Simple GitHub Actions implementation.
-   No additional secret-management platform.
-   Works with existing GitHub Actions workflows.
-   Good fit when several repositories belong to the same team.

## Cons

-   The Databricks client secret still exists in GitHub.
-   Rotation still involves a long-lived credential.
-   GitHub remains the credential-management boundary.
-   Access must be carefully allowlisted.
-   Cross-team isolation becomes more difficult if many teams share the
    same GitHub organization.
-   Does not provide workload identity.

## Best fit

``` text
3–10 repositories
+
one team
+
simple centralized GitHub secret management
```

The source design specifically identifies organization secrets as an
option when repository count or rotation requirements grow beyond the
simplest repository-secret model. fileciteturn4file5L1263-L1271

## Verdict

**Good incremental improvement over repository secrets.**

It is probably the easiest alternative to deploy if the organization is
not ready for Vault.

------------------------------------------------------------------------

# 2. Vault AppRole

## Concept

This is the Vault authentication model we briefly implemented before
moving to AWS IAM authentication.

GitHub Actions would authenticate to Vault using:

``` text
VAULT_ROLE_ID
VAULT_SECRET_ID
```

Flow:

``` text
GitHub Actions
       |
       | Role ID + Secret ID
       v
Vault AppRole
       |
       v
Vault Token
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

Our earlier POC demonstrated that AppRole can issue a token with the
required `databricks-ci` policy.

The problem is that the GitHub workflow needs a way to obtain the
AppRole credentials.

------------------------------------------------------------------------

## Pros

-   Native Vault authentication mechanism.
-   Clear separation between authentication and authorization.
-   Vault policies can enforce least privilege.
-   Centralized secret management.
-   Works across different CI/CD platforms.
-   Does not require AWS specifically.
-   Useful when the runner environment is not tied to AWS.

## Cons

-   `VAULT_ROLE_ID` and `VAULT_SECRET_ID` must themselves be protected.
-   If stored in GitHub Secrets, we have simply introduced another
    secret to protect.
-   More complex than GitHub Secrets.
-   Requires Vault infrastructure.
-   Requires token lifecycle management.
-   Less attractive when the runner already has a strong AWS workload
    identity.

## Best fit

``` text
Multiple CI/CD platforms
+
existing enterprise Vault
+
no reliable workload identity available
```

## Verdict

**Good Vault authentication method, but weaker than AWS IAM for our
AWS-hosted runner.**

Our AWS IAM POC specifically removed the need to store `VAULT_ROLE_ID`
and `VAULT_SECRET_ID`. fileciteturn4file1L287-L330

------------------------------------------------------------------------

# 3. AWS Secrets Manager + EC2 IAM Role

## Concept

Instead of running Vault, store the Databricks credentials in **AWS
Secrets Manager**.

The EC2 runner already has an IAM role, so the workflow can use that
role to read the secret.

``` text
GitHub Actions
       |
       v
AWS EC2 Runner
       |
       | IAM Role
       v
AWS Secrets Manager
       |
       | Databricks credentials
       v
Databricks CLI
       |
       v
Databricks
```

The IAM policy could restrict the runner to a specific secret:

``` text
secretsmanager:GetSecretValue
        |
        +-- specific Databricks secret ARN
```

------------------------------------------------------------------------

## Pros

-   No Vault infrastructure.
-   Uses the EC2 IAM workload identity.
-   No AWS access keys need to be stored.
-   Centralized secret storage.
-   IAM provides fine-grained access control.
-   AWS Secrets Manager supports secret rotation mechanisms.
-   Operationally simpler than running Vault if the workload is already
    AWS-centric.
-   Good fit for AWS-native CI/CD infrastructure.

## Cons

-   Ties the solution strongly to AWS.
-   GitHub Actions running outside AWS would need another access
    pattern.
-   Does not provide Vault's broader multi-platform secret-management
    model.
-   AWS IAM policy management becomes important.
-   Databricks credentials still exist as long-lived secrets.
-   Teams using other clouds may need another implementation.

## Best fit

``` text
AWS-hosted runners
+
AWS-centric platform
+
organization does not already use Vault
```

## Verdict

**Very strong alternative for an AWS-native platform.**

For a team already operating EC2 runners and AWS IAM, this may be
operationally simpler than Vault.

------------------------------------------------------------------------

# 4. GitHub Actions OIDC → Databricks Workload Identity Federation

## Concept

This is the most interesting alternative because it can eliminate the
**Databricks client secret entirely**.

GitHub Actions can issue a short-lived OIDC token.

Databricks can be configured with a federation policy that trusts GitHub
Actions and maps a specific GitHub workload to a Databricks service
principal.

``` text
GitHub Actions
       |
       | Short-lived OIDC token
       v
Databricks Federation Policy
       |
       | maps GitHub workload
       v
Databricks Service Principal
       |
       v
Databricks CLI
       |
       v
Databricks Workspace
```

Databricks currently recommends OAuth token federation for automated
workloads because it eliminates the need to manage Databricks secrets.
citeturn0search3turn0search4

For GitHub Actions, the federation policy can restrict authentication
using claims such as:

``` text
Issuer:
https://token.actions.githubusercontent.com

Audience:
https://github.com/<github-org>

Subject:
repo:<github-org>/<repo>:environment:prod
```

Databricks documents GitHub Actions workload identity federation
specifically for this model. citeturn0search2turn0search0

------------------------------------------------------------------------

## Pros

-   No Databricks client secret.
-   No Vault required.
-   No Databricks secret stored in GitHub.
-   Uses short-lived workload identity tokens.
-   Strong workload-to-service-principal mapping.
-   Excellent GitHub Actions integration.
-   Reduces secret rotation burden.
-   Very strong security model.
-   Simple runtime architecture after initial configuration.

## Cons

-   Requires Databricks account-level configuration.
-   Requires federation policy administration.
-   GitHub repository/environment claims must be designed carefully.
-   Existing enterprise identity/security teams may need to approve the
    model.
-   Requires modern Databricks federation support.
-   Does not solve secret management for unrelated external systems.

## Best fit

``` text
GitHub Actions
+
Databricks
+
organization wants to eliminate Databricks client secrets
```

## Verdict

**Potentially better than both GitHub Secrets and Vault for Databricks
authentication specifically.**

The major advantage is architectural:

``` text
GitHub Secrets
    -> stores credential

Vault
    -> stores credential

OIDC federation
    -> stores no Databricks credential
```

Databricks explicitly states that workload identity federation
eliminates the need to manage Databricks secrets for automated
workloads. citeturn0search3

------------------------------------------------------------------------

# 5. AWS IAM → Databricks Workload Identity Federation

## Concept

This takes the workload-identity idea one step further for an AWS-hosted
runner.

The EC2 already has:

``` text
github-vault-poc-runner-role
```

Instead of using that IAM role to authenticate to Vault, the AWS
workload identity can potentially be federated directly to Databricks.

Databricks documents AWS IAM outbound identity federation as a supported
federation-policy scenario, where the workload subject can be an AWS IAM
role ARN. citeturn0search2

Conceptually:

``` text
GitHub Actions
       |
       v
AWS EC2 Runner
       |
       | IAM Role
       v
AWS Workload Identity
       |
       | Federation
       v
Databricks
       |
       v
Databricks Service Principal
       |
       v
Databricks Bundle
```

------------------------------------------------------------------------

## Pros

-   No Databricks client secret.
-   No Vault required.
-   No GitHub secret required for Databricks authentication.
-   Uses the existing EC2 IAM workload identity.
-   Short-lived identity/token model.
-   Very strong workload identity architecture.
-   Fewer runtime components than Vault.
-   Excellent fit for AWS-hosted runners.

## Cons

-   Couples the CI/CD runner identity to AWS.
-   Requires careful federation-policy configuration.
-   More complex initial setup than GitHub Secrets.
-   Less portable if the organization later moves runners to another
    cloud.
-   Requires Databricks federation support and appropriate account-level
    configuration.

## Best fit

``` text
AWS self-hosted runners
+
Databricks
+
strong workload identity requirement
```

## Verdict

**One of the strongest architectures for an AWS-hosted Databricks CI/CD
platform.**

It potentially gives us the security benefits we liked about Vault + AWS
IAM while removing Vault from the runtime path.

------------------------------------------------------------------------

# 6. Side-by-Side Comparison

  -------------------------------------------------------------------------------------------
  Approach     Stores       Extra      Workload   Rotation      Complexity   Best Fit
               Databricks   Platform   Identity                              
               Secret?                                                       
  ------------ ------------ ---------- ---------- ------------- ------------ ----------------
  GitHub Repo  Yes          None       No         Manual        Low          Small POC
  Secrets                                                                    

  GitHub Org   Yes          None       No         Centralized   Low          Several repos
  Secrets                                         in GitHub                  

  Vault        Yes          Vault      No         Centralized   High         Multi-platform
  AppRole                                                                    Vault

  Vault + AWS  Yes          Vault      Yes        Centralized   High         Enterprise AWS +
  IAM                                                                        Vault

  AWS Secrets  Yes          AWS        Yes        Centralized   Medium       AWS-native
  Manager                   Secrets                                          platform
                            Manager                                          

  GitHub OIDC  **No**       None       **Yes**    Not           Medium       GitHub →
  → Databricks                                    applicable                 Databricks

  AWS IAM →    **No**       None       **Yes**    Not           Medium       AWS runner →
  Databricks                                      applicable                 Databricks
  -------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 7. Security Ranking

For the specific problem of authenticating GitHub Actions to Databricks:

### Strongest candidates

``` text
1. GitHub OIDC → Databricks WIF
2. AWS IAM → Databricks WIF
3. Vault + AWS IAM
4. AWS Secrets Manager + IAM
5. GitHub Organization Secrets
6. GitHub Repository Secrets
7. Vault AppRole with GitHub-stored credentials
```

This ranking is specifically about the combination of credential
exposure, workload identity, and operational model. It is not a
universal ranking for every enterprise use case.

Databricks currently strongly recommends workload identity federation
for automated workloads over OAuth client secrets or personal access
tokens. citeturn0search4turn0search3

------------------------------------------------------------------------

# 8. What Changes Depending on the Requirement?

## Requirement: "I just need a POC"

Use:

``` text
GitHub Repository Secrets
```

Fastest and easiest.

------------------------------------------------------------------------

## Requirement: "I have 5--10 repositories"

Consider:

``` text
GitHub Organization Secrets
```

The source design explicitly identifies organization secrets as a
reasonable option as repository count and rotation requirements
increase. fileciteturn4file5L1263-L1271

------------------------------------------------------------------------

## Requirement: "The company already runs Vault"

Consider:

``` text
Vault + AWS IAM
```

This provides a consistent enterprise secret-management model.

------------------------------------------------------------------------

## Requirement: "The runners are AWS-based and we don't have Vault"

Consider:

``` text
AWS Secrets Manager + IAM
```

This avoids introducing another platform.

------------------------------------------------------------------------

## Requirement: "We only need GitHub → Databricks authentication"

Strongly consider:

``` text
GitHub OIDC → Databricks Workload Identity Federation
```

This eliminates the Databricks client secret entirely.
citeturn0search0turn0search2

------------------------------------------------------------------------

## Requirement: "We use AWS runners and want zero Databricks secrets"

Consider:

``` text
AWS IAM → Databricks Workload Identity Federation
```

This uses the existing AWS workload identity directly.

Databricks documents AWS IAM outbound identity federation as a supported
federation-policy scenario. citeturn0search2

------------------------------------------------------------------------

# 9. Recommended Architecture Tree

A practical decision tree would be:

``` text
Do we need Databricks authentication?
              |
              v
       Can we use OIDC?
          /               YES       NO
         |         |
         v         v
GitHub OIDC     Do we have
→ Databricks    enterprise Vault?
WIF              /                     YES      NO
                |        |
                v        v
             Vault     Are runners
             + IAM     AWS-based?
                         /                           YES     NO
                        |       |
                        v       v
                 AWS Secrets   GitHub
                 Manager       Secrets
```

For AWS-hosted runners, also evaluate:

``` text
AWS IAM → Databricks Workload Identity Federation
```

before introducing Vault.

------------------------------------------------------------------------

# 10. Important Observation

The biggest architectural evolution is:

### Generation 1 --- Store credentials

``` text
GitHub Secrets
      |
      v
Databricks Client Secret
```

### Generation 2 --- Centralize credentials

``` text
Vault / AWS Secrets Manager
      |
      v
Databricks Client Secret
```

### Generation 3 --- Eliminate credentials

``` text
Workload Identity
      |
      v
Databricks Federation
```

The third model is the most interesting long-term direction for
Databricks CI/CD because there is no long-lived Databricks client secret
to rotate.

The source design itself identifies OIDC-based Databricks authentication
as a possible long-term direction specifically because it can remove the
need to store the client secret in GitHub or Vault.
fileciteturn4file5L1246-L1257

------------------------------------------------------------------------

# 11. Final Recommendation

For our POC journey, I would evaluate the approaches in this order:

``` text
1. GitHub Secrets
   ↓
2. Vault + AWS IAM
   ↓
3. GitHub OIDC → Databricks WIF
   ↓
4. AWS IAM → Databricks WIF
```

We have already completed #1 and #2.

The **next POC I would personally do is GitHub OIDC → Databricks
Workload Identity Federation**, because it lets us compare:

``` text
GitHub Secrets
       vs
Vault + AWS IAM
       vs
GitHub OIDC → Databricks
```

and answer a much more interesting question:

> **Do we actually need a secret-management system at all if the only
> secret we are protecting is the Databricks service-principal
> credential?**

That is where workload identity federation becomes particularly
compelling. Databricks currently recommends it for automated workloads
and explicitly supports GitHub Actions. citeturn0search3turn0search4
