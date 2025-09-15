# OpenShift Command Cheat Sheet (from RHOSO 18.0 Deploy Guide + Training Text)

> **Scope:** All `oc` (OpenShift) commands referenced in the RHOSO 18.0 deployment guide you shared **plus** additional commands extracted from your training text. Commands are grouped by phase/topic. Copy–paste ready.

---

## 0) Project & LVMS (TopoLVM) readiness

```bash
# Create and use the OpenStack project/namespace
oc new-project openstack
oc project openstack

# Inspect namespace labels and set pod security labels
oc get namespace openstack -ojsonpath='{.metadata.labels}' | jq
oc label ns openstack security.openshift.io/scc.podSecurityLabelSync=false --overwrite
oc label ns openstack pod-security.kubernetes.io/enforce=privileged --overwrite

# Verify LVMS (TopoLVM) capacity annotations on worker nodes
oc get nodes -l node-role.kubernetes.io/worker -o name | cut -d '/' -f 2 \
  | xargs | sed 's/ /,/g' | xargs -I{} oc get node -l "topology.topolvm.io/node in ({})" \
  -o=jsonpath='{.items[*].metadata.annotations.capacity\.topolvm\.io/local-storage}' | tr ' ' '\n'
```

---

## 1) Secondary networks (NetworkAttachmentDefinition)

```bash
# Apply and verify NetworkAttachmentDefinitions (NADs)
oc apply -f openstack-net-attach-def.yaml -n openstack
oc get net-attach-def -n openstack
```

---

## 1.5) Operator & CRD discovery (from training text)

```bash
# Log in to the cluster
oc login -u admin -p redhatocp https://api.ocp4.example.com:6443

# List installed operators (CSV = ClusterServiceVersion)
oc get csv -n openshift-operators
oc project openstack-operators
oc get csv

# CRDs related to OpenStack and field docs
oc get crd
oc get crd | grep "^openstack"
oc explain openstackclients
oc explain openstackcontrolplane.spec

# List OpenStack control plane CR (name and status), then full YAML
oc get openstackcontrolplane -n openstack
oc get openstackcontrolplane -n openstack -o yaml
```

---

## 2) MetalLB (VIPs: IPAddressPool & L2Advertisement)

```bash
# Create IP address pools for OpenStack services
oc apply -f openstack-ipaddresspools.yaml

# Inspect pools
oc describe -n metallb-system IPAddressPool

# Create and verify L2 advertisements
oc apply -f openstack-l2advertisement.yaml
oc get -n metallb-system L2Advertisement

# (OVN-Kubernetes only) Enable global IP forwarding for external traffic
oc get network.operator cluster --output=jsonpath='{.spec.defaultNetwork.type}'
oc patch network.operator cluster \
  --type=merge \
  -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"gatewayConfig":{"ipForwarding":"Global"}}}}}'
```

---

## 3) NetConfig (L3/VLAN/IPAM layout)

```bash
# Explore CRD schema and fields
oc describe crd netconfig
oc explain netconfig.spec

# Create and verify NetConfig for OpenStack
oc create -f openstack_netconfig.yaml -n openstack
oc get netconfig/openstacknetconfig -n openstack

# Sanity checks: resultant resources
oc get network-attachment-definitions -n openstack
oc get nncp  # (NMState NodeNetworkConfigurationPolicy)
```

---

## 4) Control plane (OpenStackControlPlane)

```bash
# Create OpenStack control plane
oc create -f openstack_control_plane.yaml -n openstack

# Watch/control-plane status and pods
oc get openstackcontrolplane -n openstack    # add -w to watch
oc get pods -n openstack

# Shell into the openstackclient pod when needed
oc rsh -n openstack openstackclient
```

---

## 4.1) Operator workloads inspection (openstack-operators project)

```bash
# Switch to operators project and list deployments
oc project openstack-operators
oc get deploy

# Inspect a specific operator deployment and confirm owner/labels
oc get deploy nova-operator-controller-manager -o yaml
oc describe deploy horizon-operator-controller-manager

# Show all resources owned by a given operator via label selector
oc get all -l openstack.org/operator-name=nova
oc get all -l openstack.org/operator-name=horizon
```

---

## 4.2) Service workloads inspection (openstack project)

```bash
# Switch to the OpenStack services project
oc project openstack

# List deployments (API, conductors, schedulers, etc.)
oc get deploy

# Describe a service deployment (example: Horizon)
oc describe deploy horizon

# List pods by a service label and tail logs
oc get pod -l service=horizon
oc logs pod/nova-api-0 --tail 2

# Watch events in the project (useful during deploy/restarts)
oc get events -n openstack
oc get events --sort-by='{.lastTimestamp}' -n openstack
```

---

## 5) Data plane secrets (registry, subscription, libvirt, SSH)

```bash
# Red Hat Subscription Manager (JSON inline)
oc create secret generic subscription-manager --from-literal \
  rhc_auth='{"login": {"username": "<subscription_manager_username>", "password": "<subscription_manager_password>"}}' -n openstack

# Registry credentials for pulling images
oc create secret generic redhat-registry --from-literal \
  edpm_container_registry_logins='{"registry.redhat.io": {"<username>":"<password>"}}' -n openstack

# (Optional) Libvirt client secret for migrations, etc.
oc apply -f secret_libvirt.yaml -n openstack

# Quick descriptions of useful secrets
oc describe secret dataplane-ansible-ssh-private-key-secret -n openstack
oc describe secret nova-migration-ssh-key -n openstack
oc describe secret subscription-manager -n openstack
oc describe secret redhat-registry -n openstack
oc describe secret libvirt-secret -n openstack
```

---

## 6) Data plane node sets

### a) Pre-provisioned nodes

```bash
# Create node set and wait for SetupReady
oc create --save-config -f openstack_preprovisioned_node_set.yaml -n openstack
oc wait openstackdataplanenodeset openstack-data-plane \
  -n openstack --for condition=SetupReady --timeout=10m

# Verify secrets and services
oc get secret -n openstack | grep openstack-data-plane
oc get openstackdataplaneservice -n openstack
```

### b) Unprovisioned nodes (BareMetalHost/Metal³)

```bash
# Create node set and wait
oc create --save-config -f openstack_unprovisioned_node_set.yaml -n openstack
oc wait openstackdataplanenodeset openstack-data-plane \
  -n openstack --for condition=SetupReady --timeout=10m

# Check secrets, BMH state, and services
oc get secret -n openstack | grep openstack-data-plane
oc get bmh -n openstack
oc get openstackdataplaneservice -n openstack
```

---

## 7) Data plane deployment runs (AnsibleEE)

```bash
# Launch a data plane deploy run
oc create -f openstack_data_plane_deploy.yaml -n openstack

# Watch the AnsibleEE pods and logs
oc get pod -l app=openstackansibleee -n openstack -w
oc logs -l app=openstackansibleee -n openstack -f --max-log-requests 10

# Check custom resource statuses
oc get openstackdataplanedeployment -n openstack
oc get openstackdataplanenodeset -n openstack

# From control plane: discover compute hosts in cells
oc rsh -n openstack nova-cell0-conductor-0 \
  nova-manage cell_v2 discover_hosts --verbose
```

---

## 8) Job / pod troubleshooting during data plane runs

```bash
# Follow a specific job (example: configure-network-...)
oc logs -f jobs/configure-network-<job-suffix> -n openstack | tail -n2

# Observe job condition messages (MESSAGE column)
oc get job -n openstack

# View service job logs quickly
oc logs job/<service> -n openstack

# List AnsibleEE pods, shell into one, or debug a non-running pod
oc get pods -l app=openstackansibleee -n openstack
oc rsh -n openstack <pod-name>
oc debug -n openstack <pod-name>
```

---

## 8.1) Operator reconciliation & resilience checks

```bash
# (In openstack-operators) find the operator controller pod and follow reconciliation logs
oc project openstack-operators
oc get pods | grep openstack-operator-controller
oc logs -f <openstack-operator-controller-pod> | grep Reconcile | grep Horizon

# (In openstack) delete a service pod and watch it be recreated by the operator
oc project openstack
oc delete pod <horizon-pod-name>
oc get pods -l service=horizon -w
```

---

## 9) Miscellaneous checks

```bash
# Confirm existing network policies in the openstack namespace
oc get networkpolicy -n openstack
```

---

## 9.1) Must‑gather for RHOSO

```bash
# Full RHOSO must‑gather (collects logs/CRs/events; may take ~10 minutes)
oc adm must-gather --image=quay.io/openstack-k8s-operators/openstack-must-gather

# Limit collection to specific services (NOTE the double dash before the variable)
oc adm must-gather --image=quay.io/openstack-k8s-operators/openstack-must-gather -- SOS_SERVICES=nova,glance gather

# Return to default project when finished
oc project default
```

---

### Notes
- Replace placeholders (e.g., `<username>`, `<password>`, `<pod-name>`, `<service>`, file names like `openstack_*`) with values from your environment.
- Add `-w` to any `oc get` command to watch changes live.
- Keep your current `oc project` set to `openstack` unless you intentionally operate in another namespace.
