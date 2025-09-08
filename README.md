#### Full CI/CD flow with jenkins, jenkins agent bundled with buildah,jnlp with trivy, sonarqube, trivy, argocd, kubernetes cluster.

Prerequisites:
1. Windows machine
2. VirtualBox
3. Docker desktop(Kubernetes enabled)
4. kubenets deployed on virtualbox (3-nodes)
5. DockerHub username and password (keep handy)
6. GitHub access token

Note: 
Target Cluster is : k8s cluster running on VirtualBox an that is our cluster for application hosting.
Docker Desktop Custer: Kubernetes cluster used for Argocd installation and setup.


Step 1. Build the k8s cluster(Follow given repository below)
On Master--->
# export KUBECONFIG=/etc/kubernetes/admin.conf

If want to access cluster from local 
# mkdir -p ~/.kube/config
paste the content in "~/.kube/config"
Copy content of "/etc/kubernetes/admin.conf"

Install kubectl cli on windows(Google it and install)

Argocd CRDs needs to be installed on target k8s cluster
# kubectl apply -k https://github.com/argoproj/argo-cd/manifests/crds\?ref\=stable
# kubectl get crds | grep applications

On target cluster run
# kubectl create ns argocd 
(This namespace is required on both source and target to get the status of sync)

Create dockerHub(registry) secret on target cluster for jnlp container.
# kubectl create secret docker-registry docker-credentials \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=xxxx \
  --docker-password=xxxxx \
  --docker-email=xxxxx \
  -n devops-tools



Step-2 Install argocd (a seperate cluster) on Docker Desktop.

Install Docker desktop 
open --->settings--->Enable kubernetes


Check cluster and switch across
# kubectl config get-contexts
# kubectl config use-context docker-desktop
# kubectl config use-context kubernetes-admin@kubernetes


####Install argocd on docker-desktop k8s cluster
# kubectl create namespace argocd
# kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
# kubectl get svc argocd-server -n argocd
# argocd admin initial-password -n argocd

Login to argocd
Open browser ----> https://localhost:31038(check port whatever is being assigned)

### On powershell or git bash
# argocd login localhost:31038 --insecure
Enter username --->
Enter password---->

Change password from CMD/powershell or gitbash
# argocd account update-password



Add "kubernetes-admin@kubernetes" cluster to argocd cluster
# argocd cluster add kubernetes-admin@kubernetes
This command generates a ServiceAccount in your VirtualBox cluster.
It also creates RBAC permissions so that ArgoCD can deploy to it.

# argocd cluster add kubernetes-admin@kubernetes
WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `kubernetes-admin@kubernetes` with full cluster level privileges. Do you want to continue [y/N]? y
{"level":"info","msg":"ServiceAccount \"argocd-manager\" created in namespace \"kube-system\"","time":"2025-09-07T13:31:47+05:30"}
{"level":"info","msg":"ClusterRole \"argocd-manager-role\" created","time":"2025-09-07T13:31:47+05:30"}
{"level":"info","msg":"ClusterRoleBinding \"argocd-manager-role-binding\" created","time":"2025-09-07T13:31:47+05:30"}
{"level":"info","msg":"Created bearer token secret for ServiceAccount \"argocd-manager\"","time":"2025-09-07T13:31:47+05:30"}
Cluster 'https://172.16.16.100:6443' added

# argocd cluster list


Step-3: Setup Jenkins

On Local machine clone repo for Jenkins and Sonar and deploy

# kubectl create -f jenkins/ 
# kubectl get svc -n devops-tools
Get the nodeport and access jenkins over browser i.e, http://172.16.16.100:30489

Get initial password- 
# kubectl exec -it -n devops-tools deploy/jenkins -- cat //var/jenkins_home/secrets/initialAdminPassword

Enter Initial password
Install suggested plugins

Setup your user account

Now Go to manage jenkins --> Plugins -->Available plugins
Install "kubernetes" plugin
Install "sonarqube scannar"

Go to Manage jenkins --> system --> SonarQube Servers
Name- sonar (same what you given in jeninsfile)
Server URL - http://sonarqube.devops-tools.svc.cluster.local:9000 (As both jenkins and sonar running in same k8s cluster and in same namespace)
Authentication Token  - That you already created in Jenkins credentials store.


Setup cloud(kubernetes) in jenkins
Go to Manage jenkins --> Clouds --> Select "kubernetes"
Fill below details
Kubernetes URL -- >https://kubernetes.default:443 (As jenkins is on same cluster so will resolve)
Jenkins URl --> http://jenkins.devops-tools.svc.cluster.local:8080 (Cluster internal endpoint)
Jenkins Tunnel --> jenkins.devops-tools.svc.cluster.local:50000 (For jnlp agent communication)


Credentials needs to create -- 
1. sonar-token (This ID is as per my jenkinsfile)
2. jenkins-token-github (This ID  is as per my jenkinsfile)
3. dockerhub-username-password (This ID is as per my jenkinsfile)


Step-4. Setup SonarQube
# kubectl get svc -n devops-tools
Open browser and access http://172.16.16.100:30157(Port as per above command output)

Create webhook on sonarqube for jenkins for acknowledgement of sonargate
Go to  Administration --> Configuration --> Webhook

Enter
Name --> Your choice
URL  --> http://jenkins.devops-tools.svc.cluster.local:8080/sonarqube-webhook/
Secret --> Leave it blank



Step- 5. Repository Setup:
1. Actual application repository (https://github.com/chankyswami/Diamond-Carat-Calculator.git)
Consists your application code and the jenkinsfile

2. ArgoCD Deployment repository (https://github.com/chankyswami/Diamond-Carat-Calculator-deployment.git)
Maintaing 3 files as of now:
argocd-app.yaml
deployment.yaml
service.yaml

Place manifests like above here or helm chart for application deployment (jenkinsfile stage will update image name either in values.yaml of helm chart or in deployment.yaml)

3. Deploy your application first time by running
# kubectl create -f argocd-application.yaml


Step- 6. Create a pipeline job in jenkins with scm(git), branch name is "chanky". Save it
Run CI pipeline , it will update your deployment repo with new image tag , once a commit is there in deployemnt repo , argocd will sync changes to application.
