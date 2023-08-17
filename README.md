# IaC tools

## Approach & Objective

- Pruning actively the blockchain nodes aren't working properly(on app.toml) : **pruning-keep-recent, pruning-interval**
- In addition, its disk size isn't diminished automatically.
- Propose the method to prune the blockchain nodes properly and reduce disk usage ideally.
- Approach process is using ansible framework to manage the remote nodes effectively and uniformly.
- We'll go through this by the Crescent blockchain(The one and only hybrid DEX on cosmos eco system)

## TL/DR

You run one playbook and set up a node.

```bash
ansible -i infra.yaml -m ping crescent_test

crescent_test | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

# for your snapshot upload
export AWS_ACCESS_KEY_ID=
export AWS_SECRET_ACCESS_KEY=

ansible-playbook pruning_job_deploy.yml -i infra.yaml -e "network=crescent target=crescent_test"
```

1. `infra.yaml`: Required, inventory file for your resources
1. `network`: Required. this will be on group name(ususally chain name) on inventory file
1. `target`: Required. should `all` or `representative expression of host` on inventory file

## Configure & How to do

1. Install and configure ansible.

```bash
## package install on remote server
sudo apt-add-repository ppa:ansible/ansible
sudo apt update
sudo apt install ansible

## generate the ssh keypair
ssh-keygen -t rsa -b 2048 -C "ansible keypair" -f ansible
> ansible     : private key
> ansible.pub : public key

## place the ssh keypair
(on remote server, add the pub key line)
vi ~/.ssh/authorized_keys

```

2. Build the inventory(vi infra.yaml)

```yaml
(on ansible host)

all:
  vars:
    ansible_user: ubuntu                                          
    ansible_port: 22
    ansible_ssh_private_key_file: '/home/ubuntu/.ssh/ansible'               ## ansible private ssh key
    ansible_python_interpreter: '/usr/bin/python3'
    
    user_dir: '/home/{{ ansible_user }}'
    path: '/sbin:/usr/sbin:/bin:/usr/bin:/usr/local/bin:/usr/local/go/bin'

    s3_uri: 's3://'                                                         ## s3 URI
    ## 3rd party repo
    cosmprund_repo: 'https://github.com/binaryholdings/cosmprund'
    cosmospruner_repo: 'https://github.com/notional-labs/cosmprund.git'

crescent:                                                                   ## host group name(usually chain name) 
  vars:
    folder: .{{ network }}
  hosts:
    crescent_test:                                                          ## host name
      ansible_host: x.x.x.x                                                 ## remote server IP address
      data_dir: '/data'                                                     ## CHAIN_HOME folder is will be ${DATA_DIR}//${NETWORK}
      wasm: true|'file'                                                     ## wasm folder exists and name is 'wasm' set to True, if folder name is different(like band 'file') and named it
      prune: 'cosmprund|pruner'                                             ## pruner selection 
      tx_index: true                                                        ## for indexer 'kv' node
      compact: false                                                        ## at first time, for disk diminishing set to 'true'
      prune_hour: 15                                                         ## pruning time based on UTC+0
      prune_minute: 10                                                      ## pruning time based on UTC+0
      snapshot: false                                                       ## if you want to upload snapshot to AWS s3
      extra:                                                                ## 3rd service for stopping during pruning
        - 'orchestrator'
        - 'oracle' 
```

|Variable name|Description|
|:--|:--|
|`ansible_user`|The sample file assumes `ubuntu`, but feel free to use other user name. This user need sudo privilege.|
|`ansible_port`|The sample file assumes `22`. But if you are like me, you will have a different ssh port other than `22` to avoid port sniffing.|
|`ansible_ssh_private_key_file`|The sample file assumes `~/.ssh/id_rsa`, but you might have a different key location.|
|`user_dir`|The user's home directory. In the sample inventory file this is a computed variable based on the ansible_user. It assumes that it is not a root user and its home directory is `/home/{{ansible_user}}`.|
|`path`|This is to make sure that the ansible_user can access the `go` executable.|
|`s3_uri`|This the address URI for s3 bucket|
|`ansible_host`|Required. The IP address of the server.|
|`wasm`|Wasm folder exists and name is 'wasm' set to True, if folder name is different(like band 'file') and named it|
|`prune`|pruner selection. There are 2 types of pruner. If possible, try for both of it|
|`tx_index`|for only notional pruner, if tries to prune node with indexer 'kv'|
|`compact`|at first time, for disk diminishing set to 'true'|
|`prune_hour`|pruning time based on UTC+0. for cronjob|
|`prune_minute`|pruning time based on UTC+0. for cronjob|
|`snapshot`|if you want to upload snapshot to AWS s3 or compatible(like Minio)|
|`extra`|3rd service for stopping during pruning|

## Ready to go

```bash
# for your snapshot upload
# for the security, used temporary variable export
export AWS_ACCESS_KEY_ID=
export AWS_SECRET_ACCESS_KEY=

ansible-playbook pruning_job_deploy.yml -i infra.yaml -e "network=crescent target=crescent_test"
```
