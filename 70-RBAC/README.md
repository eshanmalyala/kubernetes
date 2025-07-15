# Use RBAC to Limit User Permissions
Kubernetes is a powerful container orchestration platform, but with great power comes the need for robust security measures. In this challenge you will focus on understanding a fundamental aspect of Kubernetes security: RBAC (Role-Based Access Control). 

RBAC in Kubernetes is used to control the actions users or groups can perform over specific resources within the cluster. To implement RBAC in Kubernetes, you will make use of different resources:

### Roles and ClusterRoles:
They define a set of rules. These rules are purely additive, meaning they are no deny rules (you will only be able to allow access but not deny). Roles are tied to a namespace whereas ClusterRoles are not namespace specific, they are cluster-scoped.

## RoleBindings and ClusterRoleBindings: 
They bind a specific Role or ClusterRoles to users, groups or service accounts to grant them the permissions defined in the roles.

In this challenge you will create a Role that will allow user1 full control over Pods in the challenge2 namespace. You will also create a ClusterRole to allow user2 to list Pods in the entire cluster, but user2 will not be able to modify or create new Pods.

**First, create the namespace for this challenge using the**
```bash 
 kubectl create namespace challenge2
```
**command and then switch to it**:
```bash 
  kubectl config set-context --current --namespace=challenge2
```
### Check the provided Role definition: cat ~/challenges/02/role.yaml

It defines a Role resource named namespace-admin with a single rule that applies to the Pods resources and allows access to the create, delete, get, list, update, and watch verbs.
figure
﻿
### Create the role Role: kubectl apply -f ~/challenges/02/role.yaml. Then ensure the role has been created by running the kubectl get role command.

**Now the Role has been created, associate the Role to user1 by using a RoleBinding resource**:

```` bash
kubectl create rolebinding user-admin-binding --role namespace-admin --user user1
````

**Then ensure the binding has been created by running the kubectl get rolebinding command **﻿

Ensure user1 is able to create Pods in the challenge2 namespace but cannot perform other actions in other namespaces:

````bash Change the user: sudo su user1 ````

**Note: the kube-context for user1 is set to use the challenge2 namespace by default**

**Try to list pods in the kube-system namespace**: 
````bash kubectl get pods -n kube-system ````

Note the user is not able to fetch pods from the kube-system namespace, as you get an error. Remember your role only allows full control over the challenge2 namespace.

**The role does allow access to create Pods in the challenge2 namespace, create a pod**:
````bash kubectl run user1-pod --image busybox:1.37 --command -- sleep 3600 ````

No errors, yay! Ensure the user can also list pods in the challenge2 namespace as defined in the Role: 
**kubectl get pods**. Ensure the pod status is displayed as Running.


** Check the provided definition: cat ~/challenges/02/clusterrole.yaml**

It defines a ClusterRole resource named global-pod-reader with a single rule that applies to the Pods resources and allows only access to the get, list and watch verbs.
figure
﻿

**Run the kubectl apply -f ~/challenges/02/clusterrole.yaml command to create the new ClusterRole**.

Create the binding with the 
````bash kubectl create clusterrolebinding global-pod-reader-binding --clusterrole global-pod-reader --user user2 ````

Ensure user2 is able to list any pods within the cluster but it cannot do anything else, for example, create Pods:

**Change the user: sudo su user2**
**Note: the kube-context for user2 is set to use the challenge2 namespace by default**.

**Create a pod in the challenge2 namespace**:
````bash
 kubectl run user2-pod -n challenge2 --image busybox:1.37 --command -- sleep 3600

````
**Note the user is not able to create Pods. Remember the user is bound to a ClusterRole that only allows read access over Pods**.
﻿
Ensure the user can list pods in all namespaces:
````bash kubectl get pods --all-namespaces ````

Disconnect the session for the user2 user: exit
````

Well done! You have used RBAC to control what actions specific users can perform over certain resources within your cluster. If you are a cluster administrator, these are two good uses cases of RBAC:

Limit app developers access: Create Roles to grant developers access to only the namespaces they own. If a developer is just working on the component that allows payments on your websites, they don't need access to the kube-system namespace for example, where the kube-apiserver runs.

Manage permissions for your operational team: You will probably need a team that has access to the entire cluster, allowing them access to re-create Pods when something does not work as expected. However you may not want to allow them to access Secrets. For this, you can create a ClusterRole that will not be tied to a specific namespace and will define the resources and actions they can perform.

To verify the user permissions without actually using their credentials to authenticate against the kube-apiserver, you can also use kubectl  auth can-i command. For example, you can run the following commands to check the permissions of the user1 and user2 users:

user1 can read pods in the challenge2 namespace: kubectl auth can-i get pods -n challenge2 --as=user1

user1 can NOT read pods in the kube-system namespace: kubectl auth can-i get pods -n kube-system --as=user1

user1 can create pods in the challenge2 namespace: kubectl auth can-i create pods -n challenge2 --as=user1

user2 can read pods in the challenge2 namespace: kubectl auth can-i get pods -n challenge2 --as=user2

user2 can read pods in the kube-system namespace: kubectl auth can-i get pods -n challenge2 --as=user2

user2 can NOT create pods in the challenge2 namespace: kubectl auth can-i create pods -n challenge2 --as=user2
figure
﻿

What about the actions running Pods can perform over the cluster? Each Pod runs as a Service Account, in the next challenge you will learn how to create service accounts and provision roles to them.

# Allow a Pod to Communicate with the API Server
﻿Service Accounts provide an identity for processes running in pods, allowing them to authenticate with the Kubernetes API and access resources based on the permissions granted to that service account.

By default, every namespace in Kubernetes has a Service Account named default. If you don't specify a service account when creating a pod, Kubernetes automatically assigns this default service account to the pod. This Service Account does not have any permissions assigned to it by default (except for basic API discovery permissions if RBAC is enabled) - it will be able to authenticate and access the Kubernetes API but it will not be able to perform any operations.

When the Pod gets created, the service account token is typically mounted at /var/run/secrets/kubernetes.io/serviceaccount/token within each container of the Pod. This token file allows the application running inside the pod to authenticate with the Kubernetes API server. Along with the token, other related files, such as the CA certificate and namespace, are also mounted in the same directory, providing necessary information for secure communication with the API. For increased security, if your Pod does not need to interact with the kube-apiserver, you can set the automountServiceAccountToken field to false in the pod specification to avoid the token getting automatically mounted.

Imagine now that you have an online learning platform, where users are able to spin up lab environments to practice their knowledge. Each of the lab environments will be a Pod created on demand via your main application, which also runs in Kubernetes. For your application to provision the new Pod when a user requests a lab environment, it will need to authenticate against the kube-apiserver and send the request to create new Pods. Using the default Service Account will not work because it does not have any permissions by default.

You will now create a new service account, provision to it the required permissions via a Role and its respective RoleBinding and finally you will create a Pod and ensure it can connect and create other Pods in the cluster by interacting with the Kubernetes API Server.

First, create the namespace for this challenge using the kubectl create namespace challenge3 command and then switch to it: kubectl config set-context --current --namespace=challenge3

Create a service account: kubectl create serviceaccount pod-access-sa. Then run the kubectl get sa and ensure the Service Account pod-access-sa has been created.
figure
﻿

You will now create the Role to be used by the service account. Check the provided definition: cat ~/challenges/03/role.yaml

It defines a Role resource named pod-writer with permissions to create pods.
figure
﻿

Create the role: kubectl apply -f ~/challenges/03/role.yaml. Then run the kubectl get role command to ensure the Role has been created.
figure
﻿

In the previous challenge you created the RoleBinding using kubectl commands. In this challenge you will create it via a definition file. Check the provided definition by running the cat ~/challenges/03/rolebinding.yaml command.

It defines a RoleBinding resource named pod-writer-binding, where the subject is the pod-access-sa service account in the challenge3 namespace.
figure
﻿

Create the Role Binding: kubectl apply -f ~/challenges/03/rolebinding.yaml. Then run the kubectl get rolebinding to ensure the resource has been created as expected:
figure
﻿

Now you will create the pod that will use the new service account. Check the provided definition: cat ~/challenges/03/pod.yaml

It defines a Pod resource named api-access-pod which runs the busybox:1.37 image and uses the pod-access-sa service account. 
figure
﻿

Run the kubectl apply -f ~/challenges/03/pod.yaml command to create the pod.

Now connect to the new pod and send a request to the Kubernetes API Server to create a pod:

Connect to the pod: kubectl exec -it api-access-pod -- sh

Send the POST request:

wget --no-check-certificate --header="Content-Type: application/json" --header="Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" --post-data='{"apiVersion": "v1", "kind": "Pod", "metadata": {"name": "test-pod"}, "spec": {"containers": [{"name": "nginx", "image": "nginx:1.27.2-alpine-slim"}]}}'  https://kubernetes.default.svc/api/v1/namespaces/challenge3/pods -O-

Remember, you can copy commands, and paste them into the Terminal by using Control+Shift+V.

Note: The POST Request includes:

The contents of the /var/run/secrets/kubernetes.io/serviceaccount/token file in the Authorization header to authenticate against the kube-apiserver.

The json with the Pod definition. The pod is named test-pod and uses the nginx:1.27.2-alpine-slim image.
figure
﻿

Disconnect from the pod: exit

Check the pod test-pod has been successfully created: kubectl get pods
figure
﻿

Well done! You have successfully created a Service Account with permissions to create Pods. You have also validated the access by performing the creation operation directly from your Pod, just like your application would do. In the next challenge you will limit the creation of Pods by enforcing security policies.

# Enforce Security Standards While Creating Pods
﻿Pod Security Policy (PSP) was a Kubernetes feature that allowed administrators to define security conditions for pods. However, it was complex to configure and difficult to audit. The feature was deprecated in Kubernetes v1.21, and removed from Kubernetes in v1.25. Pod Security Standards (PSS) were introduced to provide a simpler framework for managing security.

Instead of administrators deciding what escalations to limit per Pod basis, PSS defines three levels of security:

Privileged: This policy is completely unrestricted, allowing all permissions. It's intended for trusted workloads that require broad access, typically for system-level tasks.

Baseline: A minimally restrictive policy that prevents known privilege escalations while allowing common workloads. It strikes a balance between security and usability, making it suitable for most applications.

Restricted: The most stringent policy, enforcing best practices for pod hardening. It is designed for security-critical applications, ensuring that containers run with minimal privileges and adhere to strict security controls.

The Pod Security Standards are implemented via the Pod Security Admission Controller, which can operate in three modes:

Enforce: Rejects pods that violate the specified policy.

Audit: Allows pods but logs violations for review.

Warn: Allows pods while providing warnings about policy violations.

The Pod Security Admission Controller will read the labels defined in the namespace and apply the required security measures. This means that when you apply a Pod Security Standard label to a namespace, it affects all pods created within that namespace. This lets the wider community define how they want to restrict the behavior of pods in a clear, consistent fashion, ensuring security standards are enforced across organizations. However, if you need to define more granular permissions per Pod basis, you should look at other options like the Open Policy Agent (OPA).

One of the reasons Pod Security Policies were deprecated was due to the difficulty of auditing breaches in the policy. You will now use Pod Security Standards and the Pod Security Admission Controller to restrict and audit what pods can do within your namespace. 

First, create the namespace for this challenge using the kubectl create namespace challenge4 command and then switch to it: kubectl config set-context --current --namespace=challenge4

Label the namespace so pods cannot be run as root. For this you will enforce the restricted level:

kubectl label --overwrite ns challenge4 pod-security.kubernetes.io/enforce=restricted

Now you will create a pod that doesn't meet the policy. Check the provided definition: cat ~/challenges/04/pod-non-compliant.yaml

It defines a pod named non-compliant-pod which runs a busybox container as root (runAsUser: true).
figure
﻿

Run the kubectl apply -f ~/challenges/04/pod-non-compliant.yaml command to create the pod. The Pod Security Admission Controller will reject the creation due to several reasons:

allowPrivilegeEscalation is not set to false

capabilities are not restricted

runAsNonRoot is not set to true

seccompProfile is not defined. The type should be set to RuntimeDefault or Localhost
figure
﻿

The busybox images by default run as root user, so you will need to create a new image that doesn't run as root:

Check the provided Dockerfile: cat ~/challenges/04/Dockerfile

It defines the steps required to create the image, including the addition of the myuser user.
figure
﻿

Set-up docker environment to use minikube's docker daemon: eval $(minikube docker-env)

Build the image: docker build -t non-root-busybox:0.0.1 ~/challenges/04/.
figure
It'll output Successfully built

Ensure the new non-root-busybox image is present: docker images
figure
﻿

Now that your image is ready, update the Pod definition to meet the security standards. Check the provided definition: cat ~/challenges/04/pod-compliant.yaml

It defines a pod named compliant-pod which uses the image you created previously and also fixes the vulnerabilities raised in the previous apply command. For more details, see Configure a Security Context for a Pod or Container.
figure
﻿

Run the kubectl apply -f ~/challenges/04/pod-compliant.yaml command to create the pod, the Pod Security Admission Controller will allow the creation. Then run the kubectl get pods command to ensure the pod has been created and its status is deployed as 1/1 Running.
figure
﻿

Awesome! Without much effort you have identified and fixed the security vulnerabilities in your Pod using the Pod Security Admission Controller. In the next challenge you will look into securely storing sensitive data in Kubernetes.

# Store Secrets in Kubernetes
﻿Secrets are a mechanism for securely storing and managing sensitive information, such as passwords and tokens, within your Kubernetes cluster. They play a crucial role in enhancing security by keeping sensitive data separate from application code, thereby reducing the risk of accidental exposure. Access to these secrets can be controlled through Role-Based Access Control (RBAC), ensuring that only authorized users and applications can retrieve them. Secrets can be injected into applications as environment variables or mounted as volumes, allowing for secure access at runtime.

In this challenge you will look into minimizing the exposure of your sensitive data. In particular, you will work on securing the password used by a MySQL database.

First, create the namespace for this challenge using the kubectl create namespace challenge5 command and then switch to it: kubectl config set-context --current --namespace=challenge5

Check the provided deployment definition: cat ~/challenges/05/deployment-v1.yaml

It defines a Deployment named mysql, running a single container with the mysql:9.1.0 image. The container has the MYSQL_ROOT_PASSWORD environment variable with password123 as value.
figure
﻿

Create the database deployment: kubectl apply -f ~/challenges/05/deployment-v1.yaml. Then run the kubectl get deploy to ensure the deployment has been created. Ensure the status is reported as ready 1/1.
figure
﻿

You, as developer and application owner, can have access to the mysql password, stored at the moment in the environment variable. What about a user that has read permissions over Pods? Run the kubectl get deployment mysql -o yaml | grep -i pass command, you will be able to see the password password123.
figure
﻿

Create a secret to store the password: kubectl create secret generic mysql-pass --from-literal=password=password123. Then ensure the secret mysql-pass has been created: kubectl get secrets
figure
﻿

Now update your deployment to use the new secret and remove the hardcoded password. Check the provided new version of the deployment: cat ~/challenges/05/deployment-v2.yaml. 

The value of the MYSQL_ROOT_PASSWORD is now obtained from the mysql-pass secret.
figure
﻿

Deploy the new version of the deployment: kubectl apply -f ~/challenges/05/deployment-v2.yaml, then inspect again the definition of the running deployment by running the kubectl get deployment mysql -o yaml | grep -i pass command. You will not be able to find the password hardcoded.
figure
﻿

Nice job! You have successfully stored sensitive data into a Kubernetes Secret resource and used the new Secret into your Deployment definition. This ensures only users with read access over Secret resources will be able to read sensitive data. In the next challenge you will be able to play a bit more in this virtual world before tearing it down.

# The Last Challenge
Welcome to the Final Challenge!

This is your last chance to experiment in the environment. Clicking Finish Lab will end this little world that flittered into existence just for you.

Take this opportunity to try new things. Don't be afraid to break anything; be curious!

Here are some things to try out:

Using RBAC, create a Role to allow only deploy only CronJobs, but not deployments.

Use secrets to store the TLS certificate files located at /home/pslearner/.minikube/cert.pem and /home/pslearner/.minikube/key.pem. Use the Kubernetes documentation for this.

You have unlimited power within this virtual world - take the time you need for unguided learning.
