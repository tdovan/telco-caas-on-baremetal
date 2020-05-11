# 1/ Telco Container-as-a-Service on Baremetal
This tutorial walks through the set up of kubernetes on Bare Metal using the projects from the opensource and the CNCF.
but first, let's try to define what a  telco CaaS is:
- provide the features to run network functions (data plane acceleration SR-IOV/DPDK...)
- based on the industry standards (LinuxFoundation/CNCF, CNTT, ETSI, ONAP, opensource,..) and API driven
- provide tools to operate the CaaS and NFs (lifecycle management)
- provide tools to mesure the CaaS and NFs (telemetry, monitoring, alerting)
- provide tools to secure the CaaS and NFs (business continuity, runtime, vault, SIEM)
- provide tools to operate the NFs (Service Mesh, autoscaling ...)


![General workflow](images/general-workflow.png)


![Telco CaaS architecture](images/telco-caas.png)

# 2/ Target audience
Anyone planning to support a Kubernetes in production. Could be system engineer, devops engineer, kubernetes operators.

# 3/ Prerequisites
For Bare metal, I'm using HPE Synergy Composable infrastructure with HPE OneView API and HPE 3PAR for strorage (block and persistent volume)
Further investigation with redfish is in progress.

# 4/ Quick Start
## 4.1/ Deprovisionning k8s cluster (kubespray and bare metal servers)
### 4.1.1/ Uninstall kubespray (5m)
```
conda activate python36
cd /home/tdovan/workspace/github/kubespray
ansible-playbook -i inventory/orange/inventory.ini reset.yml -b
```

### 4.1.2/ Deprovision Bare Metal Server (5m)
```
conda activate python36
cd /home/tdovan/workspace/github/ansible-synergy-3par
ansible-playbook -i inventory/synergy-inventory tasks/ov-poweroff-delete-serverprofile.yaml --limit az1,az2,az3
ansible-playbook -e "ansible_python_interpreter=/home/tdovan/anaconda3/envs/python36/bin/python" -i inventory/synergy-inventory tasks/infra-deregister-dns.yml --limit az1,az2,az3
```

### 4.1.3/ Clear OneView alarm (1m)
```
Useful when using Synergy beta unit. It clears alarm of the server otherwise oneview will not allow to re-provision without clearing the faults

pwsh
$az1=Connect-HPOVMgmt -Appliance az1.tdovan.co -UserName $username -Password $password
$az2=Connect-HPOVMgmt -Appliance az1.tdovan.co -UserName $username -Password $password
$az3=Connect-HPOVMgmt -Appliance az1.tdovan.co -UserName $username -Password $password

Get-HPOVServer -ApplianceConnection $az1 | Get-HPOVAlert -State active | Set-HPOVAlert -Cleared
Get-HPOVServer -ApplianceConnection $az2 | Get-HPOVAlert -State active | Set-HPOVAlert -Cleared
Get-HPOVServer -ApplianceConnection $az3 | Get-HPOVAlert -State active | Set-HPOVAlert -Cleared
```

## 4.2/ Provisionning k8s cluster on Bare Metal
### 4.2.1/ Provisionning Bare Metal servers with HPE OneView (30m)
```
conda activate python36
cd /home/tdovan/workspace/github/ansible-synergy-3par
ansible-playbook -e "ansible_python_interpreter=/home/tdovan/anaconda3/envs/python36/bin/python" -i inventory/synergy-inventory 1-deploy-bfs-az-all.yaml --limit az1,az2,az3 --forks 20
ansible-playbook -i inventory/synergy-inventory 2-configure-kubespray-nodes.yaml --limit az1,az2,az3
```
### 4.2.2/ Deploy k8s cluster with kubespray (30m)
```
cd /home/tdovan/workspace/github/kubespray
ansible-playbook -i inventory/orange/inventory.ini  --become --become-user=root cluster.yml --flush-cache
```

### 4.2.3/ Connecting to the cluster (1m)
```
ansible-playbook -i inventory/synergy-inventory 3-merge-kubeconfig.yaml --limit localhost
ktx kubernetes-admin@cluster.local
k get nodes
```

## 4.3/ Customize k8s
### 4.3.1/ Persistent storage with HPE 3PAR CSI (5m)
```
this is the helm v3 chart but the operator is also available
https://operatorhub.io/operator/hpe-csi-driver-operator
https://scod.hpedev.io/csi_driver/index.html
https://hub.helm.sh/charts/hpe-storage/hpe-csi-driver


helm repo add hpe https://hpe-storage.github.io/co-deployments
helm repo update
cd /home/tdovan/workspace/github/co-deployments/helm/values/csi-driver/v1.2.0
kns kube-system
helm install hpe-csi hpe-storage/hpe-csi-driver --namespace kube-system -f values-hpedemocenter.yaml
k get pods --selector 'app in (primera3par-csp,hpe-csi-node,hpe-csi-controller)'

k delete sc hpe-standard
k apply -f 3par-sc.yaml
k apply -f hpe-csi-pvc.yaml
ssh 3paradm@$ip
showvv *pvc*

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/wordpress
helm install my-wordpress bitnami/wordpress --version 9.2.1 --set service.type=ClusterIP,wordpressUsername=admin,wordpressPassword=adminpassword,mariadb.mariadbRootPassword=secretpassword,persistence.existingClaim=pvc-testtan-1,allowEmptyPassword=false

k delete pods --selector 'app in (primera3par-csp,hpe-csi-node,hpe-csi-controller)'
k get pods --selector 'app in (primera3par-csp,hpe-csi-node,hpe-csi-controller)'
k apply -f hpe-csi-pvc.yaml

helm uninstall hpe-csi --namespace kube-system
``` 

### 4.3.2/ LoadBalancer as a Service with Metallb (3m)
```
cd /home/tdovan/workspace/k8s-apps/metallb
k apply -f namespace.yaml
./metallb-createsecret.sh
k apply -f metallb.yaml
k apply -f configmap-130-139.yaml
k get svc
k kubectl patch svc kubernetes-dashboard -p '{"spec": {"type": "LoadBalancer"}}'
k get svc
k create serviceaccount dashboard-admin-sa
k create clusterrolebinding dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa
kubectl describe secret kubernetes-dashboard-token-zcklt

Access k8s dashboard https://10.12.25.130 (with chrome, type: 'thisisunsafe' to skip ssl)

--> TODO: ok when NOT using RBAC
```

### 4.3.3/ Multi-homed pod with multus-cni (3 minutes)
```
https://github.com/intel/multus-cni/blob/master/doc/how-to-use.md

cd /home/tdovan/workspace/github/ansible-synergy-3par
ansible-playbook -i inventory/synergy-inventory 3-configure-multus.yml --limit az1,az2,az3

cd /home/tdovan/workspace/k8s-apps/multus-cni
cat ./images/multus-daemonset.yml | kubectl apply -f -
k get pods --selector 'app in (multus)'

k apply -f hpe_networkattachment-macvlan-conf-1.yaml 
k apply -f hpe_networkattachment-macvlan-conf-2.yaml 

k apply -f hpe_pod-multus-1macvlan.yaml
k exec -it pod-multus-1 -- ip a

k apply -f hpe_pod-multus-2macvlan.yaml
k exec -it pod-multus-2 -- ip a
```

### 4.3.4/ Service Mesh with Istio (10m)
```
cd /home/tdovan/workspace/k8s-apps/istio/
kubectl create namespace istio-system
istioctl manifest apply --set profile=demo
kns istio-system
kubectl patch svc kiali -p '{"spec": {"type": "LoadBalancer"}}'
go to kiali dashbaord: http://10.12.25.132:20001 (admin/admin)

kns default
k label namespace default istio-injection=enabled
cd /home/tdovan/workspace/k8s-apps/istio/istio-1.5.1
k apply -f samples/bookinfo/platform/kube/bookinfo.yaml
k exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
k apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get svc istio-ingressgateway -n istio-system

export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
echo $GATEWAY_URL

http://10.12.25.131:80/productpage (refresh page to round robin through services)
go to kiali dashbaord: http://10.12.25.132:20001 (admin/admin)
```


# Detailed Step-by-Step
I have break down each steps:
* [00-deploy-hardware](00-deploy-hardware/README.md)
* [01-create-golden-image](01-create-golden-image/README.md)
* [02-create-oneview-server-template](02-create-oneview-server-template/README.md)
* [03-provision-bare-metal-server](03-provision-bare-metal-server/README.md)
* [04-deploy-kubespray](04-deploy-kubespray/README.md)
* [05-customize-kubernetes](05-customize-kubernetes/README.md)


