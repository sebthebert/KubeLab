# Kubernetes and Vault

https://kubernetes.io/docs/reference/kubectl/docker-cli-to-kubectl/

```shell
minikube start
```

## Vault

### Configuration

Start a **Dev Vault Server** in a container:

Build a new vault image with fixed VAULT_ADDR & VAULT_ROOT_TOKEN_ID:

```shell
docker build -t simple-vault:0.0.1 -f Dockerfile.vault .
```

Launch a container with this image:

```shell
docker run -d --name=simple-vault -p 8200:8200 --cap-add=IPC_LOCK simple-vault:0.0.1
```

Create a `dvault` alias:

```shell
alias dvault="docker exec -it simple-vault vault"
```

### Kubernetes configuration

Enable `kubernetes` Authentication:

```shell
curl --header "X-Vault-Token: token_root" -X POST -d '{"type":"kubernetes"}' http://127.0.0.1:8200/v1/sys/auth/kubernetes
```

Enable `approle` Authentication:

```shell
curl --header "X-Vault-Token: token_root" -X POST -d '{"type":"approle"}' http://127.0.0.1:8200/v1/sys/auth/approle
```

Create `kube_admin` role:
(values set for debugging, for better values read the [doc](https://www.vaultproject.io/api/auth/approle/index.html#parameters))

```shell
dvault write auth/approle/role/kube_admin \
    secret_id_ttl=12h \
    token_num_uses=0 \
    token_ttl=12h \
    token_max_ttl=24h \
    secret_id_num_uses=0

export kube_admin_role_id=$( dvault read auth/approle/role/kube_admin/role-id | grep role_id | xargs |cut -d' ' -f2 )

export kube_admin_secret_id=$( dvault write -f auth/approle/role/kube_admin/secret-id | grep 'secret_id ' | xargs |cut -d' ' -f2 )
```

Create `kube_read` role:

```shell
dvault write auth/approle/role/kube_read \
    secret_id_ttl=12h \
    token_num_uses=0 \
    token_ttl=12h \
    token_max_ttl=24h \
    secret_id_num_uses=0

export kube_read_role_id=$( dvault read auth/approle/role/kube_read/role-id | grep role_id | xargs |cut -d' ' -f2 )

export kube_read_secret_id=$( dvault write -f auth/approle/role/kube_read/secret-id | grep 'secret_id ' | xargs |cut -d' ' -f2 )
```

Create Policies:

```shell
docker cp kube_admin.policy.hcl simple-vault:/tmp
docker cp kube_read.policy.hcl  simple-vault:/tmp

dvault policy write kube_admin /tmp/kube_admin.policy.hcl
dvault policy write kube_read  /tmp/kube_read.policy.hcl
```

Apply Policies to Roles `kube_admin` and `kube_read`:

```shell
dvault write auth/approle/role/kube_admin policies="kube_admin"
dvault write auth/approle/role/kube_read policies="kube_read"
```

Create one `demo_secret` secret for `app1` & `app2` in `secret/my_big_company/kubernetes`:

```shell
dvault kv put secret/my_big_company/not_kubernetes/demo_secret value=not_kubernetes_secret
dvault kv put secret/my_big_company/kubernetes/app1/demo_secret value=app1_secret
dvault kv put secret/my_big_company/kubernetes/app2/demo_secret value=app2_secret
```