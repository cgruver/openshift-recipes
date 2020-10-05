# Installing Rook/Ceph on OpenShift in a disconnected network

This guide will lead you through the installation of Ceph storage in an OpenShift cluster which is isolated from internet access

## Prerequisites:

1. You need `cluster-admin` access to a running OpenShift 4 cluster (This guide was tested on OKD 4.5)
1. The OpenShift cluster needs at least 3 worker nodes with an unallocated block device.  (SSD is strongly recommended)
1. An image registry that is accessible from the OpenShift cluster.  You can use the OpenShift internal registry.
1. A workstation with internet access and either `podman` or `docker`

### Preparation:

1. Clone this repository:

    ```bash
    git clone https://github.com/cgruver/rook-ceph-disconnected-install.git
    cd rook-ceph-disconnected-install
    ```

1. Prepare the Rook/Ceph installation resources: (__Note: If using MacOS, the sed command will need empty `""` like - `sed -i "" "s|...`__)

    ```bash
    mkdir -p ~/ceph-workdir
    cp ./openshift-resources/*.yml ~/ceph-workdir
    LOCAL_REGISTRY=nexus.your.domain.com:5000
    sed -i "s|--LOCAL_REGISTRY--|${LOCAL_REGISTRY}|g" ~/ceph-workdir/cluster.yml
    sed -i "s|--LOCAL_REGISTRY--|${LOCAL_REGISTRY}|g" ~/ceph-workdir/operator-openshift.yml

1. Edit `~/ceph-workdir/ceph/cluster.yml` to reflect the worker nodes that you are using for storage, and the block deviced attached to the nodes

    Follow the docs here: [Rook Ceph Documentation](https://rook.github.io/docs/rook/v1.4/ceph-cluster-crd.html)

    The file provided assumes 3 worker nodes with a block device at /dev/sdb:

    ```yaml
    storage:
      useAllNodes: false
      useAllDevices: false
      config:
      nodes:
      - name: "okd4-worker-0.your.domain.org"
        devices:
        - name: "sdb"
          config:
            osdsPerDevice: "1"
      - name: "okd4-worker-1.your.domain.org"
        devices:
        - name: "sdb"
          config:
            osdsPerDevice: "1"
      - name: "okd4-worker-2.your.domain.org"
        devices:
        - name: "sdb"
          config:
            osdsPerDevice: "1"
    ```

### Make the Rook/Ceph images available to the OpenShift cluster:

The next step is to pull all of the images needed for the install, and make them available to the OpenShift cluster.

These instructions assume that you are using a personal workstation with Docker desktop.  You can also use `podman`, `skopeo`, or `buildah` to perform these steps.

The list of images is in the file `ceph-images`, included with this guide:

```bash
docker.io/rook/ceph:v1.4.0
docker.io/ceph/ceph:v15.2.4
quay.io/cephcsi/cephcsi:v3.0.0
quay.io/k8scsi/csi-node-driver-registrar:v1.2.0
quay.io/k8scsi/csi-resizer:v0.4.0
quay.io/k8scsi/csi-provisioner:v1.6.0
quay.io/k8scsi/csi-snapshotter:v2.1.1
quay.io/k8scsi/csi-attacher:v2.1.0
```

From your internet connected workstation or bastion host:

1. Pull the images needed for the install:

    ```bash
    for i in $(cat ceph-images)
    do 
        IMAGE_TAG=${LOCAL_REGISTRY}/$(echo $i | cut -d"/" -f2-)
        docker pull ${i}
        docker tag ${i} ${IMAGE_TAG}
    done
    ```

1. Log into your local image registry.  Since you are in a disconnected environment, you might have to change networks for this step.

    ```bash
    docker login ${LOCAL_REGISTRY}
    ```

1. Push the newly tagged images.

    ```bash
    for i in $(cat ceph-images)
    do 
        IMAGE_TAG=${LOCAL_REGISTRY}/$(echo $i | cut -d"/" -f2-)
        docker push ${IMAGE_TAG}
    done
    ```

### Now we are ready install Rook/Ceph

1. Log into your OpenShift cluster as a cluster administrator
1. Label your selected storage nodes:

    ```bash
    oc label nodes okd4-worker-0.your.domain.com role=storage-node
    ```

    Apply this label to each of the worker nodes that you have selected to host Ceph storage.

1. Apply the Rook/Ceph boilerplate:

    ```bash
    cd ~/ceph-workdir
    oc apply -f common.yml
    ```

    This will create the `rook-ceph` namespace and all of the resources needed for the Ceph cluster.

1. Install the Operator:

    ```bash
    oc apply -f operator-openshift.yml
    ```

1. Wait for all of the operator pods to reach a `Ready` state.

1. Create the Ceph cluster:

    ```bash
    oc apply -f cluster.yml
    ```

1. Wait for the cluster to deploy.

    You will see a whole lot of activity in the rook-ceph namespace.

    The cluster is available when you see a pod named `rook-ceph-osd-prepare-...` reach a completed state for each of your storage labeled worker nodes.

1. Create storage classes for Block and Obkect storage:

    ```bash
    oc apply -f ceph-storage-class.yml
    oc apply -f object-storage-class.yml
    ```

1. If you want to designate one of your new storage classes as the default for dynamic provisioning:

    ```bash
    oc patch storageclass rook-ceph-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    ```

    __Or:__

    ```bash
    oc patch storageclass rook-ceph-bucket -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    ```

### You now have a Ceph cluster capable of provisioning Block or S3 Object storage.
