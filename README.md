# Telco Container-as-a-Service on Baremetal
This tutorial walks through the set up of kubernetes on Bare Metal using the projects from the opensource and the CNCF.

# Target audience
Anyone planning to support a Kubernetes in production. Could be system engineer, devops engineer, kubernetes operators.

# Prerequisites
For Bare metal, I'm using HPE Synergy Composable infrastructure with HPE OneView API and HPE 3PAR for strorage (block and persistent volume)
Further investigation with redfish is in progress.

# Quick Start
## Deprovisionning
### Uninstalling kubespray
```
conda activate python36
cd /home/tdovan/workspace/github/kubespray
ansible-playbook -i inventory/orange/inventory.ini reset.yml -b
```

# Step-by-Spep
I have break down each steps:
* [00-deploy-hardware](00-deploy-hardware/README.md)
* [01-create-golden-image](01-create-golden-image/README.md)
* [02-create-oneview-server-template](02-create-oneview-server-template/README.md)
* [03-provision-bare-metal-server](03-provision-bare-metal-server/README.md)
* [04-deploy-kubespray](04-deploy-kubespray/README.md)
* [05-customzie-kubernetes](05-customzie-kubernetes/README.md)


