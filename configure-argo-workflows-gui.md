# Configure Argo Workflows engine locally on Windows

1. Install Rancher Desktop for Windows -> [Rancher desktop](https://rancherdesktop.io/)
2. Enable Kubernetes (Apply & Restart) from Settings → Kubernetes
3. Open CMD or Powershell and run the following commands
    - `kubectl config get-contexts` → Verify that rancher-desktop is listed here
    - `kubectl config use-context rancher-desktop`
    - `kubectl config current-context` → rancher-desktop should be the current context
4. Install Argo CLI
    - Navigate to releases → [Releases](https://github.com/argoproj/argo-workflows/releases)  and expand the Assets section of the latest release
    - Download **argo-windows-amd64.exe.gz** file
    - Extract the context with **7zip** or a similar tool
    - Create folder **C:\ArgoCli** or similar and move the .exe file to that location
    - Rename the file to **argo.exe**
    - Open the environment variables setting and add the folder under System Variables > PATH
    - To test, open cmd and type  `argo version`
5. Installing Argo Controller and UI - Before installing the Argo Workflow resources, you need to create an `argo` namespace by running the following commands in cmd/powershell
    - `kubectl create ns argo`
    - `kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.4.11/install.yaml`
6. Access Argo Workflow UI - To access the Argo Workflows UI, you will need to expose it. This can be done in multiple ways, but for this tutorial, use the port-forwarding method:
    - `kubectl -n argo port-forward svc/argo-server 2746:2746`
7. Open your browser and go to https://127.0.0.1:2746. You will be redirected to a page for authentication.
8. If unauthorized access message appears then execute:
    - `kubectl -n argo exec argo-server-${POD_ID} -- argo auth token`
    - Replace **${POD_ID}** with your actual pod. After that copy the generated token and place it inside of the token field on the Argo Workflows UI Login screen.
10. As a final step, we need to install all of the required Argo event-related CRD (custom resouce definition). So the easier approach is just to create the suggested namespace by the docs: 
    - `kubectl create namespace argo-events`
    - Once we have this namespace we need to execute these two commands:
    - ```
      kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml
      # Install with a validating admission controller
      kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install-validating-webhook.yaml
      ```
 
    - They will create cluster-wide CRD which will include Event bus, Event source, Sensor, and all of the required KINDs that we will need later on. For more information, you can refer here: [Installation - Argo Events - The Event-Based Dependency Manager for Kubernetes](https://argoproj.github.io/argo-events/installation/) 
