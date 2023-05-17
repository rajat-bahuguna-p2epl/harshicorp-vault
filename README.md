# vault-cluster-playbook

Cloud agnostic Ansible playbooks used to deploy vault cluster backed by raft storage engine in HA mode.

These playbooks generate certificate and gossip encryption key for vault cluster and bootstrap and unseal it using shamir secret sharing without needing to rely on any cloud specific kms provider or transit secrets engine. 

these playbooks are used on google cloud platform but their nature is provider agnostic, as long as you have ssh access to server's IP address , you should be able to setup a working, secure vault cluster.

These playbooks use vault's new raft storage engine so the cluster is not dependant on any other service ( like consul cluster for storage).

for sake of security, all secrets, such as certificates , are encrypted after generation with `ansible-vault`.

> `[NOTE]` : the documentation currently is pretty poor. I will update as soon as possible.

While I have been using this playbook for deploying vault cluster to on google compute engine, I make no guarantees. use at your own risk.

## procuring infrastructural resources

in case you do not want to use terraform, you can use the following commands with `gcloud` cli to setup infrastructure on google cloud.

### google cloud


1. make sure you ssh key is added to project wide metadata. make sure the ssh key you are copying has appropriate permissions by running `chmod 600 ~/.ssh/id_rsa` before this snippet.

```bash
gcloud compute project-info add-metadata \
  --metadata-from-file ssh-keys=<(\
    gcloud compute project-info describe --format json \
      | jq -r '.commonInstanceMetadata.items[]
               | if .key == "ssh-keys" then .value else empty end';
    echo "$USER:$(cat ~/.ssh/id_rsa.pub)")
```

2. confirm ssh keys were added by looking into project's metadata

```bash
# => [TODO] fix this. this snippet hung up terminal when I tried it. the error might be due to running it on a broken surface pro though.
gcloud compute \
                project-info describe \
                --format json | jq -r '.commonInstanceMetadata.items[] | select(.key | equals ("ssh-keys"))
```

3. generating three gce machines in  `us-east1-b` region

```bash
seq 3 | xargs -I {} gcloud compute instances create "us-east-agent-{}" \
 --zone "us-east1-b" \
 --machine-type "n1-standard-16" \
 --subnet "default" \
 --network-tier "PREMIUM" \
 --image "debian-10-buster-v20200521" \
 --image-project "debian-cloud" \
 --boot-disk-size "250GB" \
 --boot-disk-type "pd-ssd" \
 --boot-disk-device-name "us-east-agent-{}"
```

in case you need to remove created machines in  `us-east1-b` region, run the following snippet.

```bash
seq 3 | xargs -I {} gcloud --quiet \
                            compute instances delete \
                            --zone us-east1-b \
                            "us-east-agent-{}"
```


#### vault setup

a system running vault needs to have to following specs

- 4-8 cores
- 16-32 GB RAM
- 50 GB SSD


1. setup ingress firewall rule

```bash
gcloud compute firewall-rules create vault-ingress \
  --allow tcp:8200 \
  --target-tags "vault-api-ingress" \
  --direction "INGRESS"
```

2. setup egress firewall rule

```bash
gcloud compute firewall-rules create vault-egress \
  --allow tcp:8200 \
  --destination-ranges '0.0.0.0/0' \
  --target-tags "vault-api-egress" \
  --direction "EGRESS"
```

3. add created firewall rules to instances

```bash
seq 3 | xargs -I {} gcloud compute instances \
                            add-tags "us-east-agent-{}" \
                            --zone "us-east1-b" \
                            --tags=vault-api-ingress,vault-api-egress
```

4. list external IP addresses of created instances. use this to setup ansible inventory

```bash
gcloud compute instances list --format json | jq -r '.[] | select(.name | contains("us-east-agent")).networkInterfaces[].accessConfigs[].natIP'
```

when populating hosts, add `ansible_user=<controller username>` in front of every host record to make sure ansible logs in without any issues.

## general commands

- init directory structure

```bash
roles=("01-ansible-controller" "02-install-prerequisites" "03-install-vault" "04-vault-certificates" "05-configure-vault"
"06-unseal-vault")
for role in "${roles[@]}"; do ansible-generate -i google-cloud -r "$role" -p "$PWD"; done
```

- generate a random password to encrypt data with ansible vault and store it in `~/.gcloud_vault_pass.txt`. take a backup of this file since it is needed for decrypting secrets on ansible controller.

```bash
echo -n "$(dd if=/dev/urandom bs=64 count=1 status=none | base64)" | tee ~/.gcloud_vault_pass.txt
```

- encrypt sia wallet key

```bash
echo -n "memonics ..." | ansible-vault encrypt_string --vault-password-file="~/.gcloud_vault_pass.txt" --stdin-name 'sia_wallet_password'
```

## ansible commands

- check connectivity

```bash
ansible -i google-cloud -m ping all -vvv
```

look for lines that have `ESTABLISH SSH CONNECTION FOR USER:` . in case `USER: None`, ansible doesn't know which user name to use to connect to remote and would default to `root`. fix this by adding `ansible_user=<controller username>` in front of every host IP in `google-cloud` file or any other inventory file.

you can also manually check host connectivity by using the following snippet

```bash
gcloud compute instances list --format json | jq -r '.[] | select(.name | contains("us-east-agent")).networkInterfaces[].accessConfigs[].natIP' | xargs -I {} ssh -o UserKnownHostsFile=/dev/null -o CheckHostIP=no -o StrictHostKeyChecking=no  "$USER@{}"
```

- run complete vault deployment for google-cloud

```bash
ansible-playbook \
        -i google-cloud site.yml \
        --limit vault \
        --vault-password-file ~/.gcloud_vault_pass.txt
```

- decrypt and view content of files locked with ansible vault

```bash
ansible-vault view --vault-password-file ~/.gcloud_vault_pass.txt -- /path/to/file
```

- decrypt a variable (e.g `vault_root_token`)

```bash
ansible localhost -m debug -a var="vault_root_token" -e "@host_vars/vault_root_token.yml" --vault-password-file ~/.gcloud_vault_pass.txt
```

- setup ansible controller local software dependencies for

```bash
ansible-playbook \
        -i google-cloud site.yml \
        --tags 01-ansible-controller
```

- setup base dependencies for all inventory hosts

```bash
ansible-playbook \
        -i google-cloud site.yml \
        --tags 02-install-prerequisites
```

- install Vault

```bash
ansible-playbook \
        -i google-cloud site.yml \
        --limit vault \
        --tags 03-install-vault \
        --vault-password-file ~/.gcloud_vault_pass.txt
```

- generate certificates for vault

```bash
ansible-playbook \
        -i google-cloud site.yml \
        --limit vault \
        --tags 04-vault-certificates \
        --vault-password-file ~/.gcloud_vault_pass.txt
```

- configure vault

```bash
ansible-playbook \
        -i google-cloud site.yml \
        --limit vault \
        --tags 05-configure-vault \
        --vault-password-file ~/.gcloud_vault_pass.txt
```

- unseal vault

```bash
ansible-playbook \
        -i google-cloud site.yml \
        --limit vault \
        --tags 06-unseal-vault \
        --vault-password-file ~/.gcloud_vault_pass.txt
```

## references

- <https://learn.hashicorp.com/tutorials/vault/reference-architecture>
