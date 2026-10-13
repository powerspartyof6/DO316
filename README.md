# DO316 practice: Cat GIF VM on OpenShift Virtualization

A small lab app for studying DO316 / EX316. A Fedora `VirtualMachine` runs httpd
and serves cat GIFs. A `Service` and `Route` expose it so you can pull GIFs from
any terminal.

```
catgifs/
  namespace.yaml        Namespace "catgifs"
  vm.yaml               VirtualMachine (instancetype u1.small, Fedora DataSource, cloud-init)
  service.yaml          ClusterIP Service selecting the virt-launcher pod
  route.yaml            Edge-terminated Route
  kustomization.yaml    Applies the four files above
  extras/
    snapshot.yaml       VirtualMachineSnapshot practice
    restore.yaml        VirtualMachineRestore practice
    migration.yaml      VirtualMachineInstanceMigration practice
```

## Prerequisites

- OpenShift with the OpenShift Virtualization operator installed
- The `fedora` boot source available:
  ```bash
  oc get datasource fedora -n openshift-virtualization-os-images
  ```
- Outbound internet from the VM (the GIFs come from https://cataas.com at first boot)
- `oc` logged in with rights to create a namespace; `virtctl` optional

## Deploy

```bash
git clone https://github.com/powerspartyof6/DO316.git
cd DO316
oc apply -k catgifs/
```

Or with no clone:

```bash
oc apply -k "https://github.com/powerspartyof6/DO316/catgifs?ref=main"
```

Watch it come up (the disk clone and first boot take a few minutes):

```bash
oc get dv,vm,vmi -n catgifs -w
```

## Get cats

```bash
CATS=$(oc get route catgifs -n catgifs -o jsonpath='{.spec.host}')
curl -s "http://$CATS/cats.txt"                  # list of GIFs
curl -s "http://$CATS/cat.gif" -o cat.gif        # random cat, changes every minute
```

Open `https://$CATS/` in a browser for the gallery. If the cluster uses a
self-signed router certificate, add `-k` to HTTPS curl calls.

To show a GIF right in the terminal, install `chafa` or `viu`:

```bash
curl -s "http://$CATS/cat.gif" | chafa -
```

## Get into the VM

```bash
virtctl console catgif-vm -n catgifs     # login: fedora / catgifs
```

The password is for classroom use only. Change it in `vm.yaml`, or swap it for
`ssh_authorized_keys` and use `virtctl ssh fedora@vm/catgif-vm -n catgifs`.

## DO316 practice ideas

| Topic | Try |
|---|---|
| Lifecycle | `virtctl stop/start/restart/pause/unpause catgif-vm -n catgifs` |
| Run strategy | Change `runStrategy` to `Halted`, `Manual`, `RerunOnFailure` and compare |
| Snapshots | `oc apply -f catgifs/extras/snapshot.yaml`, stop the VM, then `restore.yaml` |
| Live migration | `oc create -f catgifs/extras/migration.yaml` (needs an RWX disk) |
| Node maintenance | `oc adm cordon` / `oc adm drain` the VM's node and watch it move |
| Storage | Hot-plug a disk: `virtctl addvolume catgif-vm --volume-name=<pvc> -n catgifs` |
| Networking | Add a Multus `NetworkAttachmentDefinition` and a second NIC |
| Exposure | Swap the Route for `virtctl expose vm catgif-vm --port=80 --type=NodePort` |
| Health | Add a `readinessProbe` (httpGet port 80) to the VM template |

## Clean up

```bash
oc delete -k catgifs/
```

Cat GIFs come from [Cat as a Service](https://cataas.com). This repo does not
include or redistribute any images.
