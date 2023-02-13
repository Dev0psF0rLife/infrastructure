## Vault Installation to Amazon Elastic Kubernetes Service via Helm ##

**Helm:** 
```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

**AWS Configuration**
This command prompts you to enter an AWS access key ID, AWS secret access key, and default region name.
```
$ aws configure
```

Create a keypair to enable you to SSH into created nodes.
```
aws ec2 create-key-pair --key-name learn-vault
```

**Create Cluster**
```
eksctl create cluster \
    --name learn-vault \
    --nodes 3 \
    --with-oidc \
    --ssh-access \
    --ssh-public-key learn-vault \
    --managed

```
**Install the MySQL Helm chart**
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

```
helm install mysql bitnami/mysql
```

Create a variable named ROOT_PASSWORD that stores the mysql root user password.
```
ROOT_PASSWORD=$(kubectl get secret --namespace default mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)
```

**Install the Vault Helm chart**

Add the HashiCorp Helm repository.
```
helm repo add hashicorp https://helm.releases.hashicorp.com
```

```
helm repo update
```

Install the latest version of the Vault Helm chart in HA mode with integrated storage.
```
helm install vault hashicorp/vault \
    --set='server.ha.enabled=true' \
    --set='server.ha.raft.enabled=true'

```
Retrieve the status of Vault on the vault-0 pod.
```
kubectl exec vault-0 -- vault status
```

**Initialize and unseal one Vault pod**

Initialize Vault with one key share and one key threshold.
```
kubectl exec vault-0 -- vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    -format=json > cluster-keys.json
```

Display the unseal key found in cluster-keys.json.
```
cat cluster-keys.json | jq -r ".unseal_keys_b64[]"
```

Create a variable named VAULT_UNSEAL_KEY to capture the Vault unseal key.
```
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```

Unseal Vault running on the vault-0 pod.
```
kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

Retrieve the status of Vault on the vault-0 pod
```
kubectl exec vault-0 -- vault status
```