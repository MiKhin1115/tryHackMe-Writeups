# CryptoCabana

## Category

Cloud

## Points

90

## Difficulty

Medium

---

## Overview

CryptoCabana is an Azure cloud security challenge from TryHackMe Hacker Holidays. The target is a static web kiosk that claims to safely back up crypto seed phrases. Although the website looks simple, it exposes cloud access details before the user even interacts with the application.

The challenge focuses on Azure Storage, exposed trust relationships, leaked credentials, and Azure Key Vault secret versioning.

---

## Challenge Description

The CryptoCabana kiosk promises users that their wallet recovery information is safely backed up. However, the challenge briefing suggests that something was able to access the stored data without the owner’s permission.

Important clues from the briefing include:

```text
Pull apart what the kiosk hands out for free before you've even clicked anything.
Follow that trust somewhere the kiosk's own page never once points you.
Somewhere in there is a second, more valuable set of keys.
A vault that won't give up the real values on the first ask.
```

The tags for the challenge are:

```text
Cloud
Azure
Storage
Key Vault
```

These clues point toward a cloud misconfiguration involving Azure Storage and Azure Key Vault.

---

## Target

```text
https://cryptocabanaf5scjagc.z13.web.core.windows.net/
```

---

## Objective

The objective is to inspect the public kiosk application, identify what cloud access it exposes, follow that access to additional resources, and recover the hidden secret from Azure Key Vault.

---

## Initial Enumeration

The target is hosted on an Azure Static Website endpoint:

```text
web.core.windows.net
```

This usually means the site is being served from an Azure Storage Account configured for static website hosting.

Because the application is static, all frontend files are visible to the user. Therefore, the first step was to inspect the website using browser Developer Tools.

Useful areas to check:

- Page source
- JavaScript files
- Network requests
- Configuration files
- Storage URLs
- SAS tokens
- Azure account names
- Key Vault references

Common filenames worth checking include:

```text
index.html
main.js
app.js
config.json
settings.json
env.js
```

---

## Frontend Analysis

After opening the site, I inspected the JavaScript and network requests. Since frontend code cannot safely hide secrets, I searched for Azure-related strings.

Useful search terms included:

```text
blob.core.windows.net
vault.azure.net
sig=
sv=
sas
tenant
client
secret
keyvault
```

The frontend exposed information that allowed access to Azure Storage. This was the first major finding.

---

## Key Finding 1: Exposed Azure Storage Access

The kiosk exposed cloud storage access to the browser. This could be in the form of a storage URL, SAS token, or configuration value.

A SAS token is a Shared Access Signature. It can grant temporary access to Azure Storage resources. SAS tokens are not always dangerous by themselves, but they become dangerous when they are exposed publicly and grant too much access.

In this challenge, the exposed trust allowed access beyond what the public website needed.

---

## Azure Storage Enumeration

After identifying the storage access details, I used them to enumerate available blobs.

Example command:

```bash
az storage blob list \
  --account-name <STORAGE_ACCOUNT_NAME> \
  --container-name <CONTAINER_NAME> \
  --sas-token "<SAS_TOKEN>" \
  --output table
```

If a full blob URL was available, files could also be accessed using `curl`:

```bash
curl "https://<storage_account>.blob.core.windows.net/<container>/<blob>?<SAS_TOKEN>"
```

The goal was not only to view the files used by the website, but also to check whether the same access could reach files not linked from the page.

This matched the challenge clue:

```text
Follow that trust somewhere the kiosk's own page never once points you.
```

---

## Key Finding 2: Hidden Storage Data

The exposed storage access allowed reading more than the kiosk’s public frontend files. By listing blobs and checking additional paths, I found hidden data that was not referenced directly by the website.

This hidden data contained a second, more valuable set of keys.

The discovered information included Azure authentication details such as:

```text
tenantId
clientId
clientSecret
subscriptionId
keyVaultName
```

These values allowed authentication as an Azure service principal.

---

## Azure Authentication

Using the discovered credentials, I logged in with the Azure CLI.

```bash
az login --service-principal \
  -u "<CLIENT_ID>" \
  -p "<CLIENT_SECRET>" \
  --tenant "<TENANT_ID>"
```

After logging in, I confirmed the active account context:

```bash
az account show
```

Then I listed accessible Azure resources:

```bash
az resource list --output table
```

Since the challenge tags included Key Vault, I focused on finding available vaults.

```bash
az keyvault list --output table
```

---

## Key Finding 3: Azure Key Vault Access

After identifying the Key Vault, I listed the available secrets.

```bash
az keyvault secret list \
  --vault-name <VAULT_NAME> \
  --output table
```

The challenge hint said:

```text
if a value looks freshly rotated, ask yourself what it looked like five minutes before that
```

This points to Azure Key Vault secret versions.

Azure Key Vault can keep older versions of secrets. If a secret was rotated, the latest value may not contain the desired data, but an older version may still be accessible.

---

## Secret Version Enumeration

Instead of only reading the latest secret value, I listed all versions of the secret.

```bash
az keyvault secret list-versions \
  --vault-name <VAULT_NAME> \
  --name <SECRET_NAME> \
  --output table
```

Then I checked older versions one by one:

```bash
az keyvault secret show \
  --vault-name <VAULT_NAME> \
  --name <SECRET_NAME> \
  --version <VERSION_ID>
```

The latest secret value did not reveal the real value. An older version contained the required secret.

---

## Flag

```text
THM{n0t_ur_k3ys_n0t_ur_c01ns!}
```

---

## Vulnerability Explanation

This challenge involved several cloud security weaknesses.

### 1. Public Frontend Exposure

The static website exposed cloud access details to the browser. Anything placed in frontend code should be treated as public.

### 2. Over-Permissive Azure Storage Access

The exposed storage access allowed reading files beyond what the public application required. This allowed access to hidden blobs.

### 3. Credential Leakage

A second set of Azure credentials was stored in a location reachable through the exposed storage permissions.

### 4. Key Vault Secret Version Exposure

The Key Vault secret had older versions still available. Even though the latest version appeared rotated, previous versions still contained sensitive information.

---

## Root Cause

The root cause was poor cloud access control design.

The application trusted client-side cloud access too much. Public users were given access to storage resources that contained sensitive internal data. That storage then exposed stronger Azure credentials, which allowed access to Key Vault.

The Key Vault configuration also allowed older secret versions to remain readable.

---

## Impact

An attacker could:

- Inspect the frontend application.
- Reuse exposed Azure Storage access.
- Read hidden blobs.
- Recover service principal credentials.
- Authenticate to Azure.
- Enumerate Key Vault secrets.
- Read older versions of secrets.
- Recover sensitive wallet backup data.

---

## Lessons Learned

- Static frontend code cannot protect secrets.
- Public applications should not expose privileged cloud access.
- SAS tokens should be short-lived and limited in scope.
- Azure Storage containers should separate public and private data.
- Service principal credentials should never be stored in publicly reachable locations.
- Secret rotation is not enough if old secret versions remain accessible.
- Key Vault access should follow least privilege.

---

## Mitigation

To prevent this issue:

1. Do not place secrets or privileged tokens in frontend code.
2. Use backend APIs to mediate access to private cloud resources.
3. Use short-lived SAS tokens with minimal permissions.
4. Disable blob listing unless required.
5. Separate public website storage from private application storage.
6. Store credentials securely in Key Vault.
7. Restrict service principal permissions using least privilege.
8. Limit access to Key Vault secret versions.
9. Remove or purge old secret versions when they are no longer needed.
10. Monitor Azure Storage and Key Vault access logs.

---

## Conclusion

CryptoCabana demonstrated how exposed frontend cloud access can lead to deeper compromise. By inspecting the static website, it was possible to find Azure Storage access, discover hidden credentials, authenticate to Azure, and retrieve a sensitive value from an older Key Vault secret version.

The main lesson is that cloud trust should never be given directly to an unauthenticated browser unless it is tightly scoped and carefully monitored.
