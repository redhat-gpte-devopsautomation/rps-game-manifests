# Roshambo Game k8s Manifests

The game is composed of two separate components; the image recognition service
provided by OpenShift AI (OpenShift DataScience), and the game components. 
These are deployed into separate Projects on the OpenShift cluster.

_NOTE: A user with the `cluster-admin` role assigned to them is required to
deploy the game. You also need the `oc` CLI installed on your machine._

## Deploy the DataScience Project

Begin by installing the Red Hat OpenShift DataScience operator via OperatorHub
in the OpenShift Web Console:

<div align="center">
<img width="85%" src="screenshots/00-datascience.png" alt="OpenShift Data Science Installed" title="OpenShift DataScience Installed"</img>
</div>

Wait 3 or 4 minutes until the operator is ready, to give time to register all controllers and CRDs.

Then you'll need to login into the OpenShift cluster from a terminal and run the following command to install the AI model:

```bash
# Login as a valid OpenShift user with cluster-admin permissions
oc login

cd ai
make
```

After this, the model is installed, the AI service is running in the `rps-ai-service` namespace, and you can continue with the installation process of the game.

## Deploy the Game Project

You can deploy the game using regular CLI commands, or using GitOps.

### Using OpenShift/Kube CLI

You'll need `helm` 3.12+, and `oc`:

```bash
export NAMESPACE="rps-game"

# Create the namespace for the application
oc new-project $NAMESPACE

# Generate and apply YAML manifests or use helm install
helm template helm/ --namespace $NAMESPACE | oc apply -f -
```

### Using OpenShift GitOps (Argo CD)

This requires an OpenShift cluster that you have access to a user with `cluster-admin` access.

#### Install OpenShift GitOps

1. Login to the OpenShift cluster's web console as the user with `cluster-admin` permission.
1. Select the **Administrator** perspective.
1. Expand the **Operators** section and select **Operator Hub**.
1. Type `openshift gitops` in the search box and click the *Red Hat OpenShift GitOps* operator from the list.
1. Follow the prompts to install the `stable` version of the operator.

Once the Operator is ready you can find the link to Argo CD (provided and managed by OpenShift GitOps) in the **Application Launcher**. The password to login as the `admin` user can be found in the `openshift-gitops-cluster` Secret in the `openshift-gitops` namespace as shown:

<div align="center">
<img width="85%" src="screenshots/01-install-gitops.png" alt="OpenShift GitOps Installed" title="OpenShift GitOps Installed"</img>
</div>

#### Deploy the Roshambo Application

1. Login to the Argo CD instance using the username `admin` and the password you obtained from the `openshift-gitops-cluster` Secret in the `openshift-gitops`.
    ![Argo CD Login](screenshots/02-argo-login.png)
1. Click the **New App** button. An overlay appears.
1. Click the **Edit as YAML** button in the overlay.
1. Paste in the following YAML to deploy the game using non-production values:
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: rps-game
    spec:
      destination:
        name: ''
        namespace: rps-game
        server: 'https://kubernetes.default.svc'
      source:
        path: helm
        repoURL: 'https://github.com/redhat-developer-demos/rps-game-manifests'
        targetRevision: HEAD
        helm:
          valueFiles:
            - values.yaml
      project: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    ```
1. Click **Save**, then click **Create**.
    ![Argo CD Application](screenshots/03-argo-app.png)
1. Label the newly created namesapce to provide GitOps with admin rights to manage it and deploy resources within it:
    ```bash
    oc label namespace rps-game argocd.argoproj.io/managed-by=openshift-gitops
    ```
**NOTE:** use the *values.production.yaml* file if you need to allocate more resources for a large audience!
