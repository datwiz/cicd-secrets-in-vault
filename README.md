# CICD Secrets in Vault

## Overview
A pattern for introducing secrets management automtion into an existing CICD environment.  Secrets are added CICD job as environment variables.
This approach was inspired by [this post on setting env vars using a python script](http://blog.tintoy.io/2017/06/exporting-environment-variables-from-python-to-bash/)

## Assumptions
Assuming the following are available in the CICD environment.
* A job script oriented CICD automation stack that executes job tasks as a series of shell commands, such as GitLab-CI or Jenkins Pipelines.
* A secrets storage engine with a python API, such as Hashicorp Vault.

## Utility Scripts
The `get-secrets-by-app-role` script is based on using the Hashicorp Vault with the [AppRole auth method](https://www.vaultproject.io/docs/auth/approle.html).
Refer to the Vault documentation for further information on using the AppRole auth method.

### Required Environment Variables
| Variable | Description | Example |
| -------- | ----------- | ------- |
| VAULT_ADDR | url of the Vault service | https://vault.example.com:8200 |
| VAULT_ROLE_ID | Vault appRole ID to be used by the CICD job | db02de05-fa39-4855-059b-67221c5c2f63 |
| VAULT_SECRET_ID | Vault secret associated with the appRole ID | 6a174c20-f6de-a53c-74d2-6018fcceff64 |
| VAULT_NAMESPACE | Namespace in Vault, if omitted work without namespace e.g. for Hashi Vault OpenSource | name/space |
| VAULT_VAR_PREFIX | Prefix for the varibles holding vault pathes, optional, defaults to V_ | V_ |

### Workflow
An example workflow to replace a `FAKE_APP_PASSWORD` varialbe in a CICD job with a Vault sourced secret.

#### Prerequisites
* A Vault service with the appRole auth method and the key/value secrets engine enabled.
  This example assumes use of the key/value v2 version.
* Existing CICD job configured with the variable `FAKE_APP_PASSWORD=fake-password`

#### Workflow Steps
1.  Add a secret to Vault at the location `secret/fake-app/users/fake-user` with a key/value entry of `password=fake-password`

2.  Create a Vault policy for the CICD job (or set of CICD jobs) that includes 'read' access to the secret.
```
# cicd-fake-app-policy
path "secret/data/fake-app/users/fake-user" {
    capabilities = ["read"]
}
path "secret/metadata/fake-app/users/fake-user" {
    capabilities = ["list"]
}
```

3.  Create an appRole linked to the new policy.  This example creates a new appRole with an appRole secret TTL of 60 days and
a non-renewable access token with a TTL of 5 minutes.  The access token will be used by the CICD job to access the secret.
```
vault write auth/approle/role/fake-role \
    secret_id_ttl=1440h \
    token_ttl=5m \
    token_max_ttl=5m \
    policies=cicd-fake-app-policy
```

4.  Retrieve the newly created appRole ID.
```
vault read auth/approle/role/fake-role/role-id
```
Take note of the role-id returned.

5.  Create am appRole secret id associated with the new appRole
```
vault write -f auth/approle/role/fake-role/secret-id
```
Take note of the secret-id returned.

6.  In the CICD job defintion insert task steps to secrets variables and add to the job environment
```
script:
  - get-vault-secrets-by-approle > ${VAULT_VAR_FILE}
  - source ${VAULT_VAR_FILE} && rm ${VAULT_VAR_FILE}
```

optional to omit writing secrets to a file
```
script:
  - source <(get-vault-secrets-by-approle)
```


While the the helper script `get-vault-secrets-by-approle` could be executed and sourced in a single step splitting into 2 steps aids in
trouble shooting.  If executed and sourced in a single command, any error messages, such as permissions errors, get processed by the
`source` statement and don't get printed to the job logs.  

7.  In the CICD server, add vault access environment variables to the CICD job setup.
```
VAULT_ADDR=https://vault.example.com:8200
VAULT_ROLE_ID=db02de05-fa39-4855-059b-67221c5c2f63
VAULT_SECRET_ID=6a174c20-f6de-a53c-74d2-6018fcceff64
``` 

8.  In the CICD server, remove the `FAKE_APP_PASSWORD` variable from the job configuration.

9.  In the CICD server, add a new variable `V_FAKE_APP_PASSWORD=secret/fake-app/users/fake-user/password`

When the CICD job is next executed, the `FAKE_APP_PASSWORD` environment variable will be populated with the secret value sourced from Vault.
