# best-practices-k8s

# Kubernetes Best Practices

## 1. Use Namespaces

Namespaces provide logical isolation within a Kubernetes cluster, allowing for resource segmentation. This is particularly useful in multi-tenant environments, where different teams or applications require their own isolated cluster space.

For example, you could create separate namespaces for development, testing, and production environments. This prevents accidental changes in the production environment by limiting access to it. 

Here is an example of how to define a namespace in a YAML file:

```yaml
apiVersion: v1  
kind: Pod   
metadata:  
  name: mypod  # Name of the Pod
  namespace: test  # Namespace where the Pod will be deployed
  labels:  
    name: mypod  # Label for the Pod
spec:  
  containers:  
  - name: mypod  # Container name
    image: nginx  # Image to be used for the container
```

The `namespace: test` line under `metadata` specifies that the pod `mypod` should be created in the `test` namespace.

## 2. Set Resource Requests & Limits

Resource requests and limits control how much CPU and memory a container can use. Requests are what a container is guaranteed to get, whereas limits are the maximum resources a container can use. Kubernetes uses these values to decide where to place pods and when to evict them.

Here is an example of how to set resource requests and limits in a YAML file:

```yaml
apiVersion: apps/v1   
kind: Deployment   
metadata:  
  name: nginx-deployment  # Name of the Deployment
  labels:  
    app: nginx  # Label for the Deployment
spec:  
  replicas: 3  # Number of replicas
  selector:  
    matchLabels:  
      app: nginx  # Selects Pods with label "app=nginx"
  template:  
    metadata:  
      labels:  
        app: nginx  # Label for the Pod created by this Deployment
    spec:  
      containers:  
      - name: nginx  # Container name
        image: nginx:1.14.2  # Image to be used for the container
        resources:  
            limits:  # Maximum resources that can be used
              memory: 200Mi  
              cpu: 1  
            requests:  # Guaranteed resources
              memory: 100Mi  
              cpu: 100m  
        ports:  
        - containerPort: 80  # Port on which the container is listening
```

The `resources:` block inside `containers:` specifies the requests and limits. The `requests:` block specifies that each `nginx` container in this pod will be guaranteed 100Mi of memory and 100m CPU. The `limits:` block specifies the maximum amount of resources the container can use: 200Mi of memory and 1 CPU.

## 3. Use readiness and liveness probes

Readiness and liveness probes are used by Kubernetes to know when a pod is ready to serve traffic and when to restart a container.

A readiness probe is used to know when a container is ready to start accepting traffic. A liveness probe is used to know when to restart a container.

```yaml
apiVersion: apps/v1   
kind: Deployment   
metadata:  
  name: my-deployment  # Name of the Deployment
spec:  
  template:  
    metadata:  
      labels:  
        app: my-test-app  # Label for the Pod created by this Deployment
    spec:  
      containers:  
     - name: my-test-app  # Container name
        image: nginx:1.14.2  # Image to be used for the container
        readinessProbe: 
          httpGet:  # Type of readiness probe
            path: /ready  # Path in the application to check
            port: 80  # Port on which the application is listening
          successThreshold: 3  # Number of successful probes to consider it ready
```

In this YAML, the `readinessProbe:` block specifies a readiness probe. This probe hits the `/ready` endpoint on port 80 of the application. If it gets a successful response three times (as specified by `successThreshold: 3`), it considers the application ready to serve traffic.

For more detailed probes, you can also specify things like `failureThreshold` (when to consider a probe failed), `initialDelaySeconds` (how long to wait before first probe), and `periodSeconds` (how long to wait between probes).

## 4. Use Role-based access control (RBAC)

RBAC is a method of regulating access to your Kubernetes cluster based on the roles of individual users within your organization. RBAC authorizes based on what actions a user can perform (verbs), what resources those actions can be performed on, and what namespaces those actions can be performed in.

Here is an example of a Role in Kubernetes:

```yaml
apiVersion: rbac.authorization.k8s.io/v1  
kind: Role  
metadata:  
  name: read-only  # Name of the Role
  namespace: default  # Namespace where the Role is created
rules:  
- apiGroups:  
  - ""  
  resources: ["*"]  # Resources that this Role can access
  verbs:  
  - get  # Actions that can be performed
  - list  
  - watch
```

This Role, named `read-only`, grants read access (get, list, watch) to all resources in the `default` namespace.

To bind this Role to a user, you would use a RoleBinding. Here is an example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-only-binding  # Name of the RoleBinding
  namespace: default  # Namespace where the RoleBinding is created
subjects:
- kind: User
  name: test-user  # Name of the User to bind the Role to
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: read-only  # Name of the Role to bind
  apiGroup: rbac.authorization.k8s.io
```

This RoleBinding binds the `read-only` Role to the user `test-user` in the `default` namespace.

For EKS you need to update aws-auth.yml, add IAM user then apply aws-auth.yml to rolebinding.

## 5. Monitor Your Cluster's Resources usage and Audit Policy Logs

Monitoring your cluster's resources and auditing policy logs are important for understanding your application's performance and security. For monitoring, you can use tools like Prometheus. For auditing, you can use audit logs provided by Kubernetes itself.

You can set up Fluent Bit and Fluentd to send your container logs to CloudWatch Logs. Here is an example of how to do it:

1. If you don't already have a namespace called `amazon-cloudwatch`, create one by running:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cloudwatch-namespace.yaml
   ```

2. Run the following command to create a ConfigMap named `cluster-info` with the cluster name and the Region to send logs to. Replace `cluster-name` and `cluster-region` with your cluster's name and Region:

   ```bash
   ClusterName=cluster-name

```bash
RegionName=cluster-region
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
kubectl create configmap fluent-bit-cluster-info \
--from-literal=cluster.name=${ClusterName} \
--from-literal=http.server=${FluentBitHttpServer} \
--from-literal=http.port=${FluentBitHttpPort} \
--from-literal=read.head=${FluentBitReadFromHead} \
--from-literal=read.tail=${FluentBitReadFromTail} \
--from-literal=logs.region=${RegionName} -n amazon-cloudwatch
```

3. Deploy the Fluent Bit daemonset to the cluster by running:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/fluent-bit/fluent-bit.yaml
   ```

4. Validate the deployment by entering:

   ```bash
   kubectl get pods -n amazon-cloudwatch
   ```

## 6. Organize Your Objects with Labels

Labels are key-value pairs attached to Kubernetes objects. They can be used to organize and select subsets of objects. Here is an example of how to use labels in a YAML file:

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: my-pod  # Name of the Pod
  labels:  
    environment: test  # Label indicating the environment
    service-type: frontend  # Label indicating the type of service
    app_version: v1.0  # Label indicating the version of the application
spec:  
  containers:  
  - name: my-container  # Container name
    image: my-image  # Image to be used for the container
```

In this example, the pod is labeled with `environment: test`, `service-type: frontend`, and `app_version: v1.0`. These labels can be used to select this pod for operations or for grouping in dashboards and reports.

## 7. Upgrade Kubernetes Version Regularly

You should always have the latest stable version of Kubernetes in your production cluster. The new releases often have many updates, additional features, and most importantly, patches to previous version security issues. This helps you in keeping your cluster away from vulnerabilities.

## 8. Use Version Control System for Configuration Files

Configuration files should be stored in a version control system before being applied to the cluster. This enables a history of changes, collaboration, and a quick rollback in case something goes wrong.

## 9. Use Autoscaling

Autoscaling in Kubernetes is handled by two components: the Horizontal Pod Autoscaler (HPA) and the Cluster Autoscaler (CA). HPA scales the number of pod replicas in a ReplicationController, Deployment, or ReplicaSet based on observed CPU utilization or other application-provided metrics. CA, on the other hand, scales your cluster size based on the demand of your workloads.


Horizontal Pod Autoscaler (HPA)

The Horizontal Pod Autoscaler automatically scales the number of pods in a replication controller, deployment, or replica set based on observed CPU utilization.

Before you create an HPA, make sure that the metric server is running in your cluster. If not, install it using the following command:
bash
Copy code
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
Once the metric server is running, you can create an HPA. Here's an example of how to create an HPA for a deployment named my-app:
bash
Copy code
kubectl autoscale deployment my-app --cpu-percent=50 --min=1 --max=10
This command creates an autoscaler that targets 50% CPU utilization for the pods in the my-app deployment, with a minimum of 1 pod and a maximum of 10 pods.

Cluster Autoscaler (CA)

The Cluster Autoscaler adjusts the size of the Kubernetes cluster (the number of nodes) based on the current needs, which means it automatically resizes the cluster when there are pods that failed to run in the cluster due to insufficient resources.

To use the CA, you need to install it in your cluster. The instructions depend on your cloud provider. Here are the instructions for GCP, AWS, and Azure.
Once the CA is installed, it should automatically adjust the size of your cluster based on the demands of your workloads.



## 10. Use Network Policies and a Firewall

Network policies and firewalls control the traffic that is allowed to reach your pods and what traffic your pods are allowed to send. By default, all traffic is allowed between pods. However, you can restrict traffic between pods in the same namespace or between pods in different namespaces.

Here is an example of a NetworkPolicy that allows traffic only from pods with the label `app: o

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy  # Name of the Network Policy
  namespace: default  # Namespace where the Network Policy is created
spec:
  podSelector:
    matchLabels:
      app: ola-api  # Selects Pods with label "app=ola-api"
  policyTypes:
  - Ingress  # Policy type
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: ola-ui  # Allows traffic only from Pods with label "app=ola-ui"
```

This NetworkPolicy, named `test-network-policy`, allows ingress traffic to any pod with the label `app: ola-api` but only from pods with the label `app: ola-ui`.

A similar policy would be applied for `ola-db` to allow traffic only from `ola-api`.

In addition to these Kubernetes-based network policies, you should also use a firewall to protect your cluster at the infrastructure level. This could be a cloud-provider firewall or a third-party firewall appliance, depending on where your cluster is hosted. The firewall should restrict access to your cluster's control plane and nodes, allowing only necessary traffic.

Keeping your firewall rules and network policies up to date is an important part of maintaining the security of your cluster. Regular reviews and audits of these rules are recommended.
