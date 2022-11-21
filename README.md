# Configure Vault PKI with Azure HSM

## Activate Azure Managed HSM  

Follow steps from the following link:
https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/quick-create-cli

## Configure App Registrations
1. Create app registration for `vault-root-ca` and log client ID, secret ID, and SP object
2. Create app registration for `vault-int-ca` and log client ID, secret ID, and SP object
3. Provide permissions to MHSM resource group

## Configure Azure Managed HSM

### Configure management plane permissions (Azure RBAC)

Using a Managed HSM Administrator account (specified during Managed HSM creation), create role assignments to allow Crypto User permissions on the specified key scopes.  

>NOTE: Providing access to `/keys` scope for this demo. In production, you may want to provide access to specific keys.

1. Create a role assignments to allow the **`vault-root-ca`** application to manage keys in HSM
Provide access to `/keys`
```
az keyvault role assignment create --hsm-name stella-test-hsm --role "Managed HSM Crypto User" --assignee <vault-root-ca SP Object ID> --scope /keys
```

2. Create a role assignment to allow the **`vault-int-ca`** application to manage keys in HSM
Provide access to `/keys`
```
az keyvault role assignment create --hsm-name stella-test-hsm --role "Managed HSM Crypto User" --assignee <vault-int-ca SP Object ID> --scope /keys
```

### Configure HSM keys
1. Log into az cli with the `vault-root-ca` service principal
```
az login --service-principal -u <vault-root-ca Client ID> -p <Client Secret Value> --tenant <Tenant ID>
```

2. Create the HSM managed keys for `vault-root-ca`
```
az keyvault key create --hsm-name <HSM Name> --name <HSM root PKI Key Name> --ops wrapKey unwrapKey sign verify --kty RSA-HSM --size 3072
```

```
az keyvault key create --hsm-name <HSM Name> --name <HSM root Seal Key Name> --ops wrapKey unwrapKey --kty RSA-HSM --size 3072
```

3. Log into az cli with the `vault-int-ca` service principal
```
az login --service-principal -u <vault-int-ca Client ID> -p <Client Secret Value> --tenant <Tenant ID>
```

4. Create the HSM managed keys for `vault-int-ca`
```
az keyvault key create --hsm-name <HSM Name> --name <HSM int PKI Key Name> --ops wrapKey unwrapKey sign verify --kty RSA-HSM --size 3072
```

```
az keyvault key create --hsm-name <HSM Name> --name <HSM int Seal Key Name> --ops wrapKey unwrapKey --kty RSA-HSM --size 3072
```


## Configure Vault
---

### Configure Vault seal
1. Update config file with seal info
2. Perform unseal using "-migrate" switch

### Configure Vault Managed Key for Vault Root PKI

```
vault write sys/managed-keys/azurekeyvault/vault-root-pki \
	allow_generate_key=true \
	tenant_id=<Tenant ID> \
	client_id=<Client ID> \
	client_secret=<Client Secret> \
	vault_name=<HSM Vault Name> \
	key_name=vault-root-pki-key \
	resource=managedhsm.azure.net \
	key_bits=3072 \
	key_type=RSA-HSM
```

### Configure Vault Managed Key for Vault Int PKI
```
vault write sys/managed-keys/azurekeyvault/vault-int-pki \
	allow_generate_key=true \
	tenant_id=<Tenant ID> \
	client_id=<Client ID> \
	client_secret=<Client Secret> \
	vault_name=<HSM Vault Name> \
	key_name=vault-int-pki-key \
	resource=managedhsm.azure.net \
	key_bits=3072 \
	key_type=RSA-HSM
```

### Validate key access
```
vault write -f sys/managed-keys/azurekeyvault/<keyname>/test/sign
```
  
### Configure Root CA PKI Secrets Engine

1. Enable PKI secrets engine.

vault-root-ca
```
vault secrets enable -path=pki-root pki
```

2. Tune secrets engine to use managed keys.
```
vault secrets tune -allowed-managed-keys=vault-root-pki pki-root
```

3. Generate root certificate
```
vault write -field=certificate pki-root/root/generate/kms \
issuer_name="pki-root-issuer" \
managed_key_name=vault-root-pki \
common_name=<Common Name> \
ttl=8760h > root_ca.crt
```

4. Configure role for specifying issuer
```
vault write pki-root/roles/servers allow_any_name=true
```

5. Configure root CA and CRL URLs
```
vault write pki-root/config/urls \
issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
```

### Configure Intermediate CA PKI Secrets Engine

1. Enable PKI secrets engine.
```
vault secrets enable -path=pki-int pki
```

2. Tune secrets engine to use managed keys and set TTL
```
vault secrets tune -allowed-managed-keys=vault-int-pki -max-lease-ttl=43800h pki-int
```

3. Generate Int CA CSR for Root signing
```
vault write -format=json pki-int/intermediate/generate/kms \
issuer_name="pki-intermediate-issuer"
managed_key_name=vault-int-pki \
common_name=<Common Name> \
ttl=8760h | jq -r '.data.csr' > pki_intermediate.csr
```

4. Sign Int CA certificate with root CA 
>Perform this step on the Root CA!
```
vault write -format=json pki-root/root/sign-intermediate \
    issuer_ref="pki-root-issuer" \
    csr=@pki_intermediate.csr \
    format=pem_bundle ttl="43800h" \
    | jq -r '.data.certificate' > intermediate.cert.pem
```

5. Import signed certificate back into Vault Int
```
vault write pki-int/intermediate/set-signed certificate=@intermediate.cert.pem
```

### Validate issuing a certificate using managed key backed root

1. Configure a new role for a test domain in `pki-int`
```
vault write pki-int/roles/example-dot-com \
     issuer_ref="$(vault read -field=default pki-int/config/issuers)" \
     allowed_domains="example.com" \
     allow_subdomains=true \
     max_ttl="720h"
```

2. Request a new certificate for `test.example.com` based on the `example-dot-com` role.
```
vault write pki-int/issue/example-dot-com \
common_name="test.example.com" ttl="24"
```

### Configure Vault Managed Key for Vault Auto-unseal
```
vault write sys/managed-keys/azurekeyvault/pki-hsm \
allow_generate_key=true \
tenant_id=<Tenant ID> \
client_id=<Client ID> \
client_secret=<Client Secret> \
vault_name=<HSM Vault Name> \
key_name=vault-root-pki-seal-key \
resource=managedhsm.azure.net \
key_bits=2048 \
key_type=RSA-HSM
```
