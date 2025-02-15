###Full CI/CD flow on kubernetes.

Step 1. Build the k8s cluster.

Step2. clone repo for jenkisn and sonar and apply.

Step3. Access jenkins over nodeport and with port defined in service.
kubectl get svc -n devops-tools

Step4. Access sonarqube over nodeport.
#kubectl get svc -n devops-tools

Step5- Get ready with below credentials and tokens


##########################################

Credentials needs to create --

sonar-token

Github token for jenkins

Docker cred with username and password

##########################################

Step6. Create above credentials in jenkins credentials store with the same ID name if defined in jenkinsfile of CI repo.

Step7. Create sonar webhook for sending quality gate status to jenkins



Step8. Install argocd (a seperate cluster)

####cluster configured
kubectl config get-contexts
docker-desktop (on docker desktop)
kubernetes-admin@kubernetes (with 3 nodes)


kubectl config use-context docker-desktop
kubectl config use-context kubernetes-admin@kubernetes

####Install argocd on docker-desktop k8s cluster
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
kubectl get svc argocd-server -n argocd
argocd admin initial-password -n argocd

####Login to argocd
https://localhost:31038
argocd login localhost:31038 --insecure
username --->admin
password---->

change password from CMD/powershell
argocd account update-password



####Add kubernetes-admin@kubernetes to argocd
argocd cluster add kubernetes-admin@kubernetes
This command generates a ServiceAccount in your VirtualBox cluster.
It also creates RBAC permissions so that ArgoCD can deploy to it.
argocd cluster list


kubectl config use-context docker-desktop
kubectl config use-context kubernetes-admin@kubernetes


Crds needs to be installe on target k8s cluster
kubectl apply -k https://github.com/argoproj/argo-cd/manifests/crds\?ref\=stable

kubectl get crds | grep applications


Step9. Create a deployement repo in Github that would be monitored by argocd. 
Place required file here like 
deployment.yamlwith service and other required files or helm chart for application (aur CI jenkinsfile stage will update image name either in values.yaml of helm chart or in deployment)
argocd-application.yaml (argocd application that will be deploy in argocd cluster)

Step10. Deploy your application first time by running
kubectl create -f argocd-application.yaml



Step11. Run CI pipeline , it will update your deployment repo with new image , once a commit is there in deployemnt repo , argocd will sync changes to application



Repositories:
1. https://github.com/chankyswami/Diamond-Carat-Calculator.git
2. https://github.com/chankyswami/Diamond-Carat-Calculator-deployment.git