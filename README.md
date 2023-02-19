# Setup Kubernetes Cluster


### Prerequisites:
- Docker installed on your machine (https://docs.docker.com/engine/install/)
- SSH key set up on your machine (https://cloud.google.com/compute/docs/connect/create-ssh-keys)
- Git installed (https://github.com/git-guides/install-git)
- At least two Ubuntu/Debian servers. (Cheapest ones will be at https://contabo.com/en)


#

## Setting up Ansible on Docker

    Ansible is a tool to manage multiple servers at once. We will have to copy our ssh key and run a few commands to enable kubernetes installation.
- Copy the file `ansible.Dockerfile`

- Run

 ```bash
 docker build -t ansible -f ansible.Dockerfile .
 ```

- Create aliases in your `.bashrc` or `.zshrc` for easy access of Ansible

```bash
alias ansible='docker run --rm -it -v $(pwd):/ansible/playbooks -v ~/.ssh:/root/.ssh --entrypoint=ansible ansible'
```

```bash
alias ansible-playbook='docker run --rm -it -v $(pwd):/ansible/playbooks -v ~/.ssh:/root/.ssh  ansible'
```

#

## Setting up Kubespray

- Clone the Kubespray repository:

```bash
git clone https://github.com/kubernetes-sigs/kubespray
cd kubespray
```

- Select Kubernetes plugins.
- This example uses `cni` for network plugin and `containerd` for container management, you can experiment with other plugins, but for the simplicity just use the ones provided here.

- Set `cni` network plugin
```bash
sed -i -r 's/^(kube_network_plugin:).*/\1 cni/g' inventory/sample/group_vars/k8s_cluster/k8s-cluster.yml
```

- Set deployment type to `host`

```bash
sed -i -r 's/^(etcd_deployment_type:).*/\1 host/g' inventory/sample/group_vars/etcd.yml
```

- Disable kube proxy

```bash
echo "kube_proxy_remove: true" >> inventory/sample/group_vars/k8s_cluster/k8s-cluster.yml
```

- Set container manager plugin to `containerd`

```bash
sed -i -r 's/^(container_manager:).*/\1 containerd/g' inventory/sample/group_vars/k8s_cluster/k8s-cluster.yml
```

#

### Setting up Kubespray Inventory

Run a command with your specified node-master and node-worker-nodes:

```cat << EOF > ./inventory/sample/hosts.yml
all:
  hosts:
    node1-master:
      ansible_host: 85.203.123.123
      ip: 85.203.123.123
      access_ip: 85.203.123.123
    node2-worker:
      ansible_host: 85.203.123.124
      ip: 85.203.123.124
      access_ip: 85.203.123.124
    node3-worker:
      ansible_host: 85.203.123.125
      ip: 85.203.123.125
      access_ip: 85.203.123.125
  children:
    kube-master:
      hosts:
        node1-master:
    kube-node:
      hosts:
        node1-master:
        node2-worker:
        node3-worker:
    etcd:
      hosts:
        node1-master:
    k8s-cluster:
      children:
        kube-master:
        kube-node:
EOF
```

- Copy your SSH key to your servers

```bash
ssh-copy-id root@85.203.123.123
ssh-copy-id root@85.203.123.124
ssh-copy-id root@85.203.123.125
```

- Disable firewalld on your nodes

```bash
ansible all -i inventory/sample/hosts.yml -a "systemctl stop firewalld" -b -v 
```

```bash
ansible all -i inventory/sample/hosts.yml -a "systemctl disable firewalld" -b -v 
```

## Install Kubernetes

Finally run the last command to install Kubernetes

```bash
ansible-playbook -i inventory/sample/hosts.yml cluster.yml -b -v
```

This can take some time for Kubespray to install all the required packages into your servers.

After installation your `kubeconfig` file will be ready inside your master machine:

```bash
/etc/kubernetes/admin.conf
```

Change `server: https://127.0.0.1:6443` to your master node IP and connect to your Kubernetes cluster
