# Ansible Collection - onaio.kubespray

## Overview

This documentation outlines the steps to be taken to bring up an on premise cluster using kubespray. The official documentation is [here](https://kubespray.io/#/). The following instructions will setup a cluster with; nfs volumes (dynamic provisioning), metallb load balancer (layer 2), ingress nginx, acme cluster issuer, containerd as CRI and calico as network plugin.

## Requirements

In addition to the [official kubespray requirements](https://github.com/kubernetes-sigs/kubespray#requirements) and [supported linux distributions](https://github.com/kubernetes-sigs/kubespray#supported-linux-distributions), add one VM to be used outside the cluster as an NFS server.

## Include library and role path for kubespray playbook on your ansible configurations
````cfg
....
library = ansible-collection-kubespray/external/kubespray/library
roles_path = roles:$VIRTUAL_ENV/usr/local/share/kubespray/roles:$VIRTUAL_ENV/usr/local/share/ansible/roles:/usr/share/kubespray/roles:ansible-collection-kubespray/external/kubespray/roles
````

## Environment Setup and Preparation

We recommend using this collection as a git submodule to your playbook repository. Do this by running:

```sh
git submodule add https://github.com/onaio/ansible-collection-kubespray.git collections/ansible_collections/onaio/kubespray
```

Then make sure the git submodules of the collection i.e kubespray and ansible nfs role are fetched by running below on the repo's root:

```sh
git submodule update --init --recursive
```

Create an ansible playbook on the root of your repo with the following contents (for our use case in this doc its called `kubernetes.yml`):

```yml
---
- import_playbook: collections/ansible_collections/onaio/kubespray/playbooks/kubespray.yml
```

### Install the dependencies for ansible to run the kubespray playbook

Install dependencies

```shell
pip3 install -r collections/ansible_collections/onaio/kubespray/external/kubespray/requirements-2.11.txt
pip3 install -r collections/ansible_collections/onaio/kubespray/requirements/base.pip
```

### Copy `collections/ansible_collections/onaio/kubespray/external/kubespray/inventory/sample/` as `inventories/<project>/kubernetes/<cluster-name>`

```shell
mkdir -p inventories/<project>/kubernetes/<cluster-name> && cp -rfp collections/ansible_collections/onaio/kubespray/external/kubespray/inventory/sample/* inventories/<project>/kubernetes/<cluster-name>
```

### Update Ansible inventory file with inventory builder

```shell
declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
CONFIG_FILE=inventories/<project>/kubernetes/<cluster-name>/hosts.yaml python3 collections/ansible_collections/onaio/kubespray/external/kubespray/contrib/inventory_builder/inventory.py ${IPS[@]}
```

### Review and change parameters under `inventories/<project>/kubernetes/<cluster-name>/group_vars`

```shell
cat inventories/<project>/kubernetes/<cluster-name>/group_vars/all/all.yml
cat inventories/<project>/kubernetes/<cluster-name>/group_vars/k8s_cluster/k8s-cluster.yml
```

### Modify default kubespray configurations

Open the generated `inventories/<project>/kubernetes/<cluster-name>/hosts.yaml` file and adjust nodes setup i.e choosing which nodes are masters and worker nodes. Also update the `ip` to the respective private ip and comment out the `access_ip`.

```yaml
 node1:
    ansible_host: #the public ip
    ip: #the private ip
    #access_ip: commented out
  node2:
```

The main configuration for the cluster is stored in `inventories/<project>/kubernetes/<cluster-name>/group_vars/k8s_cluster/k8s-cluster.yml`. In this file we will update the `supplementary_addresses_in_ssl_keys` with a list of the IP addresses or domain of the controller nodes. In that way we can access the kubernetes API server as an administrator from outside the private network.
i.e

```yaml
supplementary_addresses_in_ssl_keys: []
```

You can also see that the `kube_network_plugin` is by default set to 'calico'.

Ensure you have specified the metallb ip range. e.g 192.168.100.2/32 in `metallb_ip_range` config of `inventories/<project>/kubernetes/<cluster-name>/group_vars/k8s_cluster/addons.yml`.

## Deploying the cluster

To deploy Kubespray with Ansible Playbook - run the playbook as root. The option `--become` is required, as for example writing SSL keys in /etc/, installing packages and interacting with various systemd daemons, without --become the playbook will fail to run!

```shell
ansible-playbook -i inventories/<project>/kubernetes/<cluster-name> --become --become-user=root kubernetes.yml -t cluster --extra-vars "ansible_ssh_user=ubuntu"  -e "reset_confirmation=no"
```

Ansible will now execute the playbook, this can take up to 20 minutes.

## Actions after cluster setup using kubespray collection

The cluster uses metallb (layer2 type) as the load balancer, to make the cluster accessible to the public one has to run the following play `kubernetes.yml` with tag `post-cluster-setup` but before that we need to know the following:

*   The interface used by kubernetes network. (use `ifconfig`)
*   The external ip of ingress controller service.

```shell
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[].ip}'
```

### Update the following values in all.yml of your inventory accordingly

```yaml
kubespray_setup_type: "on-prem"

kubespray_nfs_provisioner:
  chart_version: "4.0.17"
  chart_ref: "nfs-subdir-external-provisioner"
#  chart_values:
#    image:
#      tag: v4.0.2
#    nfs:
#      path: /srv/nfs/storage
#      server: 192.168.0.8
#    storageClass:
#      defaultClass: true
#      accessModes: ReadWriteMany

kubespray_ingress_nginx:
#  chart_version: "4.2.5"
#  chart_values:
#    controller:
#      hostNetwork: true
#      kind: DaemonSet
#      dnsPolicy: ClusterFirstWithHostNet
#      service:
#        enableHttp: false
#        type: ClusterIP

kubespray_acme_cluster_issuer:
  email: "your-email@me"

cert_manager:
  version: v1.7.1

## Begin metallb config (When using metallb uncomment the configurations below)

# external ip for ingress nginx controller with port it maps to usually 80/443
# any another service to be accessed external on a custom port can be added here, provided it has an LoadBalancer service type.
# it matches the ip used for metallb 'metallb_ip_range' config mentioned above.
#port_ip_map:
#  - 192.168.100.2:80
#  - 192.168.100.2:443

# needed for metallb to work
#kube_proxy_strict_arp: true

#metallb_enabled: true
#metallb_speaker_enabled: true
#metallb_protocol: "layer2"
#metallb_port: "7472"
#
## metallb_controller_tolerations is needed to ensure the least downtime if the node holding the metallb controller goes down.
#metallb_controller_tolerations:
#  - key: "node.kubernetes.io/unreachable"
#    operator: "Exists"
#    effect: "NoExecute"
#    tolerationSeconds: 2
#  - key: "node.kubernetes.io/not-ready"
#    operator: "Exists"
#    effect: "NoExecute"

# default tunl0 for calico_network_backend IPIP mode and vxlan.calico for vxlan, if its same as the default one can omit the below variable. This is needed for metallb setup.
#kubespray_k8s_interface: tunl0
## end metallb config
```

### Update the following values in hosts.yml of your inventory accordingly if you have NFS clients and/or server

    nfs-servers:
      hosts:
        nfsserver:
          ansible_host: 10.10.11.2
          ip: 10.10.11.2
    nfs-clients:
      hosts:
        dbserver:
          ansible_host: 192.168.0.7
          ip: 192.168.0.7
      children:
        kube_node:

### Run the k8s post cluster setup play

This will setup the following:

| Setup      | Ansible tags |
|------------|-----------------|
|nfs servers & clients |  post-cluster-setup, nfs-setup, setup-nfs-server, setup-nfs-client   |
|cert-manager & cluster issuer | post-cluster-setup, setup-cert-manager, create-cluster-issuer |
|nfs provisioner | post-cluster-setup, add-helm-nfs-provisioner-chart-repo, helm-install-nfs-provisioner, deploy-helm-releases  |
|ingress nginx controller  | post-cluster-setup, add-helm-ingress-nginx-chart-repo, helm-install-ingress-nginx, deploy-helm-releases  |
|additional iptables to make cluster accessible publicly using layer2 metallb. (check the above 2 steps) | post-cluster-setup (applies only when `metallb_enabled` is true) |

```shell
ansible-playbook -i inventories/<project>/kubernetes/<cluster-name> kubernetes.yml -t post-cluster-setup --extra-vars "ansible_ssh_user=ubuntu"  -e "reset_confirmation=no"
```

There is a Gather Facts play on the kubespray that runs always. That can be skipped by appending `--skip-tags always` on your post cluster setup. 

At this point you can start installing the applications to the cluster.
