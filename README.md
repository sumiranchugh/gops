# Secure Supply Chain Setup

This repository is to setup a secure software supply chain across multiple OpenShift clusters managed by ACM


## Prerequisities

This process assumes you already have a ACM cluster up and running

You will need openshift client `oc`

### Step 1 - Install Openshift GitOps

The first step is to install OpenShift Gitops Operator on the ACM cluster and create an Instance of ARGO CD that will install all the required Operators for Secure Software Supply Chain


1. Login to the cluster

```
oc login --token=<token> --server=<server> 
```

2. Install OpenShift Gitops Operator

```
cat << EOF | oc apply --filename -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
    name: openshift-gitops-operator
    namespace: openshift-operators
spec:
    channel: latest
    installPlanApproval: Automatic
    name: openshift-gitops-operator
    source: redhat-operators
    sourceNamespace: openshift-marketplace
    config:
        env:
        - name: ARGOCD_CLUSTER_CONFIG_NAMESPACES
          value: openshift-gitops
EOF
```

3. Update Openshift gitops instance RBAC

```
cat << EOF | oc apply --filename -
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata: 
    name: openshift-gitops
    namespace: openshift-gitops
spec:
    kustomizationBuildOptions: --enable-helm --enable-alpha-plugins
    rbac:
        defaultPolicy: role:admin
        scopes: '[groups]'
        applicationSet: {}
        resourceExclusions: |
            - kinds:
                - TaskRun
        server:
            insecure: true
            route:
                enabled: true
                tls:
                    insecureEdgeTerminationPolicy: Redirect
                    termination: edge
        sso:
            dex:
                openshiftOAuth: true
            provider: dex
EOF            
```

4. Create application set to install required operators on ACM cluster

```
cat << EOF | oc apply --filename -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
    name: secure-supply-chain-acm
    namespace: openshift-gitops
spec:
    goTemplate: true
    goTemplateOptions: ["missingkey=error"]
    generators:
        - git:
            repoURL: https://github.com/sumiranchugh/gops.git
            revision: main
            directories:
                - path: gitops*
    template:
        metadata:
            name: '{{.path.basename}}'
            spec:
                project: "default"
                source: 
                    repoURL: https://github.com/sumiranchugh/gops.git
                    targetRevision: main
                    path: '{{.path.path}}'
                destination:
                    server: https://kubernetes.default.svc
                    namespace: gops
                syncPolicy:
                    automated:
                        prune: true
                    syncOptions:
                        - CreateNamespace=true
EOF


```