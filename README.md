# OpenShift Virtualization and OpenShift GitOps
Managing virtual machines of OpenShift Virtualization with OpenShift GitOps.

# TL;DR

## Provision the setup
```sh
# GitOps Operator
oc apply -f operators/gitops/operator-gitops.yaml

# OpenShift Virtualization
oc create -f operators/virtualization/operator-virtualization.yaml
# HyperConvergedController CR
oc apply -f operators/virtualization/hyperconverged.yaml

# MTV Operator
oc create -f operators/mtv/operator-mtv.yaml
# ForkliftController CR
oc apply -f operators/mtv/forklift.yaml

oc get secret/openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d

oc apply -f applicationsets/demo-vm/applicationset-demo-vm.yaml
oc apply -f applicationsets/demo-db/applicationset-demo-db.yaml
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n dev-demo-vm
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n prod-demo-vm
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n dev-demo-db
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n prod-demo-db
```

With the group `cluster-admins` you can easily define OpenShift users as admins in OpenShift Gitops. By default the `admin` user is a member of this group and you can login via Openshift no longer is the gitops password needed.

## Demonstrate route to production VMs:
```sh
oc get vmi -n prod-demo-vm -o custom-columns=VirtualMachineInstance:.metadata.name,Phase:.status.phase
VirtualMachineInstance   Phase
prod-demo-vm-1           Running
prod-demo-vm-2           Running

endpoint=$(oc get route -n prod-demo-vm prod-my-route  -ojsonpath='{.spec.host}')
for i in {1..10}; do curl http://$endpoint; done
This is demo VM 1 :)
This is demo VM 1 :)
This is demo VM 1 :)
This is demo VM 2 :)
This is demo VM 1 :)
This is demo VM 2 :)
This is demo VM 2 :)
This is demo VM 1 :)
This is demo VM 2 :)
This is demo VM 2 :)

```

## Connect from the VM to the database inside a container:
```sh
oc -n dev-demo-db get svc
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
dev-my-db   ClusterIP   X.X.X.X         <none>        3306/TCP   133m

virtctl -n dev-demo-vm console dev-demo-vm-1

mysqlshow -u developer -pdeveloper -h dev-my-db.dev-demo-db.svc.cluster.local
+--------------------+
|     Databases      |
+--------------------+
| information_schema |
| performance_schema |
| sampledb           |
+--------------------+

exit
```

## Connect from the database pod/container to the VM service:
```sh
endpoint=$(oc get route -n dev-demo-vm dev-my-route  -ojsonpath='{.spec.host}')
dbpod=$(oc -n dev-demo-db get pods -l=app=my-db -oname)
oc -n dev-demo-db exec -it  $dbpod -- curl http://$endpoint
This is demo VM 1 :)
```

## Install NMstate
```sh
# NMstate Operator
oc create -f operators/nmstate/operator-nmstate.yaml
# nmstate CR
oc apply -f operators/nmstate/nmstate.yaml
```

## Install VMExamples
```sh
# Fedora with template (current 39)
oc apply -f vmexamples/fedora01.yaml
# Fedora with boot image downloaded (35)
oc apply -f vmexamples/fedora02.yaml
# Windows 2019 with template and ISO image downloaded and install script
oc apply -f vmexamples/windows.yaml
# Windows 2022 with template and ISO image downloaded and install script
oc apply -f vmexamples/win2k22.yaml
# Windows 2025 with template and ISO image downloaded and install script
oc apply -f vmexamples/win2k25.yaml
```

In the original labs, the disk source is `http://192.168.123.100:81/Fedora35.qcow2` and `http://192.168.123.100:81/Windows2019.iso`.

## Attaching a virtual machine to the OVN-Kubernetes secondary network
```sh
oc apply -f vmexamples/network/localnet1-nncp.yaml
oc apply -f vmexamples/network/localnet1-network.yaml
```

Attention: When you use a different NS/project, you have to change the value of `netAttachDefName` in `localnet1-network.yaml`.

Edit the virtual machine, configuration ,add network interface and select `Network: vmexamples/localnet1-network`.

[Configuration for a localnet topology](https://docs.openshift.com/container-platform/4.15/networking/multiple_networks/configuring-additional-network.html#configuring-additional-network_ovn-kubernetes-configuration-for-a-localnet-topology)

[Creating a NAD for localnet topology using the CLI](https://docs.openshift.com/container-platform/4.15/virt/vm_networking/virt-connecting-vm-to-ovn-secondary-network.html#virt-creating-localnet-nad-cli_virt-connecting-vm-to-ovn-secondary-network)

[Attaching a virtual machine to an OVN-Kubernetes secondary network using the CLI](https://docs.openshift.com/container-platform/4.15/virt/vm_networking/virt-connecting-vm-to-ovn-secondary-network.html#virt-attaching-vm-to-ovn-secondary-nw-cli_virt-connecting-vm-to-ovn-secondary-network)

## Migration Toolkit for Virtualization
```sh
# MTV Operator
oc create -f operators/mtv/operator-mtv.yaml
# ForkliftController CR
oc apply -f operators/mtv/forklift.yaml
```

## Migrated VM
The VMs are configured with static IPs, it is needed to reconfigure them to use DHCP.

- Start the VM web01/web02
- Open web01/web02, start the VM and access to the Console
- Login with user root and password R3dh....
- Run the following commands
```sh
nmcli con del "Wired connection 1"
nmcli con add type ethernet ifname eth0
# Review the IP address is 10.0.2.2 now
ip a
```

## Create Ubuntu VM from ISO download

```sh
# Ubuntu from ISO
oc apply -f vmexamples/ubuntu.yaml
```

When the installation is done, install the qemu-guest-agent as root (should already be done by the cloud-init)
```sh
sudo -i
apt update
apt -y install qemu-guest-agent
systemctl enable qemu-guest-agent
systemctl start qemu-guest-agent
```

Tekton Examples
https://github.com/tosin2013/openshift-virt-tekton-ref/tree/main/components/kubevirt-pipelines
fedora-example: Refactor fedora-cleanup pipeline tasks for consistency
microshift-instance: chore: Update microshift-instance-pipeline.yaml
ollama-langchain: adding routes
server-deployer: testing qemu
windows-bios-installer: adding fedora-example
windows-efi-installer: adding fedora-example