# Istio-Multicluster for OpenShift

This repo featreus an ansible playbook that installs istio-multicluster on a set of OpenShift clusters.

The following prerequisites have to be met:

1. The istio control plane must be installed in one of the cluster. To do so, you can follow the instruction on installing [Red Had OpenShift Service Mesh](https://docs.openshift.com/container-platform/3.11/servicemesh-install/servicemesh-install.html).
2. The Pod's IPs must be routable between each other across all the cluster. To meet this requirement you can build an SDN network tunnel as described [here](https://blog.openshift.com/connecting-multiple-openshift-sdns-with-a-network-tunnel/)

Once you met the above requirements you can run the playbook to install istio-multicluster.

If you have used the link suggested above to install the network tunnel across the OpenShift SDNs, you can reuse the same inventory to run the istio-multiclutser playbook.

## Installing Istio-multicluster

See an example of the inventory [here](./ansible/inventory) and customize it for your clusters.

Here is a minimum inventory:
```
clusters:
- name: <cluster_1_name>
  url: <cluster_1_master_api_url>
  username: <cluster_1_username>
  password: <cluster_1_password>
  istio_control_plane: true  
- name: <cluster_2_name>
  url: <cluster2_master_api_url>
  username: <cluster_2_username>
  password: <cluster_2_password>
  istio_control_plane: false 
```
You must have exaclty one cluster with the `istio_control_plane` variable set to `true`.

You can choose the istio version that you want to install by setting the following variable: `istio_git_tag`.

You can run the playbook as follows:

```
ansible-playbook -i <inventory> ./ansible/playbooks/deploy-istio-multicluster/deploy-istio-multicluster.yaml
```