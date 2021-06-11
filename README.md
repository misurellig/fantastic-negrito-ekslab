# Fantastic Negrito EKS lab

Fantastic Negrito EKS lab is composed by a series of steps to keep track of Kubernetes interaction within a EKS cluster environment.

## Prerequisites

Install the following utils

  * [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
  * [kubectl](https://kubernetes.io/docs/tasks/tools/)
  * [eksctl](https://eksctl.io/)
  * [helm v3](https://helm.sh/)

## Install EKS cluster

Before proceeding be sure to use a AWS IAM principal with the required permission as specified in the [Required IAM permissions](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html#eksctl-prereqs) doc.



Use the ``eksctl-config.yaml`` file provided in the repo to isntall the EKS cluster with the following command:

```bash
eksctl create cluster -f eksctl-config.yaml
```

The eksctl utility will create a VPC with public and private subnets as well as the fantastic-negrito EKS cluster with its [managed node group](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html).

## Deploy a static web site

We’ll be using the [bitnami charts repository](https://github.com/bitnami/charts) to deploy Apache Helm chart.

The chart will be deployed into the ``ebserver-ns`` namaspece so let's create it.

```bash
# Create a namespace webserver-ns
kubectl create namespace webserver-ns
```

Static website will be deployed from the [hello-webapge](https://github.com/misurellig/hello-webpage) GitHub repo. Before deploying the bitnami chart we need to add the bitnami Helm Charts Repository.

```bash
# Add the bitnami Helm Charts Repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Deploy WordPress in its own namespace
helm -n webserver-ns \
    install apache bitnami/apache \
    --set service.type=LoadBalancer \
    --set cloneHtdocsFromGit.enabled=true \
    --set cloneHtdocsFromGit.repository="https://github.com/misurellig/hello-webpage.git" \
    --set cloneHtdocsFromGit.branch="master"
```

## Setup deployment user

We'll create a k8s deploy-user with permissions to only update deployment into namespace webserver-ns. Authentication will occur by mapping this user to a given AWS IAM user (named deploy-user) used by leveraging the [AWS IAM Authenticator for Kubernetes](https://github.com/kubernetes-sigs/aws-iam-authenticator) utility.

```bash
# Get the existing ConfigMap and save into a file called aws-auth.yaml
kubectl get configmap -n kube-system aws-auth -o yaml | grep -v "creationTimestamp\|resourceVersion\|selfLink\|uid" | sed '/^  annotations:/,+2 d' > aws-auth.yaml

# Append the deploy-user mapping to the existing configMap
cat << EoF >> aws-auth.yaml
data:
  mapRoles: |
    - rolearn: arn:aws:iam::111111111111:role/DeployRole # Change here with proper ARN
      username: deploy-user
EoF
```

Verify the file has been properly updated

```bash
cat aws-auth.yaml
```

Apply the ConfigMap to apply this mapping to the system

```bash
kubectl apply -f aws-auth.yaml
```

We'll create now the k8s webserver-deploy role and its binding. This role provides list, get, and watch access for pods and deployments, but only for the webserver-ns namespace.

```bash
cat << EoF > webserver-deploy-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: webserver-ns
  name: update-deploy
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods", "secrets", "services"]
  verbs: ["create", "list", "get", "watch", "update", "patch"]
- apiGroups: ["extensions","apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
EoF
```

To have the role working we'll create a RoleBinding resource.

```bash
cat << EoF > deployuser-role-binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: update-deploy
  namespace: webserver-ns
subjects:
- kind: User
  name: deploy-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: update-deploy
  apiGroup: rbac.authorization.k8s.io
EoF
```

It's now time to apply both role and role binding.

```bash
kubectl apply -f webserver-deploy-role.yaml
kubectl apply -f deployuser-role-binding.yaml
```

## Update deployment manually using deploy-user

We'll use the [node affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity) feature to have at least 2 pods running a node labeled with `zone=zona-1` and 2 other nodes running a node labeled with `zone=zona-2`.

Let's add the required labels to each of the two involved nodes.

```bash
# Extract and export the first node name
export FIRST_NODE_NAME=$(kubectl get nodes -o json | jq -r '.items[0].metadata.name')

# Add the label to the first node
kubectl label nodes ${FIRST_NODE_NAME} zone=zona-1
```

```bash
# Extract and export the second node name
export SECOND_NODE_NAME=$(kubectl get nodes -o json | jq -r '.items[1].metadata.name')

# Add the label to the second node
kubectl label nodes ${SECOND_NODE_NAME} zone=zona-2
```

By switching to the deploy-user, equipped with the proper permissions described in webserver-deploy-role.yaml, we'll use helm to upgrade the chart deployment to have two pods always running in two groups of nodes respectively tagged with zona-1 and zona-2.

First of all, from the deploy-user environment, let's update the kubeconfig.

```
aws eks update-kubeconfig --name fantastic-negrito
```

We can now use the `nodeAffinityPreset.type`, `nodeAffinityPreset.key`, `nodeAffinityPreset.values` helm chart properties to have the 4 pods splitted into the 2 nodes evenly.

To do that let's create the following helm chart values.yaml file.

```bash
cat <<EoF > values.yaml
service:
  type: LoadBalancer

cloneHtdocsFromGit:
  enabled: true
  repository: "https://github.com/misurellig/hello-webpage.git"
  branch: "master"

replicaCount: 4

nodeAffinityPreset:
  type: "hard"
  key: "zone"
  values:
    - zona-1
    - zona-2
EoF
```

We'll use `helm upgrade` command to update k8s deployment with node affinity.

```bash
helm -n webserver-ns upgrade apache -f values.yaml bitnami/apache
```

To verify pods to nodes allocation run the following command:

```bash
kubectl get pod -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName -n webserver-ns
NAME                      STATUS    NODE
apache-775fc9b4d6-jbbfv   Running   ip-192-168-61-132.eu-west-1.compute.internal
apache-775fc9b4d6-l65kf   Running   ip-192-168-1-176.eu-west-1.compute.internal
apache-775fc9b4d6-w92nm   Running   ip-192-168-61-132.eu-west-1.compute.internal
apache-775fc9b4d6-wqszf   Running   ip-192-168-1-176.eu-west-1.compute.internal
```
