
# K3s Install

Start with base ubuntu server. In the installer make sure to only install open ssh and not any other packages.

Most of this is done manually for now but will be migrated to the ansible roles.

## System INIT
```shell
sudo -i
apt update
apt install net-tools bash-completion
apt upgrade
# edit netplan and set a static IP Address
reboot
```

## k3s Install
```shell
sudo -i;
curl -sfL https://get.k3s.io | sh -;
grep 'alias k="kubectl"' /root/.bashrc || echo 'alias k="kubectl"' >> /root/.bashrc;
grep 'source <(kubectl completion bash)' /root/.bashrc || echo 'source <(kubectl completion bash)' >> /root/.bashrc;
source /root/.bashrc;
# Confirm K8s is operational
echo "PODS:";k get pods -A;echo;echo;echo "Nodes:";k get nodes;echo;echo;echo "Services";k get services -A;
# All the pods should be running
# Node should be status READY
# you should see traefik ingress has a LoadBalancer service with the external IP of the node.
```


## Helm Install
Go to the official github page for (Helm)[https://github.com/helm/helm] and on the releases page download the bin file and place it in /usr/local/bin

```shell
mkdir /tmp/helm
cd /tmp/helm
curl -LO https://get.helm.sh/helm-v4.1.1-linux-amd64.tar.gz
tar -xf helm-v4.1.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin
```

You will need to `export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"` in .bashrc for helm to see our k3s cluster.


## Nerdctl Install
Go to the official github page for (Nerdctl)[https://github.com/containerd/nerdctl] and on the releases page download the bin file and place it in /usr/local/bin

```shell
mkdir /tmp/nerd
cd /tmp/nerd
curl -LO https://github.com/containerd/nerdctl/releases/download/v2.2.1/nerdctl-2.2.1-linux-amd64.tar.gz
tar -xf nerdctl-2.2.1-linux-amd64.tar.gz
mv nerdctl /usr/local/bin
```

Configure nerdctl to point at the k3s hosted containerd
```shell
root@control_node_01:~# cat /etc/nerdctl/nerdctl.toml
address = "unix:///run/k3s/containerd/containerd.sock"
namespace = "k8s.io"
```


## Install ArgoCD
``` shell
helm repo add argo https://argoproj.github.io/argo-helm;
helm repo update;
kubectl create namespace argocd;
helm install argocd argo/argo-cd --namespace argocd
# kubectl port-forward svc/argocd-server -n argocd 8080:443

```

For now you can setup port forwarding to access it.
```shell
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080
```

Fetch the admin secret to access the web UI
```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode ; echo
```

## Run the ansible for further configurations

Run the ansible roles on the k3s server to generate CA Certificates for cert-manager to use and to configure a local git server for ArgoCD to point at for our GitOps configurations.

Edit //ansible/inventory/k3s/hosts.yaml with the hostname of your k3s server

Edit //ansible/role/

Make a copy of //ansible/roles/git_server/defaults/main.yaml into vars with your repo names and ssh public keys

Make sure to run ssh-copy-id on your local machine to the k3s server before running the playbook.

```shell
cd ansible
vim inventory/k3s/hosts.yaml
cp roles/git_server/defaults/main.yaml roles/git_server/vars/main.yaml
vim roles/git_server/vars/main.yaml
ansible-playbook -i inventory/k3s -K site.yaml
# you can also specify just one role with tags...
ansible-playbook -i inventory/k3s -K site.yaml --tags git_server
```
