# Istio-Multicluster for OpenShift

This repo featreus an ansible playbook that installs istio-multicluster on a set of OpenShift clusters.

The following prerequisites have to be met:

1. The Pod's IPs must be routable between each other across all the cluster. To meet this requirement you can build an SDN network tunnel as described [here](https://blog.openshift.com/connecting-multiple-openshift-sdns-with-a-network-tunnel/)
2. The istio control plane must be installed in one of the cluster. To do so, you can follow the instruction on installing [Red Had OpenShift Service Mesh](https://docs.openshift.com/container-platform/3.11/servicemesh-install/servicemesh-install.html). If you want to run a quick trial, you can rune these commands:
```
oc new-project istio-operator
oc new-app -f artifacts/istio_product_operator_template.yaml --param=OPENSHIFT_ISTIO_MASTER_PUBLIC_URL=<master public url>
oc apply -f artifacts/openshift-servicemesh[-mTLS].yaml -n istio-operator
oc set env dc/router ROUTER_ALLOW_WILDCARD_ROUTES=true -n default
oc process -f artifacts/ingressgateway-route.yaml -p SUBDOMAIN=$(oc get route registry-console -n default -o jsonpath='{.spec.host}' | cut -d '.' -f 1 --complement) -n istio-system | oc apply -n istio-system -f -
```
use the `-mTLS` version of the servicemesh definition file, if you want to enable mTLS

3. If you want to enable mTLS between services, istio must be installed with an external ca. If you have installed istio with the above link, you can just run the following:
```
oc create secret generic cacerts -n istio-system --from-file=artifacts/certs/ca-cert.pem \
    --from-file=artifacts/certs/ca-key.pem --from-file=artifacts/certs/root-cert.pem \
    --from-file=artifacts/certs/cert-chain.pem
oc patch deployment/istio-citadel -n istio-system -p '{"spec": { "template": {"spec": { "containers": [{"args": ["--append-dns-names=true","--grpc-port=8060","--grpc-hostname=citadel","--citadel-storage-namespace=istio-system","--custom-dns-names=istio-pilot-service-account.istio-system:istio-pilot.istio-system,istio-ingressgateway-service-account.istio-system:istio-ingressgateway.istio-system","--self-signed-ca=false","--signing-cert=/etc/cacerts/ca-cert.pem","--signing-key=/etc/cacerts/ca-key.pem","--root-cert=/etc/cacerts/root-cert.pem","--cert-chain=/etc/cacerts/cert-chain.pem"],"name": "citadel","volumeMounts": [{"name":"cacerts","mountPath":"/etc/cacerts","readOnly":true}]}],"volumes":[{"name":"cacerts","secret":{"secretName":"cacerts","optional":true}}]}}}}'
oc delete secret istio.default -n istio-system
```
If for the above step, you didn't use the provided example ca certificates, make sure to ovveride the certificate location with the ansible var: `certificate_dir`

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
## Test mTLS

If you decided to deploy with mTLS enabled, here is a simple test you can run to make sure that everything is in place correctly.

Log in to your clusters:

```
oc login --username=<user1> --password=<pwd1> <url1>
CLUSTER1=$(oc config current-context)
oc login --username=<user2> --password=<pwd2> <url2>
CLUSTER2=$(oc config current-context)
```

Deploy bookinfo

```
oc --context $CLUSTER1 new-project bookinfo
oc --context $CLUSTER2 new-project bookinfo
oc --context $CLUSTER1 adm policy add-scc-to-user privileged -z default -n bookinfo
oc --context $CLUSTER2 adm policy add-scc-to-user privileged -z default -n bookinfo
oc --context $CLUSTER1 apply -f <(istioctl kube-inject -f artifacts/bookinfo.yaml) -n bookinfo
oc --context $CLUSTER2 apply -f <(istioctl kube-inject -f artifacts/bookinfo.yaml) -n bookinfo

oc --context $CLUSTER1 process -f artifacts/productpage-gateway.yaml -p SUBDOMAIN=$(oc --context $CLUSTER1 get route registry-console -n default -o jsonpath='{.spec.host}' | cut -d '.' -f 1 --complement) -n bookinfo | oc --context $CLUSTER1 apply -n bookinfo -f -
```

Configure the `reviews`, `details` and `ratings` services to require mTLS authentication by appliying a istio Policy:
```
oc --context $CLUSTER1 apply -f artifacts/mTLS-policy.yaml -n bookinfo
```

Configure the mesh to use mTLS authentication when communicating with the `reviews`, `details` and `ratings` services, by applying a destination rule:
```
oc --context $CLUSTER1 apply -f artifacts/destination-rules.yaml -n bookinfo
```

Notice that the product page service is still exposed with now security, this makes it simple for us to test.
```
oc get route productpage -n bookinfo
```
Point the browser to the output of the above command and navigate the app.



@TODO Add of securing the control plane