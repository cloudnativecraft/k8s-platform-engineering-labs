# Kubernetes KIND Lab — Application Deployment and Troubleshooting

This repository contains a hands-on Kubernetes lab built with **KIND**. It uses a local multi-node cluster, an intentionally broken `Deployment`, and a `Service` that exposes the application inside the cluster.

The goal is not only to deploy an application, but also to learn a methodical approach to common Kubernetes issues: `Pending` Pods, a failing `readinessProbe`, environment-based configuration, a broken application update, and a rollback.

## Learning objectives

By completing this lab, you will learn how to:

- create a local multi-node Kubernetes cluster with KIND;
- work with namespaces and `kubectl` contexts;
- create a `ConfigMap` and a `Secret` with imperative commands;
- troubleshoot scheduling failures;
- use labels and `nodeSelector`;
- understand and fix a `readinessProbe`;
- expose a Deployment through a `ClusterIP` Service;
- inspect EndpointSlices;
- update a Deployment and roll it back;
- retrieve specific information with JSONPath.

## Repository structure

```text
.
├── kind-platform-lab.yaml
├── catalog-deployment.yaml
├── catalog-service.yaml
└── README.md
```

### `kind-platform-lab.yaml`

This configuration file is read by KIND rather than applied directly to Kubernetes. It defines:

- one `control-plane` node, which runs the Kubernetes control-plane components;
- two `worker` nodes, which are intended to run application workloads.

This setup provides a more realistic scheduling environment than a single-node cluster.

### `catalog-deployment.yaml`

This manifest defines the `catalog` Deployment. A Deployment represents the desired state of an application and manages its Pods through ReplicaSets.

It includes:

- three application replicas;
- an NGINX container image;
- the Pod label `app: catalog`;
- a `nodeSelector` requiring `workload=applications`;
- the `APP_COLOR` and `APP_ENV` variables, loaded from a ConfigMap and a Secret;
- an HTTP `readinessProbe` that determines whether a Pod may receive traffic.

The initial manifest intentionally contains two issues. None of the nodes initially has the label required by the `nodeSelector`, and the readiness probe requests an HTTP path that does not exist in the NGINX image.

### `catalog-service.yaml`

This manifest defines a `ClusterIP` Service named `catalog`.

The Service provides a stable internal address for the application. Its `app=catalog` selector targets Pods with the same label. Service port `80` forwards traffic to port `80` of the containers.

A Pod that is merely `Running` is not necessarily a usable backend. It must also match the Service selector and be considered `Ready`.

## 1. Create the cluster

Docker, KIND, and `kubectl` must be installed first.

```bash
docker version
kind version
kubectl version --client
```

Create and inspect the cluster:

```bash
kind create cluster --config kind-platform-lab.yaml
kubectl cluster-info --context kind-platform-lab
kubectl get nodes -o wide
```

When the KIND cluster is named `platform-lab`, its context is normally named `kind-platform-lab`.

## 2. Prepare the namespace

```bash
kubectl create namespace catalog
kubectl config set-context --current --namespace=catalog
```

The second command sets `catalog` as the default namespace for the current context. The following commands therefore do not need `-n catalog`.

```bash
kubectl config view --minify -o jsonpath='{..namespace}'
echo
```

Expected output:

```text
catalog
```

## 3. Create the ConfigMap and Secret

### ConfigMap

Preview the object without creating it:

```bash
kubectl create configmap catalog-config \
  --from-literal=APP_COLOR=blue \
  --dry-run=client -o yaml
```

Create it:

```bash
kubectl create configmap catalog-config \
  --from-literal=APP_COLOR=blue
```

Retrieve only the required value:

```bash
kubectl get configmap catalog-config \
  -o jsonpath='{.data.APP_COLOR}'
echo
```

Expected output: `blue`.

### Secret

Preview and create the Secret:

```bash
kubectl create secret generic catalog-secret \
  --from-literal=APP_ENV=production \
  --dry-run=client -o yaml

kubectl create secret generic catalog-secret \
  --from-literal=APP_ENV=production
```

The value in `.data.APP_ENV` is Base64-encoded. Base64 is encoding, not encryption.

```bash
kubectl get secret catalog-secret \
  -o jsonpath='{.data.APP_ENV}' | base64 -d
echo
```

Expected output: `production`.

To avoid printing a secret value in a shared terminal or CI log, check only that the key contains data:

```bash
kubectl get secret catalog-secret \
  -o jsonpath='{.data.APP_ENV}' | wc -c
```

## 4. Deploy the application

```bash
kubectl apply -f catalog-deployment.yaml
kubectl get deployment,replicaset,pods -o wide
```

At this point, the Pods should remain `Pending` because the scheduling constraint is intentionally unsatisfied.

## 5. Troubleshoot the Pending Pods

Collect evidence before modifying the Deployment or deleting Pods:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl get nodes --show-labels
kubectl get events --sort-by=.metadata.creationTimestamp
```

The Deployment requires:

```yaml
nodeSelector:
  workload: applications
```

A `nodeSelector` is a strict scheduling constraint. The Scheduler can place a Pod only on a node with the exact required label. Initially, neither worker has that label. The Pod events should therefore report that the available nodes do not match the Pod's node selector.

Label the workers without removing the workload constraint:

```bash
kubectl label node platform-lab-worker workload=applications
kubectl label node platform-lab-worker2 workload=applications
```

Verify the result:

```bash
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,WORKLOAD:.metadata.labels.workload'

kubectl get pods -o wide
```

All three Pods can now be scheduled across the two workers. Kubernetes does not guarantee equal distribution: `nodeSelector` allows placement on matching nodes but does not enforce balancing.

### `nodeSelector`, taints, and tolerations

- A label describes a node.
- `nodeSelector` requires specific labels and limits the Pod to matching nodes.
- A taint repels Pods that do not have a matching toleration.
- A toleration allows a Pod to pass a taint, but does not select or attract it to a node.

A generic toleration using `operator: Exists` is not required here. It would be overly broad because it could tolerate unrelated taints. The expected fix is to add `workload=applications` to the workers.

## 6. Troubleshoot Running but NotReady Pods

After scheduling, the containers start, but the `readinessProbe` requests a nonexistent path such as `/ready` or `/healthy`.

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get deployment catalog
kubectl rollout status deployment/catalog --timeout=20s
```

The relevant evidence is normally in the events shown by `kubectl describe`. The HTTP probe receives an unsuccessful response, typically `404`. The container remains running, but the Pod stays `NotReady`.

A failing `readinessProbe`:

- does not automatically restart the container;
- prevents the Pod from being treated as a ready Service endpoint;
- prevents the Deployment from reaching its expected availability.

This differs from a `livenessProbe`: repeated liveness failures can cause the kubelet to restart the container.

Apply a targeted fix:

```bash
kubectl patch deployment catalog --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/"}]'
```

Changing the Pod template creates a new Deployment revision and triggers a rolling update.

```bash
kubectl rollout status deployment/catalog
kubectl get pods
kubectl get deployment catalog
```

Expected status:

```text
READY   UP-TO-DATE   AVAILABLE
3/3     3            3
```

## 7. Expose the application

Apply the Service manifest:

```bash
kubectl apply -f catalog-service.yaml
```

The imperative equivalent is:

```bash
kubectl expose deployment catalog \
  --name=catalog \
  --type=ClusterIP \
  --port=80 \
  --target-port=80
```

Use only one method for the initial creation. Running both results in an `AlreadyExists` error.

Inspect the Service and EndpointSlices:

```bash
kubectl get service catalog
kubectl describe service catalog
kubectl get endpointslices \
  -l kubernetes.io/service-name=catalog
```

Test from inside the cluster:

```bash
kubectl run curl-test \
  --image=curlimages/curl:8.12.1 \
  --restart=Never \
  --rm -it \
  -- curl -sS http://catalog
```

The short DNS name `catalog` works because the test Pod and Service are in the same namespace. From another namespace, use `catalog.catalog` or `catalog.catalog.svc.cluster.local`.

## 8. Inspect the environment variables

Retrieve a Pod name dynamically and inspect its environment:

```bash
POD=$(kubectl get pods -l app=catalog \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec "$POD" -- sh -c 'env | grep "^APP_"'
```

Expected output:

```text
APP_COLOR=blue
APP_ENV=production
```

Update the ConfigMap declaratively from an imperative command:

```bash
kubectl create configmap catalog-config \
  --from-literal=APP_COLOR=green \
  --dry-run=client -o yaml | kubectl apply -f -
```

Environment variables in existing containers do not change. They are injected when containers are created. Restart the Deployment progressively:

```bash
kubectl rollout restart deployment/catalog
kubectl rollout status deployment/catalog
```

Retrieve a new Pod and verify the value:

```bash
POD=$(kubectl get pods -l app=catalog \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec "$POD" -- printenv APP_COLOR
```

Expected output: `green`.

## 9. Simulate a failed update and roll it back

Set an image that does not exist:

```bash
kubectl set image deployment/catalog \
  catalog=nginx:version-inexistante
```

Investigate the failed rollout:

```bash
kubectl rollout status deployment/catalog --timeout=30s
kubectl get pods
kubectl describe pod <new-pod-name>
kubectl rollout history deployment/catalog
```

After repeated image-pull attempts, the new Pod should normally enter `ImagePullBackOff`. During a rolling update, old ready Pods may remain available. A failed new version therefore does not necessarily cause an immediate Service outage.

Roll back:

```bash
kubectl rollout undo deployment/catalog
kubectl rollout status deployment/catalog
```

A command such as `--to-revision=9` is valid only after the rollout history confirms that revision 9 is the intended target.

```bash
kubectl rollout history deployment/catalog
kubectl get deployment,replicaset,pods
kubectl get endpointslices \
  -l kubernetes.io/service-name=catalog
```

## 10. Imperative commands and JSONPath challenge

Scale to five replicas:

```bash
kubectl scale deployment/catalog --replicas=5
kubectl rollout status deployment/catalog
```

Display only Pod names and their nodes with JSONPath:

```bash
kubectl get pods -l app=catalog \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
```

The `custom-columns` version is also correct and often easier to read:

```bash
kubectl get pods -l app=catalog \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName'
```

Display only the container image:

```bash
kubectl get deployment catalog \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
```

Scale back to three replicas:

```bash
kubectl scale deployment/catalog --replicas=3
kubectl rollout status deployment/catalog
```

Count the ready Service endpoints:

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=catalog \
  -o jsonpath='{range .items[*].endpoints[?(@.conditions.ready==true)]}{.addresses[0]}{"\n"}{end}' \
  | wc -l
```

Expected output: `3`.

## Troubleshooting method

For each incident:

1. observe the current state with `kubectl get`;
2. inspect details and events with `kubectl describe`;
3. inspect logs when the container has actually started;
4. identify the failing constraint or configuration;
5. apply the smallest targeted fix;
6. verify the rollout, Pods, and endpoints.

## Lab assessment

### What was completed correctly

- Created a three-node KIND cluster.
- Created and selected the `catalog` namespace.
- Created the ConfigMap and Secret imperatively after previewing them with `--dry-run=client`.
- Retrieved and decoded a Secret value with JSONPath.
- Used `describe`, events, and labels to investigate `Pending` Pods.
- Added the required label to the workers instead of immediately deleting Pods.
- Identified the failing `readinessProbe` and changed its path to `/`.
- Inspected the Service and EndpointSlices.
- Confirmed that a ConfigMap change does not update environment variables in existing processes.
- Used Deployment rollout and rollback mechanisms.
- Used `scale`, `custom-columns`, and JSONPath to retrieve targeted information.

### Points to correct or clarify

- `kubernetes.io/os=linux` does not solve the original issue when every node already has that label. The relevant constraint is `workload=applications`.
- A toleration does not match a Pod to a node. It only allows the Pod to tolerate a taint.
- The workload `nodeSelector` should not be removed. Make the workers eligible by adding the correct label.
- Three Pods are not necessarily spread across three nodes. With two labelled workers, several Pods may run on the same worker.
- The kubelet executes readiness probes; “the cluster runs the probe” is too imprecise.
- This probe failure is not a parsing error. NGINX serves `/`, but not the incorrectly configured path.
- A `NotReady` Pod may appear in an EndpointSlice with `conditions.ready=false`, but it is not normally used as a ready backend.
- Every `apply`, `patch`, `scale`, `restart`, or `undo` should be followed by verification.
- Revision numbers are not fixed. Inspect `kubectl rollout history` before using `--to-revision`.

## Key concepts

- The Scheduler normally explains placement failures in Pod events.
- `nodeSelector` enforces a strict node-label requirement.
- Tolerations and node selectors solve different scheduling problems.
- `Running` describes the Pod execution phase; `Ready` indicates whether it should receive traffic.
- A readiness probe controls traffic eligibility, whereas a liveness probe may trigger a restart.
- A Service selects Pods by label and relies on EndpointSlices.
- Environment variables from a ConfigMap or Secret are fixed when the container starts.
- A Deployment retains ReplicaSets, enabling rollbacks.
- A command is not complete until its effect has been verified.

## Cleanup

```bash
kind delete cluster --name platform-lab
```

This removes the entire KIND cluster and every Kubernetes object created during the lab.
