# Creating end to end GitOps pipeline on Red Hat OpenShift using ArgoCD and Tekton
 ![image](https://github.com/user-attachments/assets/fd36dd40-0314-483a-90e7-f7801ad3d6b3)

## What is Tekton?
In the realm of cloud-native development, Continuous Integration and Continuous Delivery (CI/CD) have become critical components for building, testing, and deploying applications seamlessly. With the rise of Kubernetes and containerization, developers need efficient tools to manage their CI/CD pipelines effectively. Enter Tekton, a powerful open-source framework designed specifically for cloud-native CI/CD workflows.
Tekton is a Kubernetes-native framework that focuses on providing a declarative and extensible approach to building CI/CD systems. Born as an open-source project under the umbrella of the Continuous Delivery Foundation (CDF), Tekton leverages the Kubernetes API and utilizes custom resource definitions (CRDs) to define pipeline resources, tasks, and workspaces. It brings the advantages of scalability, portability, and reproducibility to your CI/CD workflows, making it an excellent choice for cloud-native environments.

### Key Features and Concepts
1.	Tasks: The fundamental building blocks of a Tekton pipeline are tasks. Each task represents a specific unit of work, such as building code, running tests, or deploying an application. Tasks can be combined and reused across pipelines, promoting modularity and code sharing.
2.	Pipelines: Pipelines provide a way to orchestrate tasks in a specific order to create an end-to-end CI/CD workflow. With Tekton, you can define complex pipelines that include multiple stages, parallel execution, and conditional branching.
3.	Resources: Resources represent the inputs and outputs of tasks within a pipeline. They can include source code repositories, container images, or any other artifacts required for the pipeline execution. Tekton enables you to define and manage resources as Kubernetes CRDs.
4.	Workspaces: Workspaces allow you to share files between tasks within a pipeline. They provide a mechanism for passing data and artifacts between different stages of the CI/CD workflow. Workspaces ensure isolation and reproducibility, making it easier to manage complex pipe.
 ![image](https://github.com/user-attachments/assets/ff1b8fad-e826-40d9-be35-af780d0ad0be)

6. A task can consist of multiple steps, and pipeline may consist of multiple tasks. The tasks may run in parallel or in sequence.

## What is ArgoCD
ArgoCD has a native support in OpenShift called OpenShift GitOps which is based on ArgoCD.
Argo CD is a declarative continuous delivery tool for Kubernetes that enables developers to automate application deployments across multiple clusters. It follows the GitOps philosophy, where the desired state of the application is defined in a Git repository, and Argo CD ensures that the actual state matches the desired state continuously. By leveraging Kubernetes custom resources, Argo CD provides a declarative approach to application deployment, making it easier to manage complex configurations and rollbacks.

### Key features of ArgoCD
1.	GitOps Approach: With Argo CD, the desired state of your application is defined in a Git repository, allowing you to manage deployments using familiar Git workflows. This approach brings version control, auditability, and collaboration to the deployment process, making it easier to track changes and maintain a reliable application state.
2.	Declarative Application Definitions: Argo CD uses Kubernetes manifests, such as YAML files, to define the desired state of your application. This declarative approach eliminates the need for manual intervention during the deployment process, ensuring consistency and reproducibility across different environments.
3.	Continuous Delivery: Argo CD continuously monitors the state of your applications and automatically reconciles any differences between the desired and actual states. It detects changes in the Git repository and triggers deployments, rollbacks, or updates accordingly, ensuring that your applications are always up to date.
4.	Multi-Cluster Support: Argo CD simplifies the management of multiple Kubernetes clusters. It provides a unified view of all your clusters, allowing you to deploy applications to multiple environments from a single control plane. This centralized approach enhances operational efficiency and simplifies the management of complex infrastructures.
5.	Rollback and Rollforward: Argo CD makes it effortless to roll back or roll forward to specific application versions. By leveraging the version history stored in the Git repository, you can easily revert to a previous state or progress to a newer version, providing flexibility and agility in managing deployments.

## Let’s get our hands dirty
You will need your personal GitHub account with a Personal Access Token, and your personal registry with an encrypted password.
 ![image](https://github.com/user-attachments/assets/a6e11cb7-168c-4ce5-a8e3-9850d7fb7d56)
 
### Let's explain the architecture
1. This is a sample pipeline based on a .Net core application
2. We have 2 repositories [https://github.com/ericbos111/dotnetcore-ci-ocp-pipeline](https://github.com/ericbos111/dotnetcore-ci-ocp-pipeline) for the application code and Tekton resources, and [https://github.com/ericbos111/dotnetcore-gitops](https://github.com/ericbos111/dotnetcore-gitops) where the ArgoCD resources are defined.
3. Whenever there is a trigger in the first repository, when there is a change in application code, through the trigger of webhook, tekton will start to clone, build source code, build the docker image, and push to registry
4.	Tekton will also then commit the changes to the other repository, so the image tag is pushed
5.	The resources that includes all the yamls that is needed to deploy the application such as deployment, services, quote, replica set, is stored in second repository which is used for GitOps (ArgoCD)
6.	After Tekton completes its task, ArgoCD will then sync with the latest changes of your application, it could be a change in replica count, latest image, rollbacks etc

### Step 1: Install ArgoCD and Tekton through the operatorhub in OpenShift
* ArgoCD is called OpenShift GitOps in OpenShift
* Tekton is called OpenShift Pipelines in OpenShift
Navigate to OperatorHub in OpenShift and install OpenShift Gitops in OpenShift and OpenShift Pipelines in OpenShift

![image](https://github.com/user-attachments/assets/489b7b38-0b85-4452-99f8-b595d92e5318)
 
### Step 2: Create your account in quay.io
Create your account in quay.io and create a repository named dotnetcore. You can also use another registry if you have.
![image](https://github.com/user-attachments/assets/925bd060-79a4-4b5e-9e0e-afb6f9fb8163)
 
If needed, change the password of quay.io by going to the account settings and clicking on **generate new encrypted password**.
![image](https://github.com/user-attachments/assets/3f845fee-6439-47cb-8a4c-1fa0b98f93a1)
 
Note down the password.
### Step 3: Set up Tekton
1. Fork both repositories if you haven't already
2. Create your own namespace
3. Create a secret to push your image to your registry
```
oc create secret docker-registry quay-secret \
--docker-server=quay.io \
--docker-username= <QUAY_USERNAME> \
--docker-password=<ENCRYPTED_PASSWORD>
```
4. Create a github token secret that will allow Tekton to push changes to your Github
```
apiVersion: v1
kind: Secret
metadata:
  name: git-user-pass
  annotations:
    tekton.dev/git-0: https://github.com
type: kubernetes.io/basic-auth
stringData:
  username: <github user>
  password: <github personal access token>
```
Apply the yaml file to create the secret in your namespace
```
oc apply -f secret.yaml
```
5. Link the secrets to the pipeline service account
```
oc secret link pipeline quay-secret
oc secret link pipeline git-user-pass
```
6. Navigate to k8s/pipeline.yaml, and replace with your github repo, edit line 10 and 22, and line 16 needs to reflect your personal quay.io registry.
7. Apply the Tekton resources. Navigate to the k8s folder and run the following commands
```
oc apply -f dotnetcore-api-pvc.yaml
oc apply -f el-route.yaml
oc apply -f eventlistener.yaml
oc apply -f git-update-deployment-task.yaml
oc apply -f pipeline.yaml
oc apply -f triggerbinding.yaml
oc apply -f triggertemplate.yaml
```
8. This will create all Tekton resources and create a webhook URL, copy the URL by looking at its route
```
oc get route
```
![image](https://github.com/user-attachments/assets/c5f244b7-b214-482e-8cf5-b6add620c412)
 
in my case it is “el-dotnetcore-api-dotnetcore.apps.cluster-l8wqt.l8wqt.sandbox952.opentlc.com”. Copy this route and navigate to your github pipeline repo, navigate to settings, then webhook
9. Create the webhook by clicking on **add webhook**
![image](https://github.com/user-attachments/assets/ae3fcbbf-621d-4925-8fc1-a3780dc0b5e5)
  
Don't worry if the webhook doesn't work, you can still continue with the exercise. Your cluster might not be publicly accessible.
To check the pipeline, navigate to pipeline and go to your respective project, to see your pipeline. If you get an error message about the Dockerfile, make sure your pipeline uses a PVC as the shared workspace.

![image](https://github.com/user-attachments/assets/2af7e97a-d16c-4f69-843e-e3b0093c1eb7)
 
### Step 4: Set up ArgoCD
1. Apply permission to ArgoCD
oc adm policy add-cluster-role-to-user cluster-admin -z openshift-gitops-argocd-application-controller -n openshift-gitops
2. Fork the repository (you probably did that already)
3. Create an ArgoCD application, pointing to the repository that you have created. Replace the destination namespace with your own namespace. Leave the project as default.
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vote-app-dev
  namespace: openshift-gitops
spec:
  destination:
    namespace: <your-namespace>
    server: https://kubernetes.default.svc 
  project: default 
  source: 
    path: environments/dev
    repoURL: <your-gitops-forked-repo>
    targetRevision: main
  syncPolicy: 
    automated:
      prune: true
      selfHeal: false
    syncOptions:
    - CreateNamespace=true
```
4. Apply the yaml configuration
```
oc apply -f argo.yaml
```
5. Login to ArgoCD by getting its route, and password like this. The username is admin
```
oc get route -n openshift-gitops
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```
You can also login with OpenShift authentication.
Once you login, you can see your application deployed by ArgoCD
 ![image](https://github.com/user-attachments/assets/9c037091-3aaa-4b6a-8e62-71b4faf6258b)

 ![image](https://github.com/user-attachments/assets/8edc4e83-a409-47a3-8bb2-6e7890bb0db8)
 
### Step 6: Make a trigger by committing a new change in application code
1. Change something in application code and commit/push new changes
```
git commit -am "new changes"
git push 
```

![image](https://github.com/user-attachments/assets/45d6da41-235a-4e01-bd18-7959d5c807d0)
 
2. The webhook will start the CI using Tekton

![image](https://github.com/user-attachments/assets/38a2ef52-1afe-49cc-b33d-070a3434616b)

4. Wait for some time for CI to finish
5. Navigate to your other Github repo, and you will notice that Tekton has pushed the latest changes to your GitOps repo.

![image](https://github.com/user-attachments/assets/f9301fe8-74eb-42f2-927e-1d3397289f76)

7. You can either wait 3 minutes for ArgoCD to automatically sync with latest changes of your repo, or you can manually click on sync on Argo

![image](https://github.com/user-attachments/assets/f2d0b2b5-e954-4651-9c48-90df9ef85dbc)

#### Congratulations, your end to end GitOps using Tekton and ArgoCD is ready 

### References
1.	[Introduction to OpenShift Pipelines](https://www.redhat.com/en/blog/introducing-openshift-pipelines)
2.	[GitOps documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.16/html/understanding_openshift_gitops/about-redhat-openshift-gitops)
3.	[original pipeline repo](https://github.com/SaifRehman/dotnetcore-ci-ocp-pipeline)
4.	[original gitops repo](https://github.com/SaifRehman/dotnetcore-gitops)

