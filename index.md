---
title: 'CKA practice exam scenario: Configure Network Policies for Selective Pod Communication'

description: |
  This exercise tests your ability to configure Kubernetes Network Policies to enforce
  specific communication patterns between pods in different deployments. You'll need to
  create deployments and configure network policies to allow selective traffic flow. Have fun!

kind: challenge

playground: k3s

playgroundOptions:
  tabs:
  - machine: dev-machine
  - machine: cplane-01
  - machine: node-01

  machines:
    - name: dev-machine
      users:
      - name: root
        default: true
        welcome: 'Welcome to the CKA networking exercise. Follow the instructions to configure secure pod communication.'
    - name: cplane-01
    - name: node-01

cover: __static__/pod-networking.png

createdAt: 2024-11-03
updatedAt: 2024-11-03

difficulty: medium

categories:
  - kubernetes
  - networking

tagz:
  - cka
  - kubernetes
  - network-policies

tasks:
  verify_deployments:
    run: |
      if [ $(kubectl get deployment -n app frontend -o jsonpath='{.status.replicas}') -eq 2 ] && \
         [ $(kubectl get deployment -n app backend -o jsonpath='{.status.replicas}') -eq 2 ] && \
        echo "Deployments created successfully!"
        exit 0
      else
        exit 1
      fi
  verify_labels:
    run: |
      if [ $(kubectl get pods -n app -l role=frontend --no-headers | wc -l) -eq 2 ] && \
         [ $(kubectl get pods -n app -l tier=api --no-headers | wc -l) -eq 2 ] && \
         [ $(kubectl get pods -n app -l "tier=api,!role" --no-headers | wc -l) -eq 1 ]; then
        echo "Pod labels configured correctly!"
        exit 0
      else
        exit 1
      fi
  verify_network_policies:
    run: |
      # Get a frontend pod and the backend pod with role=backend

      FRONTEND_POD=$(kubectl get pod -n app -l role=frontend -o jsonpath='{.items[0].metadata.name}')
      FRONTEND_POD_IP=$(kubectl get pod -n app -l role=frontend -o jsonpath='{items[0].status.podIP}')
      BACKEND_POD_WITH_ROLE=$(kubectl get pod -n app -l role=backend -o jsonpath='{.items[0].metadata.name}')
      BACKEND_POD_WITH_ROLE_IP=$(kubectl get pod -n app -l role=backend -o jsonpath='{items.[0].status.podIP}')
      BACKEND_POD_WITHOUT_ROLE=$(kubectl get pod -n app -l "tier=api,!role" -o jsonpath='{.items[0].metadata.name}')
      BACKEND_POD_WITHOUT_ROLE_IP=$(kubectl get pod -n app -l "tier=api,!role" -o jsonpath='{.items[0].status.podIP}')
      
      # Test connectivity from frontend to both backend pods
      FRONTEND_TO_BACKEND=$(kubectl -n app exec $FRONTEND_POD -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 $BACKEND_POD_WITH_ROLE_IP:8000)
      FRONTEND_TO_BACKEND=$(kubectl -n app exec $FRONTEND_POD -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 $BACKEND_POD_WITHOUT_ROLE_IP:8000)
      
      # Test connectivity from both types of backend pods to frontend pod
      BACKEND_WITH_ROLE_TO_FRONTEND=$(kubectl -n app exec $BACKEND_POD_WITH_ROLE -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 $FRONTEND_POD_IP:80)
      BACKEND_WITHOUT_ROLE_TO_FRONTEND=$(kubectl -n app exec $BACKEND_POD_WITHOUT_ROLE -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 $FRONTEND_POD_IP:80 || echo "timeout")
      
      if [ "$FRONTEND_TO_BACKEND" -eq 200 ] && \
         [ "$BACKEND_WITH_ROLE_TO_FRONTEND" -eq 200 ] && \
         [ "$BACKEND_WITHOUT_ROLE_TO_FRONTEND" = "timeout" ]; then
        echo "Network policies configured correctly!"
        exit 0
      else
        exit 1
      fi

---

In this exercise, you will configure network policies to control traffic flow between two deployments in a Kubernetes cluster. You'll need to ensure specific pods can communicate based on their labels while blocking unauthorized traffic.

<img src="__static__/pod-networking.png" style="margin: 0px auto; max-width: 600px; width: 100%" alt="Diagram showing desired network policy configuration between frontend and backend pods">

First, create a namespace called "app", create two deployments:
1. Frontend:
   - Deployment "frontend" with 2 replicas running nginx:1.20
2. Backend:
   - Deployment "backend" with 2 replicas running leskis/default-go

::simple-task
---
:tasks: tasks
:name: verify_deployments
---
#active
Waiting for deployments to be created...

#completed
Great! Both deployments are running correctly.
::

::hint-box
---
:summary: Hint 1
---
```bash
# Create namespace
kubectl create namespace app

# Create deployments
kubectl create deployment app -n app --image=nginx:1.20 --replicas=2
kubectl create deployment backend -n app --image=leskis/default-go --replicas=2
```
::

Now, label the pods:
1. All frontend pods should have: role=frontend
2. All backend pods should have: tier=api
3. Only one backend pod should have the additional label: role=backend

::simple-task
---
:tasks: tasks
:name: verify_labels
---
#active
Checking for correct pod labels...

#completed
Perfect! All pods are properly labeled.
::

::hint-box
---
:summary: Hint 2
---
```bash
# Label all frontend pods
kubectl label pods -n app -l app=app role=frontend

# Label all backend pods with tier=api
kubectl label pods -n app -l app=backend tier=api

# Get the name of one backend pod
BACKEND_POD=$(kubectl get pods -n app -l app=backend -o jsonpath='{.items[0].metadata.name}')

# Add role=backend label to just that one pod
kubectl label pod -n app $BACKEND_POD role=backend
```
::

Finally, create network policies to ensure:
1. All frontend pods can send traffic to any backend pod with label tier=api on port 8000
2. Only the backend pod with label role=backend can send traffic to frontend pods with label role=frontend on port 80
3. All other traffic should be denied by default

::simple-task
---
:tasks: tasks
:name: verify_network_policies
---
#active
Verifying network policies and testing connectivity...

#completed
Excellent! The network policies are correctly configured and enforcing the desired traffic patterns.
::

::hint-box
---
:summary: Hint 3
---
Create two network policies:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: app
spec:
  podSelector:
    matchLabels:
      tier: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - port: 8000
      protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-frontend
  namespace: app
spec:
  podSelector:
    matchLabels:
      role: frontend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - port: 80
      protocol: TCP
```
::
