# Building VM images using OpenShift Pipelines/Tekton

With OpenShift Virtualization, the complete lifecycle of virtual machines can be controlled over the API by utilizing YAML-formatted manifests describing the virtual machines' specifications and state.

The same is true for the lifecycle of the disk images that are referenced by the virtual machines' templates.

In the simplest deployments, the use of vanilla images such as the RHEL KVM guest image, combined with cloud-init config, is enough to bring a virtual machine to life.

As a foundation for the creation of golden images you can utilize any OpenShift cluster with OpenShift Virtualization and Red Hat OpenShift Pipelines installed. A storage class that provides file and block storage must also be in place.

Start by making a KVM guest image available (e.g., the RHEL KVM guest image from the Red Hat Portal). It can be fetched with a resource type DataVolume, which utilizes the CDI (Containerized Data Importer) to import the qcow2-formatted image into a PV (Persistent Volume) in OpenShift.

## Prerquisits

The following procedure is based on these Cluster prerequisites:

- OpenShift 4.13
- Red Hat OpenShift Pipelines
- OpenShift Virtualization Operator
- File and block storage, block storage with RWX support is required VM live migration

Make sure that the ClusterTasks are available by checking the featuregate of the Hyperconverged Object. The key deployTektonTaskResources must be set to "true".

```shell
oc get hyperconverged -n openshift-cnv kubevirt-hyperconverged -o json| jq .spec.featureGates.deployTektonTaskResources
false

oc patch hyperConverged -n openshift-cnv kubevirt-hyperconverged -p '{"spec":{"featureGates":{"deployTektonTaskResources":true}}}' --type=merge
hyperconverged.hco.kubevirt.io/kubevirt-hyperconverged patched
```
???

You should now be able to find the ClusterTask named disk-virt-customize:

```shell
oc get tasks -n openshift-cnv
NAME                      AGE
cleanup-vm                1m
copy-template             1m
create-vm-from-manifest   1m
create-vm-from-template   1m
disk-virt-customize       1m
disk-virt-sysprep         1m
modify-data-object        1m
modify-vm-template        1m
modify-windows-iso-file   1m
wait-for-vmi-status       1m
```

Since 4.16: The KubeVirt Tekton tasks are now shipped as a part of the OpenShift Container Platform Pipelines catalog

You will find the tasks under https://github.com/openshift-pipelines/tektoncd-catalog/tree/p/tasks
For installtion you can use ArtifactHub
Tasks: https://artifacthub.io/packages/search?repo=redhat-tekton-tasks&page=1

```shell
export OCPVersion=4.16.2
kubectl apply -f https://github.com/openshift-pipelines/tektoncd-catalog/raw/p/tasks/cleanup-vm/$OCPVersion/cleanup-vm.yaml
kubectl apply -f https://github.com/openshift-pipelines/tektoncd-catalog/raw/p/tasks/copy-template/$OCPVersion/copy-template.yaml
kubectl apply -f https://github.com/openshift-pipelines/tektoncd-catalog/raw/p/tasks/create-vm-from-manifest/$OCPVersion/create-vm-from-manifest.yaml
kubectl apply -f https://github.com/openshift-pipelines/tektoncd-catalog/raw/p/tasks/create-vm-from-template/$OCPVersion/create-vm-from-template.yaml
kubectl apply -f https://github.com/openshift-pipelines/tektoncd-catalog/raw/p/tasks/disk-virt-customize/$OCPVersion/disk-virt-customize.yaml
kubectl apply -f https://github.com/openshift-pipelines/tektoncd-catalog/raw/p/tasks/disk-virt-sysprep/$OCPVersion/disk-virt-sysprep.yaml
kubectl apply -f https://github.com/openshift-pipelines/tektoncd-catalog/raw/p/tasks/modify-data-object/$OCPVersion/modify-data-object.yaml
kubectl apply -f https://github.com/openshift-pipelines/tektoncd-catalog/raw/p/tasks/modify-vm-template/$OCPVersion/modify-vm-template.yaml
kubectl apply -f https://github.com/openshift-pipelines/tektoncd-catalog/raw/p/tasks/modify-windows-iso-file/$OCPVersion/modify-windows-iso-file.yaml
kubectl apply -f https://github.com/openshift-pipelines/tektoncd-catalog/raw/p/tasks/wait-for-vmi-status/$OCPVersion/wait-for-vmi-status.yaml
task.tekton.dev/cleanup-vm created
task.tekton.dev/copy-template created
task.tekton.dev/create-vm-from-manifest created
task.tekton.dev/create-vm-from-template created
task.tekton.dev/disk-virt-customize created
task.tekton.dev/disk-virt-sysprep created
task.tekton.dev/modify-data-object created
task.tekton.dev/modify-vm-template created
task.tekton.dev/modify-windows-iso-file created
task.tekton.dev/wait-for-vmi-status created
```
???

## Create a Project

oc new-project image-builde

## Create RHEL9 Image

- `modify-data-object` task imports a PVC from RHEL image. The name of the PVC is generated.
- `disk-virt-customize` task runs virt-customize commands on the PVC that install git, vim, pip and flask python framework.
- `generate-ssh-keys` task generates two secrets with private and public keys. The name of the secrets are generated. The task itself runs in parallel with 1. and 2. tasks.
- `create-vm-from-manifest` task creates a VM called flasker-vm-* from the prepared PVC with a public key attached.
- `execute-in-vm` task starts a VM and makes SSH connection to it.
clones flask repository

### modify-data-object

At this point you need to make an QCOW2 image available, which you want to customize. To get the image into your cluster you can utilize a DataVolume; this imports the image from a source of your choice directly into a PVC.

The source can be an existing PVC, that contains an image that you clone, or you can pull it from the web or use an image in a container registry.

    - name: modify-data-object
      params:
        - name: manifest
          value: |
            apiVersion: cdi.kubevirt.io/v1beta1
            kind: DataVolume
            metadata:
              generateName: rhel-server-
            spec:
              pvc:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 5Gi
                volumeMode: Filesystem
              source:
                registry:
                  url: docker://registry.redhat.io/rhel9/rhel-guest-image:9.2
                  pullMethod: node

In this case, you  use the RHEL cloud-image which you download from a container registry.
source:                
  registry:
    url: docker://registry.redhat.io/rhel9/rhel-guest-image:9.2
    pullMethod: node

We can also use URLs directly.
source:
  http:
    url: https://archives.fedoraproject.org/pub/archive/fedora/linux/releases/35/Cloud/x86_64/images/Fedora-Cloud-Base-35-1.2.x86_64.raw.xz

You could also clone an existing image, which is managed by a DataSource from the openshift-virtualization-os-images namespace.

sourceRef:
  kind: DataSource
  name: rhel9
  namespace: openshift-virtualization-os-images

The target PVC needs to be created in volumeMode Filesystem, so you can directly mount and modify it in the virt-customize container.


```

## Summery

More information on Tekton tasks related to OpenShift virtualization is available [here](https://github.com/kubevirt/kubevirt-tekton-tasks).