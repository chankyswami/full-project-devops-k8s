#### Full CI/CD flow with jenkins, jenkins agent bundled with buildah,jnlp with trivy, sonarqube, trivy, argocd, kubernetes cluster.

Prerequisites:
1. Windows machine
2. VirtualBox
3. Docker desktop(Kubernetes enabled)
4. kubenets deployed on virtualbox (3-nodes)
5. DockerHub username and password (keep handy)
6. GitHub access token

Note: 
Target Cluster is : k8s cluster running on VirtualBox and that is our cluster for application hosting.
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
# kubectl create -k https://github.com/argoproj/argo-cd/manifests/crds\?ref\=stable
# kubectl get crds | grep applications

On target cluster(k8s cluster) run
# kubectl create ns argocd 
(This namespace is required on both source and target to get the status of sync)

Create dockerHub(registry) secret on target cluster for jnlp container.
# kubectl create ns devops-tools
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


Step-3. Apply Jenkins and SonarQube deployment

# kubectl config use-context kubernetes-admin@kubernetes
Clone repo for Jenkins and Sonar and deploy on k8s cluster(virtualbox one)
# kubectl create -f jenkins/ 
# kubectl create -f sonar/

Step-4. Setup SonarQube

# kubectl get svc -n devops-tools
Open browser and access http://172.16.16.100:30157(Port as per above command output)

Create webhook on sonarqube for jenkins for acknowledgement of sonargate
Go to  Administration --> Configuration --> Webhook

Create sonar token for analysis:
Click on A(administratr) -->My account-->Security-->create token (global analysis)

Enter
Name --> Your choice
URL  --> http://jenkins.devops-tools.svc.cluster.local:8080/sonarqube-webhook/
Secret --> Leave it blank


Step-5: Setup Jenkins


# kubectl get svc -n devops-tools
Get the nodeport and access jenkins and sonar over browser i.e, http://172.16.16.100:30489

Get initial password of jenkins- 
# kubectl exec -it -n devops-tools deploy/jenkins -- cat //var/jenkins_home/secrets/initialAdminPassword

Enter Initial password
Install suggested plugins

Setup your user account 

Now Go to manage jenkins --> Plugins -->Available plugins
Install "kubernetes" plugin
Install "sonarqube scannar"


Credentials needs to create in jenkins cred store -- 
1. sonar-token (Give ID is as per my jenkinsfile)
Create token in sonarqube ---->Click on A(administratr) -->My account-->Security-->create token (global analysis)

2. jenkins-token-github (Give ID as per my jenkinsfile)
3. dockerhub-username-password (Give ID as per my jenkinsfile)

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


Step- 6. Repository Setup:
1. Actual application repository (https://github.com/chankyswami/gfj-frontend.git).
Consists your application code and the jenkinsfile. It is being used for CI purpose.

2. Helm chart repository (https://github.com/chankyswami/gfj-frontend-deployment.git).
Maintaining helm chart and values.yaml file.


3. ArgoCD application repo (https://github.com/chankyswami/argocd-apps.git).
Consists all the applications being deployed by argocd.
# kubectl create -f argocd-application.yaml


Step- 7. Create a pipeline job in jenkins with scm(git), branch name is "main". Save it
Run CI pipeline , CI will generate a new image and will push it to dockerHub.

Now ArgoCD image updated controller is monitoring DockerHUB image path for the new image tag , once it get the new tag , it will notify Argocd and update the live application.


Step-8. Current context of deployment(gfj-frontend):
Service exposed over "clusterIP"
So now how to access it from your laptop:

########################################################

Expose clusterIP svc via ingress only(tested)
  
Laptop
  ↓
Ingress (NodePort internally)
  ↓
ClusterIP Service (gfj-frontend)
  ↓
Pods


# kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml

# kubectl get pods -n ingress-nginx

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gfj-frontend-ingress
  namespace: gfj-app
spec:
  ingressClassName: nginx
  rules:
  - host: gfj.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gfj-frontend
            port:
              number: 80
# kubectl apply -f ingress.yaml

# kubectl get nodes -o wide

Edit /etc/hosts (Linux / Mac) or C:\Windows\System32\drivers\etc\hosts:
192.168.56.101   gfj.local


http://gfj.local:31256(ingress controller port as it is exposed on nodeport)

#####################################################################
Expose clusterIP svc via metalB and ingress without port (tested)

Your nodes are on range:
172.16.16.0/24

Step 1. Install ingres resource ,

 apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gfj-frontend-ingress
  namespace: gfj-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: gfj-dev.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gfj-frontend
            port:
              number: 80

Apply above file.

STEP 2: Install MetalLB
Pick unused IPs (outside DHCP range).
I picked 172.16.16.105-172.16.16.110
# kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml

# kubectl get pods -n metallb-system

controller Running
speaker Running


STEP 3: Configure MetalLB IP pool
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: gfj-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.16.16.105-172.16.16.110
---
apiVersion: metallb.io/v1beta1You should see:
kind: L2Advertisement
metadata:
  name: gfj-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - gfj-pool


# kubectl apply -f metallb-config.yaml

STEP 4: Convert Ingress Controller to LoadBalancer
# kubectl patch svc ingress-nginx-controller \
  -n ingress-nginx \
  -p '{"spec":{"type":"LoadBalancer"}}'


# kubectl get svc -n ingress-nginx
You should see:
EXTERNAL-IP: 172.16.16.240
PORT(S): 80:xxxx/TCP, 443:yyyy/TCP


STEP 5: Update /etc/hosts of laptop (windows or linux)
172.16.16.105 gfj-dev.company.com

# sudo systemd-resolve --flush-caches   -----if needed

From browser---http://gfj-dev.company.com
##########################################################################################
SSL/TLS implemented and tested

OPTION 1: Self-Signed Certificate (Dev / Lab / Interview Demo)
Step 1: Generate cert & key on your laptop or a node
# openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout key.pem \
  -out cert.pem \
  -subj "/CN=gfj-dev.company.com"

This creates:
key.pem   → private key
cert.pem  → public certificate


Step 2: Create TLS secret in Kubernetes
# kubectl create secret tls gfj-tls \
  -n gfj-app \
  --cert=cert.pem \
  --key=key.pem

# kubectl describe secret gfj-tls -n gfj-app


Step:3 Update ingress object

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gfj-frontend-ingress
  namespace: gfj-app
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - gfj-dev.company.com
    secretName: gfj-tls
  rules:
  - host: gfj-dev.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gfj-frontend
            port:
              number: 80
			  
Step4: now access
https://gfj-dev.company.com

##################################################################