
# K3s Install

Start with base ubuntu server. In the installer make sure to only install open ssh and not any other packages.

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
```shell
foo
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

## Setup a local git server to host the git repo for ArgoCD
