# Provision a GKE Cluster for the Splunk K8s Operator
Provision a GKE Cluster using Terraform for the Splunk K8s Operator and deploy Splunk Enterprise on Google Kubernetes Engine (GKE).
  
The instructions in this repo will:
 - Provide `terraform` to create an GKE cluster  
 - Deploy K8s Metrics Server
 - Deploy K8s Dashboard
 - Install Splunk Operator for K8s
 -  Deploy a Standalone deployment of Splunk Enterprise on EKS  

### Before you start
Before you begin, you will need the following:
 - GCP Account
 - GCP CLI Installed
 - Kubernetes CLI
 - wget installed
 - Terraform 0.14.11
  
### 1. Deploy GKE Cluster
Clone the repo
```bash
git clone git@github.com:anthonygrees/gke_splunk_k8s_demo.git
cd gke_splunk_k8s_demo
```
  
Create a `terraform.tfvars` file with the following:
```bash
project_id = "REPLACE_ME"
region     = "us-central1"
creds      = "~/directory/your_gcp_project_creds.json"
```
  
After you have saved your customised variables file, initialize your Terraform workspace, which will download the provider and initialize it with the values provided in your `terraform.tfvars` file.
```bash
terraform init
```
  
Apply the terraform to create the EKS cluster
```bash
terraform apply
```
  
This process should take approximately 7 minutes. Upon successful application, your terminal prints the outputs  
```bash
google_container_node_pool.primary_nodes: Creation complete after 1m46s [id=projects/testterraform-312108/locations/asia-southeast1/clusters/testterraform-312108-gke/nodePools/testterraform-312108-gke-node-pool]

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

kubernetes_cluster_host = "35.185.888.888"
kubernetes_cluster_name = "yourProject-3333333-gke"
project_id = "yourProject-3333333"
region = "asia-southeast1"
```
  
![gke](/images/gke.png)
  
### 2. Configure kubectl
Now that you've provisioned your GKE cluster, you need to configure kubectl.
   
Run the following command to retrieve the access credentials for your cluster and automatically configure kubectl.  
  
```bash
gcloud container clusters get-credentials $(terraform output -raw kubernetes_cluster_name) --region $(terraform output -raw region)
```
  
`NOTE`: You may see the following warning message when you try to retrieve your cluster credentials. This may be because your Kubernetes cluster is still initializing/updating. If this happens, you can still proceed to the next step.
```
WARNING: cluster dos-terraform-edu-gke is not running. The kubernetes API may not be available.
```
  
### 3. Deploy and access Kubernetes Dashboard via Proxy
To verify your cluster is correctly configured and running, you will deploy the Kubernetes dashboard and navigate to it in your local browser.  
  
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```
  
Your output will look like this:
```bash
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```
  
Now, create a proxy server that will allow you to navigate to the dashboard from the browser on your local machine. This will continue running until you stop the process by pressing `CTRL + C`.
```bash
kubectl proxy
```
  
You should be able to access the Kubernetes dashboard [here](http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)
   
### 4. Authenticate the dashboard  (New Terminal)
To use the Kubernetes dashboard, you need to create a ClusterRoleBinding and provide an authorization token.
  
In another terminal (do not close the kubectl proxy process), create the ClusterRoleBinding resource.
```bash
kubectl apply -f https://raw.githubusercontent.com/anthonygrees/eks_splunk_k8s_demo/master/kubernetes-dashboard-admin.rbac.yaml
```
  
Then, generate the authorization token.
```bash
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep service-controller-token | awk '{print $1}')
```
  
Select "Token" on the Dashboard UI then copy and paste the entire token you receive into the dashboard authentication screen to sign in. You are now signed in to the dashboard for your Kubernetes cluster.
  
Navigate to the "Cluster" page by clicking on "Cluster" in the left navigation bar. You should see a list of nodes in your cluster.
  
![gke_dash](/images/gke_dash.png)
  
### 5. GKE nodes and node pool
On the Dashboard UI, click Nodes on the left hand menu.  
  
Notice there are 6 nodes in your cluster, even though gke_num_nodes in your `gke.tf` file was set to 2. This is because a node pool was provisioned in each of the three zones within the region to provide high availability. 
```bash
gcloud container clusters describe yourProject-gke --region asia-southeast1 --format='default(locations)'
locations:
- asia-southeast1-a
- asia-southeast1-b
- asia-southeast1-c
```
  
### 6. Installing the Splunk Operator
A Kubernetes cluster administrator can install and start the Splunk Operator by running:
```bash
kubectl apply -f https://github.com/splunk/splunk-operator/releases/download/1.0.1/splunk-operator-install.yaml
```
  
After the Splunk Operator starts, you'll see a single pod running within your current namespace:
```bash
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
splunk-operator-75f5d4d85b-8pshn   1/1     Running   0          5s
```
  
![operator](/images/operator.png)

### 7. Deploy a Standalone deployment of Splunk Enterprise
  
Let’s ask our operator pod to build us a standalone demo instance to play with!  
  
```bash
kubectl -n default apply -f s1.yaml 
```
  
You will now see the sevices running:
```bash
kubectl get services

NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                               AGE
kubernetes                      ClusterIP   10.175.240.1     <none>        443/TCP                               41m
splunk-operator-metrics         ClusterIP   10.175.244.161   <none>        8383/TCP,8686/TCP                     4m29s
splunk-s1-standalone-headless   ClusterIP   None             <none>        8000/TCP,8088/TCP,8089/TCP,9997/TCP   70s
splunk-s1-standalone-service    ClusterIP   10.175.240.126   <none>        8000/TCP,8088/TCP,8089/TCP,9997/TCP   69s
```
  
![services](/images/services.png)
  

### 8. Splunk Web Access
Get your Pod details  
```bash
kubectl get pods            

NAME                                  READY   STATUS    RESTARTS   AGE
splunk-default-monitoring-console-0   0/1     Running   0          39s
splunk-operator-5845f6d45c-hk8q8      1/1     Running   0          6m40s
splunk-s1-standalone-0                1/1     Running   0          2m59s
```
  
You can use a simple network port forward to open port 8000 for Splunk Web access:  
```bash
kubectl port-forward splunk-s1-standalone-0 8000
```
  
To access our Spunk Operator for Kubernetes built instance, we will need to grab the secret which contains the HEC token and password, among other secrets the Operator syncs to the Splunk instance. 
```bash
kubectl -n default get secret splunk-default-secret -o yaml
```
  
Your output will be:
```yaml
apiVersion: v1
data:
  hec_token: FAKE--Ny0yNzJBLUFDNDAtNjRCQ0M2QzQ4RjI2
  idxc_secret: FAKE---DBveE43WklHQm9pZnF5VE5s
  pass4SymmKey: FAKE--Nzk0cGJEWkF2UFJ5VEtVV1VY
  password: FAKE--VJud1hDYTdlR0pTc2x6YWt3
  shc_secret: FAKE--ZV3VYOHRrbHBuWUFOeVpMZDBn
kind: Secret
```
`Note`: The output is encoded.  The above is for demonstration purposes only and the secrets are not real !
  
To `Decode` your passwords use the following:
```bash
kubectl get secret splunk-default-secret -o go-template=' {{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
```
  
Your Output will be decoded like this:
```bash
kubectl get secret splunk-default-secret -o go-template=' {{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
 hec_token: FAKE-D927-272A-AC40-64BCC6C48F26
idxc_secret: FAKE-X0oxN7ZIGBoifqyTNl
pass4SymmKey: FAKE-9794pbDZAvPRyTKUWUX
password: FAKEHQRnwXCa7eGJSslzakw
shc_secret: FAKEWuX8tklpnYANyZLd0g
```
`Note`: The output is now decoded.  The above is for demonstration purposes only and the secrets are not real !  You need to retrieve your own !
  
  
Log into Splunk Enterprise at http://localhost:8000 using the `admin` account with the password.
  


### Cleanup
Remember to destroy any resources you create once you are done with this tutorial. Run the destroy command and confirm with yes in your terminal.
  
```bash
terraform destroy
```

  
### Errors
If you get issues with your gcloud credentials, try this....
```bash
source /usr/local/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/path.zsh.inc
```
```bash
source /usr/local/Caskroom/google-cloud-sdk/latest/google-cloud-sdk/completion.zsh.inc
```