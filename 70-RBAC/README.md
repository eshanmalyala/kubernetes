Use RBAC to Limit User Permissions
Kubernetes is a powerful container orchestration platform, but with great power comes the need for robust security measures. In this challenge you will focus on understanding a fundamental aspect of Kubernetes security: RBAC (Role-Based Access Control). 

RBAC in Kubernetes is used to control the actions users or groups can perform over specific resources within the cluster. To implement RBAC in Kubernetes, you will make use of different resources:

Roles and ClusterRoles: They define a set of rules. These rules are purely additive, meaning they are no deny rules (you will only be able to allow access but not deny). Roles are tied to a namespace whereas ClusterRoles are not namespace specific, they are cluster-scoped.

RoleBindings and ClusterRoleBindings: They bind a specific Role or ClusterRoles to users, groups or service accounts to grant them the permissions defined in the roles.

In this challenge you will create a Role that will allow user1 full control over Pods in the challenge2 namespace. You will also create a ClusterRole to allow user2 to list Pods in the entire cluster, but user2 will not be able to modify or create new Pods.

Before starting, you will ensure your Kubernetes cluster is ready (Note: minikube is being used):

Once ready, click the Open environment button to access the lab Ubuntu Desktop.

It takes about one to three minutes for the button to become available.

At the bottom of your lab's Desktop, click Terminal Emulator.

In the Terminal, ensure the environment is ready by running the status command.

If the output is SYSTEM COMPLETE, you are good to go! If you still get INITIALIZING, give it a few minutes and try again, it will take about three to six minutes; the lab is spinning up a kubernetes cluster for you!

Once the environment is up, ensure all kubernetes components are up: kubectl get pods -A

The output of the command should display all pods as Running, and ready containers as 1/1.

If any containers are NOT shown as 1/1, give it a minute and try again before proceeding further.
figure
Note: You can copy commands, and paste them into the Terminal by using Control+Shift+V.

Great! Now that your environment is ready, you can start creating the Role for user1:

First, create the namespace for this challenge using the kubectl create namespace challenge2 command and then switch to it: kubectl config set-context --current --namespace=challenge2

Check the provided Role definition: cat ~/challenges/02/role.yaml

It defines a Role resource named namespace-admin with a single rule that applies to the Pods resources and allows access to the create, delete, get, list, update, and watch verbs.
figure
﻿

Create the role Role: kubectl apply -f ~/challenges/02/role.yaml. Then ensure the role has been created by running the kubectl get role command.
figure
namespace-admin will appear in the output.

Now the Role has been created, associate the Role to user1 by using a RoleBinding resource:

kubectl create rolebinding user-admin-binding --role namespace-admin --user user1

Then ensure the binding has been created by running the kubectl get rolebinding command.
figure
﻿

Ensure user1 is able to create Pods in the challenge2 namespace but cannot perform other actions in other namespaces:

Change the user: sudo su user1

Note: the kube-context for user1 is set to use the challenge2 namespace by default.

Try to list pods in the kube-system namespace: kubectl get pods -n kube-system

Note the user is not able to fetch pods from the kube-system namespace, as you get an error. Remember your role only allows full control over the challenge2 namespace.
figure
﻿

The role does allow access to create Pods in the challenge2 namespace, create a pod: kubectl run user1-pod --image busybox:1.37 --command -- sleep 3600

No errors, yay! Ensure the user can also list pods in the challenge2 namespace as defined in the Role: kubectl get pods. Ensure the pod status is displayed as Running.

Disconnect the session for the user1 user: exit
figure
﻿

Good job! The Role for user1 is working as expected. Now you will use a ClusterRole to grant read access to user2 over all Pods in the cluster:

Check the provided definition: cat ~/challenges/02/clusterrole.yaml

It defines a ClusterRole resource named global-pod-reader with a single rule that applies to the Pods resources and allows only access to the get, list and watch verbs.
figure
﻿

Run the kubectl apply -f ~/challenges/02/clusterrole.yaml command to create the new ClusterRole.

Create the binding with the kubectl create clusterrolebinding global-pod-reader-binding --clusterrole global-pod-reader --user user2 command. 

Ensure user2 is able to list any pods within the cluster but it cannot do anything else, for example, create Pods:

Change the user: sudo su user2

Note: the kube-context for user2 is set to use the challenge2 namespace by default.

Create a pod in the challenge2 namespace: kubectl run user2-pod -n challenge2 --image busybox:1.37 --command -- sleep 3600

Note the user is not able to create Pods. Remember the user is bound to a ClusterRole that only allows read access over Pods.
figure
﻿

Ensure the user can list pods in all namespaces: kubectl get pods --all-namespaces

Disconnect the session for the user2 user: exit
figure
﻿

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
