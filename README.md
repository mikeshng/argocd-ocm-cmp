# argocd-cmp-maestro
ArgoCD and Maestro Integration using ArgoCD CMP.

This is a PoC of a PoC of a PoC.

# Overview
ArgoCD guestbook Application reconciles => The target Git folder is: https://github.com/argoproj/argocd-example-apps/tree/master/guestbook
=> The CMP plugin looks at all the yaml manifests in that folder => wrap them inside an AnsibleJob
=> AnsibleJob is created instead of the original Git repo manifests => AnsibleJob reconciles
=> AnsibleJob calls Ansible/AWX with the Maestro job template passing in the workload
=> Ansible/AWX receives the manifests workload => base64 decode it and call Maestro server appropriately.

# Quick Start

```
git clone https://github.com/mikeshng/argocd-cmp-maestro.git
cd argocd-cmp-maestro
kind delete cluster
kind create cluster
kind export kubeconfig
kubectl create ns argocd
kubectl apply -f ansiblejob_crd.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.11.2/manifests/core-install.yaml
kubectl apply -f proj.yaml 
kubectl apply -f cm_plugin.yaml 
kubectl apply -f deploy_cmp.yaml 
kubectl wait --for=condition=available deployment/argocd-repo-server -n argocd --timeout=300s
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-repo-server --namespace argocd
kubectl apply -f app.yaml
sleep 10s
kubectl -n guestbook get ansiblejob guestbook -o yaml
kubectl -n guestbook get ansiblejob guestbook -o json | jq -r '.spec.extra_vars.work_manifests' | base64 -d
# done
```

## Expected Output

```
$ kubectl -n guestbook get ansiblejob guestbook -o yaml
apiVersion: tower.ansible.com/v1alpha1
kind: AnsibleJob
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
...
  creationTimestamp: "2024-05-28T04:11:04Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: guestbook
  name: guestbook
  namespace: guestbook
  resourceVersion: "709"
  uid: b0795f46-73fb-47e9-8672-8c7d04d7e523
spec:
  connection_secret: awxaccess
  extra_vars:
    target_cluster: cluster1
    work_manifests: YXBpVmVyc2lvbjogYXB...
  inventory: Demo Inventory
  job_template_name: Demo Maestro Template

$ kubectl -n guestbook get ansiblejob guestbook -o json | jq -r '.spec.extra_vars.work_manifests' | base64 -d
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-ui
spec:
  replicas: 1
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
    spec:
      containers:
      - image: gcr.io/heptio-images/ks-guestbook-demo:0.2
        name: guestbook-ui
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: guestbook-ui
```
