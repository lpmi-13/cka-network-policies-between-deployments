---
title: 'CKA Practice: Configure Network Policies To Restrict Traffic Between Pods'

description: |
  This exercise tests your ability to configure kubernetes network nolicies to make sure only pods with specific labels can communicate with each other.

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

cover: __static__/pod-networking-with-labels.png

createdAt: 2024-11-03
updatedAt: 2024-12-10

difficulty: easy

categories:
  - kubernetes
  - networking

tagz:
  - cka
  - kubernetes
  - network-policies

tasks:
  verify_namespace:
    run: |
      if [ $(kubectl get namespace | grep app | wc -l) -gt 0 ]; then
        echo "Namespace created!"
        exit 0
      else
        exit 1
      fi
  verify_deployment_frontend:
    run: |
      if [ $(kubectl get deployment -n app frontend -o jsonpath='{.status.replicas}') -eq 2 ]; then
         echo "Frontend deployment created successfully!"
         exit 0
      else
         exit 1
      fi
  verify_deployment_backend:
    run: |
      if [ $(kubectl get deployment -n app backend -o jsonpath='{.status.replicas}') -eq 2 ]; then
         echo "Backend deploymenet created successfully!"
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
    # we don't want this to start firing too soon
    needs:
      - verify_labels
    # default timeout is 10-15 seconds, and we want the checking loop to run faster than that
    timeout_seconds: 7
    run: |
      # Get a frontend pod and one backend pod with role=backend and one backend pod without a role
      FRONTEND_POD=$(kubectl get pod -n app -l role=frontend -o jsonpath='{.items[0].metadata.name}')
      FRONTEND_POD_IP=$(kubectl get pod -n app -l role=frontend -o jsonpath='{.items[0].status.podIP}')

      BACKEND_POD_WITH_ROLE=$(kubectl get pod -n app -l role=backend -o jsonpath='{.items[0].metadata.name}')
      BACKEND_POD_WITH_ROLE_IP=$(kubectl get pod -n app -l role=backend -o jsonpath='{.items[0].status.podIP}')

      BACKEND_POD_WITHOUT_ROLE=$(kubectl get pod -n app -l "tier=api,role!=backend" -o jsonpath='{.items[0].metadata.name}')
      BACKEND_POD_WITHOUT_ROLE_IP=$(kubectl get pod -n app -l "tier=api,role!=backend" -o jsonpath='{.items[0].status.podIP}')
      
      # Test connectivity from frontend to both backend pods
      FRONTEND_TO_BACKEND=$(kubectl -n app exec $FRONTEND_POD -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 $BACKEND_POD_WITH_ROLE_IP:8000)
      FRONTEND_TO_BACKEND_WITHOUT_ROLE=$(kubectl -n app exec $FRONTEND_POD -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 $BACKEND_POD_WITHOUT_ROLE_IP:8000)
      
      # Test connectivity from both types of backend pods to frontend pod
      BACKEND_WITH_ROLE_TO_FRONTEND=$(kubectl -n app exec $BACKEND_POD_WITH_ROLE -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 $FRONTEND_POD_IP:80)

      BACKEND_WITHOUT_ROLE_TO_FRONTEND=$(kubectl exec -n app $BACKEND_POD_WITHOUT_ROLE -- curl -s --connect-timeout 5 $FRONTEND_POD_IP:80 2>&1 || true)
      
      if [ "$FRONTEND_TO_BACKEND" -eq 200 ] && \
         [ "$FRONTEND_TO_BACKEND_WITHOUT_ROLE" -eq 200 ] && \
         [ "$BACKEND_WITH_ROLE_TO_FRONTEND" -eq 200 ] && \
         [[ "$BACKEND_WITHOUT_ROLE_TO_FRONTEND" == *"command terminated"* ]]; then
         echo "Network policies configured correctly!"
         exit 0
      else
        exit 1
      fi

---

In this exercise, you will configure network policies to control traffic flow between two deployments in a Kubernetes cluster. You'll need to ensure specific pods can communicate based on their labels while blocking unauthorized traffic. Have fun!

<img src="__static__/pod-networking-with-labels.png" style="margin: 0px auto; max-width: 600px; width: 100%" alt="Diagram showing desired network policy configuration between frontend and backend pods">

First, create a namespace called "app":

::simple-task
---
:tasks: tasks
:name: verify_namespace
---
#active
Waiting for namespace to be created...

#completed
Good, the namespace is ready to go.
::


::hint-box
---
:summary: Hint 1
---
Check the [documentation for creating namespaces](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-namespace-em-)
::

Now, create two deployments in that namespace.


1. Frontend:
   - Deployment "frontend" with 2 replicas running `nginx:1.20`
   - Accessible on port 80
2. Backend:
   - Deployment "backend" with 2 replicas running `leskis/default-go`
   - Accessible on port 8000

::simple-task
---
:tasks: tasks
:name: verify_deployment_frontend
---
#active
Waiting for frontend deployment to be created...

#completed
Great! Frontend deployment is running correctly.
::

::simple-task
---
:tasks: tasks
:name: verify_deployment_backend
---
#active
Waiting for backend deployment to be created...

#completed
Great! Backend deployment is running correctly.
::

::hint-box
---
:summary: Hint 2
---
Check the [documentation for creating deployments](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-deployment-em-)
::

Now, label the pods:
1. All frontend pods should have: `role=frontend`
2. All backend pods should have: `tier=api`
3. Only one backend pod should have the additional label: `role=backend`

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
:summary: Hint 3
---
Check the [documentation for adding labels](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#label)
::

::hint-box
---
:summary: Hint 4
---
```bash
# Get the name of one backend pod
BACKEND_POD=$(kubectl get pods -n app -l app=backend -o jsonpath='{.items[0].metadata.name}')
```
::

Finally, create network policies to make sure:
1. All frontend pods can send traffic to any backend pod with label `tier=api` on port 8000
2. Only the backend pod with label `role=backend` can send traffic to frontend pods with label `role=frontend` on port 80
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
:summary: Hint 5
---
Create two network policies, one for the frontend => backend, and another from the backend => frontend. Here's the first one:
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
```
::
