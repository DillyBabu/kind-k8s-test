Kind (Kubernetes in Docker) is primarily used for local Kubernetes development and testing.

## Pre-requisites:
My testing environment:
OS - Windows 11 Home Edition (WSL enabled)
RAM - 36 GB
IDE - VSCode

## Install Docker Desktop, which is required to run containers locally.
Install Docker desktop ## https://docs.docker.com/desktop/setup/install/windows-install/

## Install kubectl, the command-line tool for interacting with Kubernetes clusters.
Install kubectl ## https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/

## Download the Kind binary for Windows.
Install kind ## https://kind.sigs.k8s.io/docs/user/quick-start/##installation
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.27.0/kind-windows-amd64

## Move the Kind binary to a directory in your system's PATH for easy access.
Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe

# Install Kind Cluster:
kind create cluster --config kind.yaml

## List all Kind clusters currently available.
kind get clusters

Output: sample-cluster

## List K8s nodes
kubectl get nodes

Output:
| NAME                            | STATUS   | ROLES           | AGE    | VERSION   |
|--------------------------------|----------|-----------------|--------|-----------|
| sample-cluster1-control-plane   | Ready    | control-plane   | 5d5h   | v1.32.2   |
| sample-cluster1-worker          | Ready    | <none>          | 5d5h   | v1.32.2   |

## Deploy MetalLB, a load balancer implementation for Kubernetes.
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml

## Wait for the MetalLB controller deployment to become available.
kubectl wait --namespace metallb-system --for=condition=Available deployment/controller --timeout=90s

Output: deployment.apps/controller condition met

## Inspect the Docker network used by Kind and extract the subnet information.
docker inspect network kind | select-string subnet

Output: "Subnet": "172.18.0.0/16",

## Apply the MetalLB configuration file to set up IP address allocation. Select a range of IP's from the same IP range from the kind subnet output and modify the metalLB-config.yaml and apply.

kubectl apply -f .\metalLB-config.yaml

Output:
ipaddresspool.metallb.io/kind-pool created
l2advertisement.metallb.io/l2adv created

kubectl get svc -n metallb-system

Output:
| NAME                      | TYPE        | CLUSTER-IP      | EXTERNAL-IP   | PORT(S)   | AGE   |
|---------------------------|-------------|-----------------|--------------|-----------|-------|
| metallb-webhook-service   | ClusterIP   | 10.96.188.169   | <none>       | 443/TCP   | 4m7s  |

## Deploy the NGINX ingress controller in the Kind cluster.
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml

## Wait for the NGINX ingress controller pods to become ready.
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=90s

## List all services in the ingress-nginx namespace.
kubectl -n ingress-nginx get services

Output:
| NAME                                 | TYPE           | CLUSTER-IP     | EXTERNAL-IP    | PORT(S)                      | AGE   |
|-------------------------------------|----------------|----------------|----------------|------------------------------|-------|
| ingress-nginx-controller             | LoadBalancer   | 10.96.197.9    | 172.18.0.180   | 80:31408/TCP,443:32418/TCP   | 67s   |
| ingress-nginx-controller-admission   | ClusterIP      | 10.96.79.182   | <none>         | 443/TCP                      | 67s   |

## Apply an example ingress resource to test the NGINX ingress controller.
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/usage.yaml

Output:
pod/foo-app created
service/foo-service created
pod/bar-app created
service/bar-service created
ingress.networking.k8s.io/example-ingress created

## List all pods across all namespaces.
kubectl get pods -A

## List all services across all namespaces.
kubectl get services -A

Output:

| NAMESPACE      | NAME                                 | TYPE           | CLUSTER-IP      | EXTERNAL-IP    | PORT(S)                      | AGE   |
|----------------|--------------------------------------|----------------|-----------------|----------------|------------------------------|-------|
| default        | bar-service                         | ClusterIP      | 10.96.35.149    | <none>         | 8080/TCP                     | 31s   |
| default        | foo-service                         | ClusterIP      | 10.96.118.107   | <none>         | 8080/TCP                     | 31s   |
| default        | kubernetes                          | ClusterIP      | 10.96.0.1       | <none>         | 443/TCP                      | 9m39s |
| ingress-nginx  | ingress-nginx-controller            | LoadBalancer   | 10.96.197.9     | 172.18.0.180   | 80:31408/TCP,443:32418/TCP   | 2m46s |
| ingress-nginx  | ingress-nginx-controller-admission  | ClusterIP      | 10.96.79.182    | <none>         | 443/TCP                      | 2m46s |
| kube-system    | kube-dns                            | ClusterIP      | 10.96.0.10      | <none>         | 53/UDP,53/TCP,9153/TCP       | 9m38s |
| metallb-system | metallb-webhook-service             | ClusterIP      | 10.96.188.169   | <none>         | 443/TCP                      | 8m46s |

## Get details of the ingress-nginx service. This command helps in getting the Metal LB's load balancer IP for external communication.
kubectl get service -n ingress-nginx

Output:
| NAME                                 | TYPE           | CLUSTER-IP     | EXTERNAL-IP    | PORT(S)                      | AGE   |
|-------------------------------------|----------------|----------------|----------------|------------------------------|-------|
| ingress-nginx-controller             | LoadBalancer   | 10.96.197.9    | 172.18.0.180   | 80:31408/TCP,443:32418/TCP   | 3m56s |

## Forward port 8080 on your local machine to port 80 of the ingress-nginx service.
kubectl port-forward service/ingress-nginx-controller 8080:80 -n ingress-nginx

Output:
Forwarding from 127.0.0.1:8080 -> 80

## Verify nginx service:
curl http://localhost:8080/foo

foo-app

curl http://localhost:8080/bar

bar-app