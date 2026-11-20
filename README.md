# VKS + Argo CD MVP: one namespace, one cluster, Contour with a static IP and a wildcard certificate

**Goal:** vSphere Namespace `system-1-ns-poc-1` holds cluster `wrkld-system-1-poc-1`. VKS installs Contour
on it, with Envoy on a **static load-balancer IP**, and the team's **wildcard certificate** can be used by
every namespace through Contour's TLS certificate delegation.

**Tools:** Kustomize only, no Helm. One shared **base** per add-on, plus a small **overlay** per cluster
that says which cluster and which IP. All of it is plain YAML you can preview with `kubectl kustomize`.

```
setup/                                        applied by hand: kubectl apply -k setup/
  argocd-instance.yaml                        Argo CD instance
  namespaces/system-1-ns-poc-1.yaml           what Argo CD may do in that namespace + its registration
  projects.yaml                               vks-clusters (Supervisor side), guest-config (inside clusters)
  appsets.yaml                                one app per cluster folder, one per guest-config folder
  guest-clusters/wrkld-system-1-poc-1.yaml    registers the cluster itself (apply once it's AVAILABLE)
addons/                                       BASES, shared by every cluster
  cert-manager/                               required by Contour
  contour/                                    Envoy as type LoadBalancer
clusters/system-1-ns-poc-1/wrkld-system-1-poc-1/
  kustomization.yaml                          cluster.yaml + addons/
  cluster.yaml                                the cluster (unchanged)
  addons/kustomization.yaml                   OVERLAY: names, target cluster
  addons/contour-static-ip.yaml               OVERLAY: this cluster's static IP
guest-config/system-1-ns-poc-1/wrkld-system-1-poc-1/
  ingress-certs.yaml                          namespace + TLSCertificateDelegation (inside the cluster)
```
`clusters/.../wrkld-system-1-poc-2` and `system-1-ns-poc-2/...` are untouched and keep working as before.

## How the base and overlay fit together
```
addons/contour/                 (base: same for every cluster)
  AddonConfig  contour          envoy.service.type: LoadBalancer, loadBalancerIP: ""
  AddonInstall contour          target: CLUSTER-NAME
        |
        v   clusters/<ns>/<cluster>/addons/kustomization.yaml  (overlay: only what's different)
            namePrefix: <cluster>-               -> wrkld-system-1-poc-1-contour
            patch: target-cluster                -> cluster-name: wrkld-system-1-poc-1
            patch: contour-static-ip.yaml        -> loadBalancerIP: 10.0.0.50
        v
  what Argo CD applies on the Supervisor, in namespace system-1-ns-poc-1
```
See the result before pushing (kubectl has Kustomize built in):
```sh
kubectl kustomize clusters/system-1-ns-poc-1/wrkld-system-1-poc-1
```

## Order of events (sync waves)
| Where | Wave | Object | What happens |
|---|---|---|---|
| Supervisor | 0 | Cluster | VKS builds (or keeps) the cluster |
| Supervisor | 1 | AddonInstall cert-manager | starts once the cluster is Healthy |
| Supervisor | 2 | AddonConfig contour | Contour's settings: LoadBalancer + static IP |
| Supervisor | 3 | AddonInstall contour | VKS installs Contour; Envoy gets the static IP |
| Guest cluster | 0 | Namespace ingress-certs | after you register the cluster |
| Guest cluster | 1 | TLSCertificateDelegation | waits (retrying) until Contour is installed |

Argo CD marks an add-on Healthy as soon as its object exists. The real install state is in
`kubectl get clusteraddon -n system-1-ns-poc-1`.

---

## Before you start: get the IP and the certificate
1. **Static IP.** Pick a free IP the load balancer can hand out:
   - NSX Advanced Load Balancer: from its IPAM pool.
   - NSX (default): from the load balancer IP pool.
   - NSX (manual): from a floating IP pool **outside** that range, managed in NSX Manager.

   Make sure nothing else uses it.
2. **DNS.** Point the wildcard record (e.g. `*.apps.example.com`) at that IP.
3. **Wildcard certificate.** Get `wildcard.crt` (full chain) and `wildcard.key`. They never go into Git.

## Step 1: Contour on the static IP
1. Put your IP in `clusters/system-1-ns-poc-1/wrkld-system-1-poc-1/addons/contour-static-ip.yaml`.
2. Check the result: `kubectl kustomize clusters/system-1-ns-poc-1/wrkld-system-1-poc-1`
   (expect the Cluster, 2 AddonInstalls and 1 AddonConfig with your IP, and no `CLUSTER-NAME` left).
3. Replace the old `setup/` files with this `setup/` folder, push to `main`, then:
   ```sh
   kubectl apply -k setup/
   ```
   Same object names as before, so this updates them in place. The project now allows add-on objects,
   and the new `guest-config` project and ApplicationSet are added.
4. Watch, on the Supervisor:
   ```sh
   kubectl get addoninstall,addonconfig -n system-1-ns-poc-1
   kubectl get clusteraddon -n system-1-ns-poc-1
   ```
5. Check the IP, inside the guest cluster:
   ```sh
   kubectl get secret wrkld-system-1-poc-1-kubeconfig -n system-1-ns-poc-1 -o jsonpath='{.data.value}' | base64 -d > ~/.kube/poc-1.kubeconfig
   kubectl --kubeconfig ~/.kube/poc-1.kubeconfig get svc envoy -n tanzu-system-ingress
   ```
   **Expect:** `TYPE LoadBalancer`, `EXTERNAL-IP` = your static IP.

## Step 2: wildcard certificate delegation
1. Register the cluster with Argo CD (it's AVAILABLE already):
   ```sh
   kubectl apply -f setup/guest-clusters/wrkld-system-1-poc-1.yaml
   kubectl get managedentity -n system-1-argocd        # vks-system-1-ns-poc-1-wrkld-system-1-poc-1: Ready
   ```
2. The `wrkld-system-1-poc-1-config` app creates namespace `ingress-certs` and the delegation.
3. Add the certificate (once, in the guest cluster):
   ```sh
   kubectl --kubeconfig ~/.kube/poc-1.kubeconfig create secret tls wildcard-tls -n ingress-certs \
     --cert=wildcard.crt --key=wildcard.key
   ```
4. Any team can now use it from its own namespace:
   ```yaml
   apiVersion: projectcontour.io/v1
   kind: HTTPProxy
   metadata:
     name: my-app
     namespace: my-team
   spec:
     virtualhost:
       fqdn: my-app.apps.example.com
       tls:
         secretName: ingress-certs/wildcard-tls     # <namespace>/<secret>, allowed by the delegation
     routes:
     - services:
       - name: my-app
         port: 80
   ```
   `curl -v https://my-app.apps.example.com` should show the wildcard certificate.

---

## Next cluster (same pattern)
1. Copy the cluster folder to `clusters/<ns>/<new-cluster>/`. Replace the old cluster name with the new one in
   every file, and put that cluster's IP in `contour-static-ip.yaml`.
2. Copy `guest-config/<ns>/wrkld-system-1-poc-1/` to `guest-config/<ns>/<new-cluster>/` (no edits needed).
3. Copy `setup/guest-clusters/wrkld-system-1-poc-1.yaml` with the new name. Add
   `<new-cluster>-<ns>` (namespace `ingress-certs`) to the `guest-config` project's destinations,
   and run `kubectl apply -k setup/`.
4. Push. Once the cluster is AVAILABLE, apply its guest-cluster file and create its `wildcard-tls` secret.

**New vSphere Namespace:** create it in vCenter, then copy `setup/namespaces/system-1-ns-poc-1.yaml` with the new
name, list it in `setup/kustomization.yaml`, add it to the `vks-clusters` project, and run `kubectl apply -k setup/`.

## Security notes
- The certificate's private key stays in one namespace in each cluster. Teams reference it; they can't read it.
- Contour delegates to every namespace (`"*"`). List specific namespaces in `ingress-certs.yaml` to restrict it.
- With static IPs, anyone who can patch `services/status` in a guest cluster could hijack traffic. Don't give
  developers that permission. Check with `kubectl auth can-i patch service/status`.
- Argo CD gets access to a vSphere Namespace or guest cluster only through files in `setup/`, applied by a person.

## Troubleshooting
| Symptom | Fix |
|---|---|
| Envoy `EXTERNAL-IP` stays `<pending>` | IP not free or not in the right pool; check `kubectl describe svc envoy -n tanzu-system-ingress` |
| Envoy is `NodePort` | AddonConfig not picked up: name must be `<cluster>-contour`; check `kubectl get addonconfig -n <ns>` |
| Add-on sync `forbidden` | `setup/namespaces/<ns>.yaml` not applied |
| `...-config` app: cluster not found | Guest cluster not registered (Step 2.1) |
| `...-config` app: no matches for TLSCertificateDelegation | Contour not installed yet; it retries on its own |
| HTTPProxy shows `invalid` / certificate not delegated | Secret name or namespace differs from `ingress-certs/wildcard-tls` |
