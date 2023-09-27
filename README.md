1. Install K3S

     ```bash
     curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" sh -s - \
     --tls-san k3s.local \
     --write-kubeconfig-mode 644 \
     --disable traefik
     ```

2. Copy Kubeconfig file on local machine

      ```bash
      scp user@k3s.local:/etc/rancher/k3s/k3s.yaml ~/.kube/config-k3s
    	sed -i 's/127.0.0.1/k3s.local/g' ~/.kube/config-k3s
      ```

3. Install kubectl on local machine

	  Documentation: https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/

    ```bash
    curl -LO https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    sudo mv ./kubectl /usr/local/bin/kubectl
  	kubectl version --client
    ```

    ```bash
  	export KUBECONFIG=~/.kube/config-k3s
  	kubectl get nodes
    ```

4. Install and access Kubernetes Dashboard

    a. Prerequisite:
   
      Apply the following:
   
	```yaml
	cat <<EOF | kubectl apply -f -
	apiVersion: v1
	kind: ServiceAccount
	metadata:
	  name: admin-user
	  namespace: kube-system
	EOF
 	```
 	```yaml
 	cat <<EOF | kubectl apply -f -
	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRoleBinding
	metadata:
	  name: admin-user
	roleRef:
	  apiGroup: rbac.authorization.k8s.io
	  kind: ClusterRole
	  name: cluster-admin
	subjects:
	- kind: ServiceAccount
	  name: admin-user
	  namespace: kube-system
	EOF
	```
 	```yaml
 	cat <<EOF | kubectl apply -f -
	apiVersion: v1
	kind: Secret
	metadata:
	  name: admin-token
	  namespace: kube-system
	  annotations:
	    kubernetes.io/service-account.name: admin-user
	type: kubernetes.io/service-account-token
	EOF
	```

    b. Install:

     Documentation: https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
  
      ```bash
      TOKEN=$(kubectl -n kube-system describe secret admin-token | awk '$1=="token:"{print $2}') # Used later
      kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
      ```

    c. (Optional) Access through Kubernetes API:

      ```yaml
      kubectl port-forward -n kubernetes-dashboard svc/kubernetes-dashboard 8443:443
      ```

     - Go to https://127.0.0.1:8443/
     - Paste the `$TOKEN` value
  
     d. (Only if NGINX Ingress Controller is not installed) Change the kubernetes-dashboard service to LoadBalancer type:

	```bash
	kubectl edit service kubernetes-dashboard -n kubernetes-dashboard
	```
	- Change `spec.type` to `LoadBalancer`
	- Go directly to step 7

6. (Not needed) Install NGINX Ingress Controller

    Documentation: https://kubernetes.github.io/ingress-nginx/deploy/#quick-start
  
    ```bash
    helm upgrade \
      --install ingress-nginx ingress-nginx \
      --repo https://kubernetes.github.io/ingress-nginx \
      --namespace ingress-nginx \
      --create-namespace \
      --set controller.kind=DaemonSet \
      --set controller.hostNetwork=true \
      --set controller.hostPort.enabled=true
    ```

7. Access Kubernetes Dashboard through an ingress:
	
    - Create a DNS record on your machine pointing to the K3S VM (e.g.: `dashboard.k3s.local`)
      
  Apply the following:

   ```bash
	cat <<EOF | kubectl apply -f -
	apiVersion: networking.k8s.io/v1
	kind: Ingress
	metadata:
	  name: kubernetes-dashboard
	  namespace: kubernetes-dashboard
	spec:
	  rules:
	  - host: "dashboard.k3s.local"
	    http:
	      paths:
	      - pathType: Prefix
	        path: /
	        backend:
	          service:
	            name: kubernetes-dashboard
	            port:
	              number: 443
	EOF
   ```

8. Install Kong

   ```bash
   helm pull kong/kong --untar --version 2.26.3
   cd kong
   wget https://raw.githubusercontent.com/remblondel/k3s-install/main/kong-values.yaml
   helm install kong -n kong ./ -f kong-values.yaml --create-namespace
   ```
