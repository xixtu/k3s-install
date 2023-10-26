1. Install K3S Nodes

    a. Prerequisite:

    ```bash
    dnf config-manager --setopt strict=False --setopt best=False --save
    ```

    b. Server Node installation

      - Install command:
          ```bash
          curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" sh -s - \
          --tls-san <CLUSTER_FQDN> \
          --write-kubeconfig-mode 644 \
          --disable traefik
          ```
      - Install command for the next server nodes:
          ```bash
          curl -sfL https://get.k3s.io | K3S_TOKEN=<TOKEN> sh -s - server \
              --server https://<ip or hostname of server1>:6443 \
              --tls-san=<CLUSTER_FQDN>
          ```

      - DNS configuration (if caching is enabled):
         ```bash
         echo "nameserver <xx.xx.xx.xx>" | sudo tee /etc/k3s-resolv.conf

         echo 'kubelet-arg:' | sudo tee -a /etc/rancher/k3s/config.yaml
         echo '- "resolv-conf=/etc/k3s-resolv.conf"' | sudo tee -a /etc/rancher/k3s/config.yaml

         systemctl restart k3s
         kubectl rollout restart deployment coredns -n kube-system
         ```

    - Registry configuration

        Documentation: https://docs.k3s.io/installation/private-registry
        ```yaml
        cat << 'EOF' > /etc/rancher/k3s/registries.yaml
        configs:
          "<REGISTRY_ADDRESS>":
            tls:
              ca_file: <PATH_TO_CA.CRT>
        EOF
        ```
    - Taint server node to avoid workload scheduling
        ```bash
        kubectl get nodes
        kubectl taint node <K3S_MASTER_NODE_NAME> node-role.kubernetes.io/master=effect:NoSchedule
        ```

    - Save token for Agent Node installation   
        ```bash
        cat /var/lib/rancher/k3s/server/token
        ```

    b. Agent Node installation

    - Install command:
        ```bash
        curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent" \
        K3S_URL=https://<CLUSTER_FQDN>:6443 \
        K3S_TOKEN="<TOKEN>" \
        sh -s -
        ```
   

2. Copy Kubeconfig file on local machine

      ```bash
      scp <USER>@<K3S_SERVER>:/etc/rancher/k3s/k3s.yaml ~/.kube/config-k3s
      sed -i 's/127.0.0.1/<CLUSTER_FQDN>/g' ~/.kube/config-k3s
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

4. Install Kong

   a. Get the helm values file and update its content:
   ```bash
   wget https://raw.githubusercontent.com/remblondel/k3s-install/cluster-install/kong-values.yaml
   sed -i 's/CLUSTER_APPS_FQDN/<CLUSTER_APPS_FQDN>/g' kong-values.yaml
   ```

   b. Install the helm chart
   ```bash
   helm repo add kong https://charts.konghq.com
   helm repo update
   helm install kong kong/kong --version 2.26.3 -f kong-values.yaml -n kong --create-namespace
   ```

5. Install and access Kubernetes Dashboard

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
  
     d. Configure access using Kong Ingress:
   
	- Annotate the kubernetes-dashboard to tell Kong Proxy to use HTTPS to talk to the upstream service
        ```bash
        kubectl annotate service kubernetes-dashboard -n kubernetes-dashboard konghq.com/protocol=https
        ```
 	- Create the ingress object:

        ```yaml
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
6. Install Portainer Agent

   After installing the portainer agent, launch the following command to avoid single point of failure:
   ```
   kubectl scale deployment/portainer-agent --replicas=2 -n portainer
   ```
   
7. Install Kube-Prometheus-Stack

   a. Get the helm values file and update its content:
   ```bash
   wget https://raw.githubusercontent.com/remblondel/k3s-install/cluster-install/kube-prometheus-stack-values.yaml
   sed -i 's/CLUSTER_APPS_FQDN/<CLUSTER_APPS_FQDN>/g' kube-prometheus-stack-values.yaml
   ```

   b. Install the helm chart
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n tooling-system --create-namespace --version 51.9.4 -f kube-prometheus-stack-values.yaml
   ```
