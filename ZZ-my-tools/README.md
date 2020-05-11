# as a System engineer
```
tmux ls
``` 

```
grep, ag, lsof, nmap, net-tools, pciutils
lsof -i :443
``` 

```
sed, awk
``` 

```
yum install pciutils
ethtool -i ens3f0 
lspci -vvv -s 0000:0c:00.0
0c:00.0 Ethernet controller: Broadcom Inc. and subsidiaries BCM57840 NetXtreme II Ethernet Multi Function (rev 11)
        Subsystem: Hewlett-Packard Company 3820C 10/20Gb Converged Network Adapter (NPAR 1.5)
        Physical Slot: 3

lsscsi
[0:0:0:0]    storage HP       P240nr           7.00  -
[0:1:0:0]    disk    HP       LOGICAL VOLUME   7.00  /dev/sda
[1:0:0:0]    storage HP       P542D            7.00  -
[1:0:1:0]    enclosu HPE      D3940 Stor Mod   3.85  -
[1:0:2:0]    enclosu HPE      D3940 Stor Mod   3.85  -
[1:0:3:0]    enclosu HPE      12G SAS Conn Mod 0.86  -
[1:0:4:0]    enclosu HPE      12G SAS Conn Mod 0.86  -
[2:0:0:1]    disk    3PARdata VV               3224  /dev/sdb
[2:0:0:254]  enclosu 3PARdata SES              3224  -
[2:0:1:1]    disk    3PARdata VV               3224  /dev/sdc
[2:0:1:254]  enclosu 3PARdata SES              3224  -
[3:0:0:0]    disk    HP iLO   Internal SD-CARD 2.10  /dev/sdd

``` 


 for, while

# as a DevOps engineer
```
github, gitlab
docker, helm, kustomize
Ansible: automate everything
Python, conda, go (Jinja, sprig)
``` 

# as a kubernetes administrator
``` 
Kubespray: deploying and managing k8s but not the only one
kubectl (alias k & completion): cant’ do without
kns, ktx: switch ns and context
Stern: log
Krew manage plugin
k9s
k run --restart=Never --rm -it --image=busybox tan-debug
```