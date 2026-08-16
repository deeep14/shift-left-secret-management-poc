# Approach 1 --- GitHub Secrets + Databricks Service Principal

## Purpose

This POC validates a simple Shift-Left CI/CD model where GitHub Actions
retrieves Databricks service-principal credentials directly from GitHub
repository secrets and uses them to validate and deploy a Databricks
Asset Bundle.

The goal is to establish the baseline before moving secret retrieval
into Vault.

------------------------------------------------------------------------

## Architecture

``` text
GitHub Repository
       |
       | GitHub Actions
       v
Self-hosted GitHub Runner
       |
       | DATABRICKS_CLIENT_ID
       | DATABRICKS_CLIENT_SECRET
       v
Databricks CLI
       |
       v
Databricks Workspace
```

For the POC, the self-hosted runner was an EC2 instance.

------------------------------------------------------------------------

## Prerequisites

-   Databricks workspace
-   Databricks service principal
-   Databricks OAuth client ID and client secret
-   Databricks CLI
-   Databricks Asset Bundle
-   GitHub repository
-   GitHub Actions self-hosted runner
-   Repository secrets

------------------------------------------------------------------------

## Databricks Bundle

The bundle was initially created as `databricks_vault_poc`.

Validation:

``` bash
databricks bundle validate -t dev
```

Expected:

``` text
Validation OK!
```

Deployment:

``` bash
databricks bundle deploy -t dev
```

Expected:

``` text
Deployment complete!
```

The bundle was also deployed to `prod` successfully during the initial
bundle POC.

------------------------------------------------------------------------

## Service Principal Authentication

A Databricks service principal was created for the POC.

The credentials were:

``` text
DATABRICKS_CLIENT_ID
DATABRICKS_CLIENT_SECRET
```

The service principal was verified locally with:

``` bash
databricks current-user me
```

The returned user was the service principal.

Bundle validation also confirmed that the CLI was authenticating as the
service principal:

``` text
User: <service-principal-client-id>
```

------------------------------------------------------------------------

## GitHub Repository

POC repository:

``` text
shift-left-secret-management-poc
```

The repository contains the Databricks bundle and GitHub Actions
workflow.

------------------------------------------------------------------------

## GitHub Secrets

The following repository secrets were created:

``` text
DATABRICKS_CLIENT_ID
DATABRICKS_CLIENT_SECRET
```

The actual values must never be committed to the repository.

------------------------------------------------------------------------

## CI/CD Behaviour Verified

### Direct push to `main`

Result:

``` text
Validate -> Deploy
```

The deployment completed successfully and changes were visible in the
Databricks UI.

### Pull request

Result:

``` text
Validate only
```

The deployment did not happen.

### Merge PR to `main`

Result:

``` text
Validate -> Deploy
```

The updated Databricks job configuration was deployed and the change was
visible in the Databricks UI.

------------------------------------------------------------------------

## Self-hosted Runner

A self-hosted runner was configured for the POC.

The workflow uses:

``` yaml
runs-on: deepak-ec2-runner
```

The runner executes the GitHub Actions jobs and provides the machine on
which the Databricks CLI runs.

------------------------------------------------------------------------

## Why Approach 1 Works

The GitHub Actions runner receives the Databricks credentials through
GitHub Secrets.

The Databricks CLI then uses:

``` text
DATABRICKS_HOST
DATABRICKS_CLIENT_ID
DATABRICKS_CLIENT_SECRET
```

to authenticate as the Databricks service principal.

The credentials are not stored in the Git repository, but they are still
managed as GitHub secrets.

------------------------------------------------------------------------

## Advantages

-   Simple to implement
-   Minimal infrastructure
-   Easy to understand
-   Works immediately with GitHub Actions
-   Good baseline for a POC

The source design also identifies repository secrets as the
straightforward option for a single proof of concept or a small number
of repositories. fileciteturn3file4L220-L235

------------------------------------------------------------------------

## Limitations

-   Long-lived Databricks client secrets remain in GitHub
-   Rotation requires updating GitHub secrets
-   Scaling across many repositories can become difficult
-   Secret ownership is distributed across repositories/orgs

The reference design notes that organization-level secrets can be
preferable when the number of repositories or rotation requirements
increases. fileciteturn3file4L220-L235

------------------------------------------------------------------------

## Final Flow

``` text
GitHub Secrets
      |
      | client ID + client secret
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
Databricks Workspace
```

------------------------------------------------------------------------

## Status

**POC completed successfully.**

The following were verified:

-   Databricks Bundle validation
-   Databricks Bundle deployment
-   Service principal authentication
-   GitHub Actions validation on PRs
-   GitHub Actions deployment on merges/pushes to `main`
-   Self-hosted runner execution
