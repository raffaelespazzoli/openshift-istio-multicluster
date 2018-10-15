# Insatll istio

export the endpoints ips

```
export PILOT_POD_IP=$(kubectl -n istio-system get pod -l istio=pilot -o jsonpath='{.items[0].status.podIP}')
export POLICY_POD_IP=$(kubectl -n istio-system get pod -l istio-mixer-type=policy -o jsonpath='{.items[0].status.podIP}')
export STATSD_POD_IP=$(kubectl -n istio-system get pod -l istio=statsd-prom-bridge -o jsonpath='{.items[0].status.podIP}')
export TELEMETRY_POD_IP=$(kubectl -n istio-system get pod -l istio-mixer-type=telemetry -o jsonpath='{.items[0].status.podIP}')
export ZIPKIN_POD_IP=$(kubectl -n istio-system get pod -l app=jaeger -o jsonpath='{range .items[*]}{.status.podIP}{end}')
```

for each remote:
```
oc new-project istio-system

helm template install/kubernetes/helm/istio-remote --namespace istio-system \
--name istio-remote \
--set global.remotePilotAddress=${PILOT_POD_IP} \
--set global.remotePolicyAddress=${POLICY_POD_IP} \
--set global.remoteTelemetryAddress=${TELEMETRY_POD_IP} \
--set global.proxy.envoyStatsd.enabled=true \
--set global.proxy.envoyStatsd.host=${STATSD_POD_IP} \
--set global.remoteZipkinAddress=${ZIPKIN_POD_IP} | oc apply -f -
```
create a kube config file to be used by the main cluster from to connect to the remote clusters using the istio-mini. ca files must be gathered

```
apiVersion: v1
clusters:
   - cluster:
       certificate-authority-data: ${CA_DATA}
       server: ${SERVER}
     name: ${CLUSTER_NAME}
contexts:
   - context:
       cluster: ${CLUSTER_NAME}
       user: ${CLUSTER_NAME}
     name: ${CLUSTER_NAME}
current-context: ${CLUSTER_NAME}
kind: Config
preferences: {}
users:
   - name: ${CLUSTER_NAME}
     user:
       token: ${TOKEN}
```
create the kubeconfig as a scret and label it for each remote in the main cluster
```


kubectl create secret generic ${CLUSTER_NAME} --from-file ${KUBECFG_FILE} -n ${NAMESPACE}
kubectl label secret ${CLUSTER_NAME} istio/multiCluster=true -n ${NAMESPACE}
```

install bookinfo 
oc new-project bookinfo
oc adm policy add-scc-to-user privileged -z default -n bookinfo
oc apply -f <(istioctl kube-inject -f artifacts/bookinfo.yaml) -n bookinfo
oc expose svc productpage -n bookinfo


ansible-playbook -vv -i ~/git/openshift-enablement-exam/misc/istio-mesh-extension/ansible/inventory ~/git/openshift-istio-multicluster/ansible/playbooks/istio-multicluster/deploy-istio-multicluster.yaml

ansible nodes -vv -i /tmp/git/openshift-enablement-exam/misc/casl/inventory --private-key=~/.ssh/rspazzol-etl3.pem -e openstack_ssh_public_key=rspazzol -m shell -a "sysctl -w vm.max_map_count=262144"

oc --context $CLUSTER1 new-project istio-operator
oc --context $CLUSTER1 new-app -f artifacts/istio_product_operator_template.yaml --param=OPENSHIFT_ISTIO_MASTER_PUBLIC_URL=https://console.raffa1.casl-contrib.osp.rht-labs.com:8443 -n istio-operator
oc --context $CLUSTER1 apply -f artifacts/openshift-servicemesh.yaml -n istio-operator 