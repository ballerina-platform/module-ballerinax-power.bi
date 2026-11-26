# Ballerina Power BI connector

[![Build](https://github.com/ballerina-platform/module-ballerinax-power.bi/actions/workflows/ci.yml/badge.svg)](https://github.com/ballerina-platform/module-ballerinax-power.bi/actions/workflows/ci.yml)
[![GitHub Last Commit](https://img.shields.io/github/last-commit/ballerina-platform/module-ballerinax-power.bi.svg)](https://github.com/ballerina-platform/module-ballerinax-power.bi/commits/main)
[![GitHub Issues](https://img.shields.io/github/issues/ballerina-platform/ballerina-library/module/power.bi.svg?label=Open%20Issues)](https://github.com/ballerina-platform/ballerina-library/labels/module%2Fpower.bi)

## Overview

[Power BI](https://www.microsoft.com/en-us/power-platform/products/power-bi) is Microsoft's business analytics service for building interactive reports and dashboards on top of shared datasets.

The Ballerina Power BI connector supports version 1.0 of the [Power BI REST API](https://learn.microsoft.com/en-us/rest/api/power-bi/). It lets Ballerina applications manage workspaces, datasets, reports, dashboards, dataflows, gateways, capacities, deployment pipelines and scorecards, push data into push datasets, run DAX queries, trigger and monitor refreshes, generate embed tokens, and use the tenant-wide admin operations.

## Setup guide

To use the Power BI connector, you need a Power BI account in a Microsoft 365 organization and an application registered in Microsoft Entra ID. The connector sends a Microsoft Entra ID access token for the Power BI service with each request. The token can be issued to a signed-in user (delegated access) or to the application itself (service principal access).

> **Note:** The screenshots in this guide are for illustration only. The Microsoft Entra admin center changes over time, so treat them as a visual reference rather than an exact match — follow the described actions and choose the values (application name, redirect URI, permissions, and so on) that fit your own scenario.

### Step 1: Register an application in Microsoft Entra ID

1. Sign in to the [Microsoft Entra admin center](https://entra.microsoft.com).
2. Navigate to **Entra ID** > **App registrations** and click **New registration**.

   ![App registrations](https://raw.githubusercontent.com/ballerina-platform/module-ballerinax-microsoft.excel/refs/heads/master/docs/resources/setup-guide-1.jpg)

3. Enter a name of your choice and select the account types appropriate for your organization. If you will call the API as a user, add a **Redirect URI** under the **Web** platform — this is where the sign-in flow returns the authorization code (a Microsoft-hosted page such as `https://jwt.ms` is a convenient choice for Step 4). The **Web** platform is required because the code is redeemed with a client secret; the public "Mobile and desktop applications" platform does not accept one. A service principal needs no redirect URI. Click **Register**.

   ![Register an application](https://raw.githubusercontent.com/ballerina-platform/module-ballerinax-microsoft.excel/refs/heads/master/docs/resources/setup-guide-2.jpeg)

4. On the app's **Overview** page, note the **Application (client) ID** and the **Directory (tenant) ID** — you need both to request a token in Step 4.

   ![Overview of application](https://raw.githubusercontent.com/ballerina-platform/module-ballerinax-microsoft.excel/refs/heads/master/docs/resources/setup-guide-3.jpeg)

### Step 2: Grant Power BI access

How the application gets access depends on whether the connector calls the API as a user or as a service principal. Follow the option that fits your integration.

#### Option A: As a user (delegated permissions)

1. In the registered application, go to **API permissions** > **Add a permission** and select **Power BI Service** from the **Microsoft APIs** tab.
2. Choose **Delegated permissions** and add the scopes your integration needs, for example:

   | Permission | Purpose |
   |---|---|
   | `Workspace.Read.All` / `Workspace.ReadWrite.All` | Read or manage workspaces |
   | `Dataset.Read.All` / `Dataset.ReadWrite.All` | Read datasets, or push data, run queries and trigger refreshes |
   | `Report.Read.All` / `Report.ReadWrite.All` | Read or manage reports |
   | `Dashboard.Read.All` / `Dashboard.ReadWrite.All` | Read or manage dashboards and tiles |
   | `Tenant.Read.All` / `Tenant.ReadWrite.All` | Call the admin operations (the signed-in user must be a Fabric administrator) |

   Request only the scopes your integration needs, then click **Add permissions**.

3. Back on the **API permissions** page, click **Grant admin consent for \<your tenant\>** and confirm **Yes**, if your organization requires admin consent. Each permission should then show **Granted for \<your tenant\>** in the **Status** column.

#### Option B: As a service principal

Don't add Power BI permissions to the app registration — a service principal's access comes from Power BI tenant settings and workspace roles, the permissions are never used, and the admin operations require that the application has no Power BI permissions that need admin consent.

1. Create a Microsoft Entra security group and add the application to it.
2. A Fabric administrator must enable **Service principals can call Fabric public APIs** (formerly **Allow service principals to use Power BI APIs**) under **Developer settings** in the admin portal's tenant settings, and apply it to that security group. For the admin operations, also enable **Service principals can access read-only admin APIs** under **Admin API settings** for the same group.
3. Add the application to each workspace it uses, as a **Member** or **Admin**. A service principal has no **My workspace**, so it can only call the operations that take a workspace (group) ID.

### Step 3: Create a client secret

1. Go to **Certificates & secrets** > **New client secret**, add a description and an expiry, and click **Add**.

   ![Add a client secret](https://raw.githubusercontent.com/ballerina-platform/module-ballerinax-microsoft.excel/refs/heads/master/docs/resources/setup-guide-8.jpeg)

2. Copy the secret **Value** immediately — it is shown only once. (The **Secret ID** next to it is just a label, not a credential.)

   ![Certificates & secrets](https://raw.githubusercontent.com/ballerina-platform/module-ballerinax-microsoft.excel/refs/heads/master/docs/resources/setup-guide-7.jpeg)

### Step 4: Obtain an access token

Replace `<TENANT_ID>`, `<CLIENT_ID>`, and `<CLIENT_SECRET>` below with the values from Steps 1 and 3. Use your directory (tenant) ID for a single-tenant application, or `organizations` for a multi-tenant one. Both options request the token from the same endpoint, `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token`.

#### Option A: As a user (authorization code flow)

**1. Get an authorization code.** This is an interactive sign-in and consent step, so it cannot be curled — paste this URL into a browser instead. The scope below requests `Dataset.ReadWrite.All`; replace or extend it with the scopes you added in Step 2, each prefixed with `https://analysis.windows.net/powerbi/api/` and separated by `%20`.

```text
https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/authorize?client_id=<CLIENT_ID>&response_type=code&redirect_uri=https%3A%2F%2Fjwt.ms&response_mode=query&scope=https%3A%2F%2Fanalysis.windows.net%2Fpowerbi%2Fapi%2FDataset.ReadWrite.All&state=12345
```

Sign in and accept the consent prompt. You land on `https://jwt.ms/?code=<AUTH_CODE>&state=12345` — copy the `code` value from that URL. It is single-use and short-lived, so use it within a few minutes.

**2. Exchange the code for an access token.**

macOS / Linux:

```bash
curl -X POST "https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token" \
  --data-urlencode "client_id=<CLIENT_ID>" \
  --data-urlencode "client_secret=<CLIENT_SECRET>" \
  --data-urlencode "scope=https://analysis.windows.net/powerbi/api/Dataset.ReadWrite.All" \
  --data-urlencode "code=<AUTH_CODE>" \
  --data-urlencode "redirect_uri=https://jwt.ms" \
  --data-urlencode "grant_type=authorization_code"
```

Windows (PowerShell — call `curl.exe` explicitly; plain `curl` is aliased to `Invoke-WebRequest` and takes different flags):

```powershell
curl.exe -X POST "https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token" `
  --data-urlencode "client_id=<CLIENT_ID>" `
  --data-urlencode "client_secret=<CLIENT_SECRET>" `
  --data-urlencode "scope=https://analysis.windows.net/powerbi/api/Dataset.ReadWrite.All" `
  --data-urlencode "code=<AUTH_CODE>" `
  --data-urlencode "redirect_uri=https://jwt.ms" `
  --data-urlencode "grant_type=authorization_code"
```

#### Option B: As a service principal (client credentials flow)

No sign-in is involved; the application requests the token with its own credentials. The scope is always `https://analysis.windows.net/powerbi/api/.default`.

macOS / Linux:

```bash
curl -X POST "https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token" \
  --data-urlencode "client_id=<CLIENT_ID>" \
  --data-urlencode "client_secret=<CLIENT_SECRET>" \
  --data-urlencode "scope=https://analysis.windows.net/powerbi/api/.default" \
  --data-urlencode "grant_type=client_credentials"
```

Windows (PowerShell):

```powershell
curl.exe -X POST "https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token" `
  --data-urlencode "client_id=<CLIENT_ID>" `
  --data-urlencode "client_secret=<CLIENT_SECRET>" `
  --data-urlencode "scope=https://analysis.windows.net/powerbi/api/.default" `
  --data-urlencode "grant_type=client_credentials"
```

In either case, the JSON response includes an `access_token` field — that is the `token` value for `Config.toml`.

> The access token is short-lived (about an hour). Request a new one when it expires — repeat the client credentials request, or for a user, add `offline_access` to the scopes and redeem the returned `refresh_token` at the same endpoint with `grant_type=refresh_token`.

## Quickstart

To use the Power BI connector in your Ballerina application, update the `.bal` file as follows:

### Step 1: Import the module

Import the `power.bi` module.

```ballerina
import ballerinax/power.bi;
```

### Step 2: Instantiate a new connector

1. Create a `Config.toml` file and configure the access token obtained in the steps above:

```toml
token = "<Access Token>"
```

2. Create a `bi:ConnectionConfig` with the access token and initialize the connector with it.

```ballerina
configurable string token = ?;

final bi:Client powerbi = check new ({
    auth: {
        token
    }
});
```

### Step 3: Invoke the connector operation

Now, utilize the available connector operations.

#### List the datasets in My workspace

```ballerina
public function main() returns error? {
    bi:Datasets _ = check powerbi->getDatasets();
}
```

### Step 4: Run the Ballerina application

```bash
bal run
```

## Examples

The `Power BI` connector provides practical examples illustrating usage in various scenarios. Explore these [examples](https://github.com/ballerina-platform/module-ballerinax-power.bi/tree/main/examples/), covering the following use cases:

1. [Push dataset sales feed](https://github.com/ballerina-platform/module-ballerinax-power.bi/tree/main/examples/push_dataset_sales_feed) - Create a push dataset in a workspace if it is missing, push sales rows into it and read per-region totals back with a DAX query.

2. [Dataset refresh monitoring](https://github.com/ballerina-platform/module-ballerinax-power.bi/tree/main/examples/dataset_refresh_monitoring) - Review the refresh schedule and recent refresh history of every refreshable dataset in a workspace, and optionally re-run refreshes that failed.

## Build from the source

### Setting up the prerequisites

1. Download and install Java SE Development Kit (JDK) version 21. You can download it from either of the following sources:

    * [Oracle JDK](https://www.oracle.com/java/technologies/downloads/)
    * [OpenJDK](https://adoptium.net/)

   > **Note:** After installation, remember to set the `JAVA_HOME` environment variable to the directory where JDK was installed.

2. Download and install [Ballerina Swan Lake](https://ballerina.io/).

3. Download and install [Docker](https://www.docker.com/get-started).

   > **Note**: Ensure that the Docker daemon is running before executing any tests.

4. Export Github Personal access token with read package permissions as follows,

    ```bash
    export packageUser=<Username>
    export packagePAT=<Personal access token>
    ```

### Build options

Execute the commands below to build from the source.

1. To build the package:

   ```bash
   ./gradlew clean build
   ```

2. To run the tests:

   ```bash
   ./gradlew clean test
   ```

3. To build the without the tests:

   ```bash
   ./gradlew clean build -x test
   ```

4. To run tests against different environments:

   ```bash
   ./gradlew clean test -Pgroups=<Comma separated groups/test cases>
   ```

5. To debug the package with a remote debugger:

   ```bash
   ./gradlew clean build -Pdebug=<port>
   ```

6. To debug with the Ballerina language:

   ```bash
   ./gradlew clean build -PbalJavaDebug=<port>
   ```

7. Publish the generated artifacts to the local Ballerina Central repository:

    ```bash
    ./gradlew clean build -PpublishToLocalCentral=true
    ```

8. Publish the generated artifacts to the Ballerina Central repository:

   ```bash
   ./gradlew clean build -PpublishToCentral=true
   ```

## Contribute to Ballerina

As an open-source project, Ballerina welcomes contributions from the community.

For more information, go to the [contribution guidelines](https://github.com/ballerina-platform/ballerina-lang/blob/master/CONTRIBUTING.md).

## Code of conduct

All the contributors are encouraged to read the [Ballerina Code of Conduct](https://ballerina.io/code-of-conduct).

## Useful links

* For more information go to the [`power.bi` package](https://central.ballerina.io/ballerinax/power.bi/latest).
* For example demonstrations of the usage, go to [Ballerina By Examples](https://ballerina.io/learn/by-example/).
* Chat live with us via our [Discord server](https://discord.gg/ballerinalang).
* Post all technical questions on Stack Overflow with the [#ballerina](https://stackoverflow.com/questions/tagged/ballerina) tag.
