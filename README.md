# Kubernetes Authentication with Pinniped
Clone the repository to your local machine and follow below steps from the cloned directory.

## Setup kind cluster with Pinniped Supervisor (with Gitlab OIDC)
This cluster will be used for shared-services setup such as Pinniped Supervisor. All users of one or many workload cluster(s) will authenticate against Pinniped Supervisor running on this shared-services cluster.

1. Create ingress-ready kind cluster 
    ```sh
    kind create cluster --config=./kind-config.yaml
    ```
2. Setup Contour
    ```sh
    kubectl apply -f https://projectcontour.io/quickstart/contour.yaml

    kubectl patch daemonsets -n projectcontour envoy -p '{"spec":{"template":{"spec":{"nodeSelector":{"ingress-ready":"true"},"tolerations":[{"key":"node-role.kubernetes.io/master","operator":"Equal","effect":"NoSchedule"}]}}}}'

    # Confirm all pods are in "Running" state
    kubectl get pods -A
    ```
3. Install Supervisor
    ```sh
    cd pinniped-supervisor

    kapp deploy --app pinniped-supervisor --file https://get.pinniped.dev/v0.12.0/install-pinniped-supervisor.yaml

    # Confirm all pods in pinniped-supervisor namespace are in "Running" state
    kubectl -n pinniped-supervisor get pods
    ```
4. Configure Supervisor as an OIDC Issuer
    ```sh
    cd pinniped-supervisor

    # a. Expose the Supervisor app as a Service. Note that we will expose this Service using an Ingress to outside world
    kubectl -n pinniped-supervisor expose deployment pinniped-supervisor --port=80 --target-port=8080 --dry-run=client -oyaml > pinniped-supervisor-svc.yaml

    kubectl apply -f pinniped-supervisor-svc.yaml

    # b. We will need TLS certificate to expose Supervisor service as an Ingress over HTTPs. 
    # Note that Supervisor expects SAN certificate i.e. Domain name should also be part of SAN entries. 
    # Follow below steps to generate the self-signed SAN certificate.
    # From the git cloned directory, run below commands
    cd certs
    ./gencert.sh <DOMAIN_NAME> # e.g. ./gencert.sh supervisor.k8slabs.com

    # c. Confirm the server certificate has SAN entry
    openssl x509 -in server.crt -text -noout | grep -A1 "Subject"

    # d. Now create TLS secret in pinniped-supervisor namespace
    kubectl create secret tls supervisor-tls --namespace pinniped-supervisor --key=./server.key --cert=./server.crt --dry-run=client -oyaml > supervisor-cert.yaml

    kubectl apply -f supervisor-cert.yaml

    # e. We will now setup Ingress to expose Supervisor service to outside world
    # You would need to setup Domain name and DNS records to access Supervisor Service using domain name
    # From the git cloned directory, run below commands

    cd pinniped-supervisor
    # f. Replace "hosts" entry in pinniped-supervisor-ingress.yaml file with your own domain name. Make sure that the TLS certificate was generated for this exact domain name.
    kubectl apply -f pinniped-supervisor-ingress.yaml 

    # g. Next we will configure Supervisor to act as an OIDC provider by creating FederationDomain resources in the same namespace where the Supervisor app is installed.
    # Replace "issuer" entry in pinniped-supervisor-federationdomain.yaml with your own domain name.
    kubectl apply -f pinniped-supervisor-federationdomain.yaml

    # h. Next we will configuer an OIDCIdentityProvider for the Supervisor with GitLab OIDC

    # i. Follow steps on this [page](https://pinniped.dev/docs/howto/configure-supervisor-with-gitlab/#:~:text=Configure%20your%20GitLab%20Application) to configure your GitLab application.

    # Create an OIDCIdentityProvider in the same namespace as the Supervisor
    # Replace "clientID" and "clientSecret" with your own GitLab application credentials
    kubectl apply -f pinniped-supervisor-oidcprovider.yaml

    # Once your OIDCIdentityProvider has been created, you can validate your configuration by running:
    kubectl describe OIDCIdentityProvider -n pinniped-supervisor gitlab
    # Look at the status field. If it was configured correctly, you should see phase: Ready
    ```

## Setup first workload cluster with Pinniped Concierge
Users of this workload cluster will authenticate against Pinniped Supervisor setup on shared-services cluster.

1. Create kind workload-1 cluster
    ```sh
    kind create cluster --name workload-1
    # change context to workload-1 cluster if not done
    kubectl config use-context kind-workload-1
    ```
2. Install Concierge
    ```sh
    kapp deploy --app pinniped-concierge --file https://get.pinniped.dev/v0.12.0/install-pinniped-concierge.yaml
    ```
3. Configure Concierge JWT Authentication describing how to validate tokens from your Supervisor's FederationDomain
    ```sh
    # From the git cloned directory, run:
    cd pinniped-concierge    
    # Since we are using self-signed Supervisor certificate, we need to copy CA certificate used to sign Supervisor certificate as base64-encoded in "certificateAuthorityData" in jwt-authenticator-with-supervisor.yaml
    # To base64 encode CA certificate, run:
    cat ../certs/ca.crt | base64
    kubectl apply -f jwt-authenticator-workload1.yaml
    ```
4. Login to the cluster
    ```sh
    # Generate a Pinniped-compatible kuebconfig file
    # From the git cloned directory, run:
    cd workdload1-config
    pinniped get kubeconfig > pinniped-kubeconfig.yaml

    # Use the generated kubeconfig with kubectl to access the cluster
    kubectl get namespaces --kubeconfig ./pinniped-kubeconfig.yaml
    # Once the user completes authentication, the kubectl command will automatically continue and complete the user's requested command. 

    #Pinniped provides authentication (usernames and group memberships) but not authorization.  Ff the user gets an access denied error, then they may need authorization to list namespaces. For example, an admin could grant the user "edit" access to all cluster resources via the user's username:
    kubectl create clusterrolebinding my-user-can-edit --clusterrole edit --user my-username@example.com

    # Temporary session credentials such as ID, access, and refresh tokens are stored in:
    ~/.config/pinniped/sessions.yaml
    ```

## Setup second workload cluster with Pinniped Concierge
1. Create kind workload-2 cluster
    ```sh
    kind create cluster --name workload-2
    # change context to workload-1 cluster if not done
    kubectl config use-context kind-workload-2
    ```
2. Install Concierge
    ```sh
    kapp deploy --app pinniped-concierge --file https://get.pinniped.dev/v0.12.0/install-pinniped-concierge.yaml
    ```
3. Configure Concierge JWT Authentication describing how to validate tokens from your Supervisor's FederationDomain
    ```sh
    # From the git cloned directory, run:
    cd pinniped-concierge    
    # Since CA certificate value is already replaced in above step, run:
    # The only difference in this file compared to workload1 file is the "audience" field
    kubectl apply -f jwt-authenticator-workload2.yaml
    ```
4. Login to the cluster
    ```sh
    # Generate a Pinniped-compatible kuebconfig file
    # From the git cloned directory, run:
    cd workdload2-config
    pinniped get kubeconfig > pinniped-kubeconfig.yaml

    # Use the generated kubeconfig with kubectl to access the cluster
    kubectl get namespaces --kubeconfig ./pinniped-kubeconfig.yaml
    # If the user is already authenticated with Supervisor then there should not be login prompt. 
    # If the user gets an access denied error, then they may need authorization to list namespaces. For example, an admin could grant the user "edit" access to all cluster resources via the user's username:
    kubectl create clusterrolebinding my-user-can-edit --clusterrole edit --user my-username@example.com

    # Temporary session credentials such as ID, access, and refresh tokens are stored in:
    ~/.config/pinniped/sessions.yaml
    # Short-lived certificates to access workload clusters are stored in:
    ~/.config/pinniped/credentials.yaml
    ```