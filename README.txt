# argoCD-workshop
* references
    * https://argo-cd.readthedocs.io/en/stable/
    * https://www.atlassian.com/git/tutorials/gitops
    * https://about.gitlab.com/topics/gitops/
    * https://codefresh.io/learn/gitops/
    * https://chatgpt.com/

## preface
* goals of this workshop
* workshop plan
    * installing argocd
        1. run k8s locally with docker desktop
        1. install argocd
            ```
            kubectl create namespace argocd
            kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
            ```
        1. log into argocd
            * `kubectl port-forward svc/argocd-server -n argocd 8080:443`
            * `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo`
                * it is password
            * https://localhost:8080
                * login: admin
                * password from previous substep
                    * once you changed the password => delete the argocd-initial-admin-secret from the Argo CD namespace
                        * no other purpose than to store the initially generated password
        1. verify that there is nothing in `Applications`
    * adding app
        1. checkout this app: https://github.com/mtumilowicz/helm-workshop
            * create 1.1 docker image with `greeting` endpoint
            * create 1.2 docker image with `greeting2` endpoint
            * command: `./gradlew bootBuildImage`
        1. `kubectl apply -f greeting-app/application.yaml`
        1. verify that there is app added in `Applications`
        1. http://localhost:31234/app/greeting
            * should return `greeting`
    * sync
        1. `push commit that changes deployment.image.version`
            * from `1.1` and `1.2`
        1. wait few seconds
        1. verify that argocd is in sync for that App
        1. http://localhost:31234/app/greeting
            * should return `greeting2`

## gitOps
* problem: configuration still feels disconnected from the live system
    * technologies like Docker Containers, Ansible, Terraform, and Kubernetes utilize static declarative configuration files
    * configuration files are naturally added to Git for tracking and review
        * stored in a central location, documented and accessible by many team members
    * configuration merged => live system is manually updated
        * to match the state of the static repo
    * solution: exact problem GitOps solves
* is code-based infrastructure and operational procedures that rely on Git as a source control system
* evolution of Infrastructure as Code (IaC) and a DevOps best practice
* leverages Git as the single source of truth
    * mechanism for creating, updating, and deleting system architecture
    * any configuration drift, such as manual changes or errors, is overwritten by GitOps automation
        * environment converges on the desired state defined in Git
    * live syncing pull request workflow is the core essence of GitOps
* ensures that a system’s cloud infrastructure is immediately reproducible based on the state of a Git repository
* GitOps "operator"
    * mechanism that sits between the pipeline and the orchestration system
    * pull request starts the pipeline that then triggers the operator
    * operator examines the state of the repository and then start of the orchestration and syncs them
* vs DevOps
    * GitOps is a branch of DevOps
    * in GitOps: Git repository is the source of truth for the deployment state
        * uses Git workflows
    * in DevOps: application or server configuration files is the source of truth for the deployment state
        * may not necessarily be centralized in a single repository
        * uses configuration management tools (like Ansible)
        * continuous reconciliation may not be a primary focus
* cons
    * automating git commits may create conflicts
        * multiple CI processes can cause conflicts when writing to the same repository
            * example: when multiple CI processes try to push changes to the same branch
                * Git cannot automatically resolve the conflicts, leading to potential integration issues
        * solution: using multiple repositories
            * example: one for each namespace
    * too many git repositories
        *  number of GitOps repositories typically increases with each new environment or application
        * solution: use fewer Git repositories (e.g., one per cluster)
    * doesn’t centrally manage secrets
        * secrets are manage outside the standard CI/CD process

## argoCD
* is a declarative, GitOps continuous delivery tool for Kubernetes
* follows GitOps pattern of using Git repositories as the source of truth for defining the desired application state
    * Kubernetes manifests can be specified in several ways
        * helm
            * values
                * As of v2.6, values files can be sourced from a separate repository than the Helm chart (multiple sources for Applications)
                    * most common scenario
                        * use an external/public Helm chart and override the Helm values with your own local values
                            * cloning the Helm chart => duplication + need to monitor it manually for upstream changes
                    * is a beta feature
                * declarative syntax
                    ```
                    source:
                      helm:
                        valueFiles:
                        - values-production.yaml
                    ```
            * release name
                * overriding the Helm release name might cause problems when the chart you are deploying is using the app.kubernetes.io/instance label
                * declarative syntax
                    ```
                    source:
                        helm:
                          releaseName: myRelease
                    ```
            * hooks
                * Argo CD cannot know if it is running a first-time "install" or an "upgrade" - every operation is a "sync'.
                    * apps that have pre-install and pre-upgrade will have those hooks run at the same time
        * other: kustomize, jsonnet, plain directory of YAML/json manifests, custom
* components
    * API Server
        * provides endpoints for managing and querying applications, repositories, projects, and other resource
    * Repository Server
        * manages interactions with Git repositories
            * argocd-repo-server is responsible for cloning Git repository, keeping it up to date and generating manifests using the appropriate tool
                * clones the repository into /tmp (or the path specified in the TMPDIR env variable)
                    * Pod might run out of disk space if it has too many repositories or if the repositories have a lot of files
                        * mount a persistent volume
                * Every 3m (by default) Argo CD checks for changes to the app manifests.
                    * assumes by default that manifests only change when the repo changes
                    * change this duration using the `timeout.reconciliation` setting in the argocd-cm ConfigMap
                    * To eliminate this delay from polling, the API server can be configured to receive webhook events
                        * supported Git webhook notifications from GitHub, GitLab, Bitbucket, Bitbucket Server and Gogs
    * Application Controller
        * specific controller that handles the reconciliation of applications
* automates the deployment of the desired application states in the specified target environments
    * can track updates to branches, tags, or pinned to a specific version of manifests at a Git commit
* implemented as a Kubernetes controller
    * continuously monitors running applications and compares the current, live state against the desired target state (Git repo)
        * deployed application whose live state deviates from the target state is considered `OutOfSync`
            * argoCD provides facilities to automatically or manually sync the live state back to the desired target state
                * automated sync policy
                    * triggered when it detects differences between the desired manifests in Git, and the live state in the cluster
                    * benefit: CI/CD pipelines no longer need direct access to the Argo CD API server to perform the deployment
                        * pipeline makes a commit and push to the Git repository with the changes to the manifests
                    * pruning
                        * default: automated sync will not delete resources when Argo CD detects the resource is no longer defined in Git
                    * self-healing
                        * default: changes that are made to the live cluster will not trigger automated sync
                        * selfHeal flag is set to true => sync will be attempted again after self heal timeout (5 seconds by default)
                            * controlled by `--self-heal-timeout-seconds`
* resource objects
    * can be defined declaratively using Kubernetes manifests
    * can be updated using kubectl apply, without needing to touch the argocd command-line tool
        * `kubectl apply -n argocd -f application.yaml`
    * Application
        * represents a deployed application instance in an environment
        * example
            ```
            apiVersion: argoproj.io/v1alpha1
            kind: Application
            metadata:
              name: greeting-app
              namespace: argocd
            spec:
              destination: // reference to the target cluster and namespace
                namespace: default
                server: https://kubernetes.default.svc
              source: // reference to the desired state in Git
                path: greeting-app/helm
                repoURL: https://github.com/mtumilowicz/argocd-workshop.git
                targetRevision: HEAD
              project: default
              syncPolicy:
                automated:
                  prune: true
                  selfHeal: true
            ```
        * Argo CD is able to manage itself since all settings are represented by Kubernetes manifests
            * create Kustomize based application which uses base Argo CD manifests from https://github.com/argoproj/argo-cd and apply required changes on top
    * AppProject
        * epresents a logical grouping of applications
        * example: frontend, backend, data-services, and infrastructure
            * if unspecified, an application belongs to the default project
                * created automatically and by default, permits deployments from any source repo, to any cluster, and all resource Kinds
        * logical grouping of applications, which is useful when Argo CD is used by multiple teams
        * enforce role-based access control (RBAC)
            * ensures that team A cannot accidentally modify the applications managed by team B
                * restrict what may be deployed (trusted Git source repositories)
                * restrict where apps may be deployed to (destination clusters and namespaces)
                * restrict what kinds of objects may or may not be deployed (e.g. RBAC, CRDs, DaemonSets, NetworkPolicy etc...)
                * defining project roles to provide application RBAC (bound to OIDC groups and/or JWT tokens)
    * other: ConfigMap, Secret, Cluster etc
* can display a badge with health and sync status for any application
* notification subscriptions
    * example: slack
        ```
        apiVersion: argoproj.io/v1alpha1
        kind: Application
        metadata:
          annotations:
            notifications.argoproj.io/subscribe.on-sync-succeeded.slack: my-channel1;my-channel2
        ```
* best practices
    * clean separation of application code vs. application config
        * separate Git repository to hold your Kubernetes manifests
        * keeping the config separate from your application source code
        * pros
            * modify just the manifests without triggering an entire CI build
                * example: bump the number of replicas in a Deployment spec
            * much cleaner Git history
            * separation of access
                * developers may not necessarily be the same people who can/should push to production environments
            * pushing manifest changes to the same Git repository can trigger an infinite loop
    * ensuring manifests at git revisions are truly immutable
        *  it is possible for manifests to change without source code change
            * example: caused by changes made to an upstream helm/kustomize repository
                ```
                resources:
                - github.com/argoproj/argo-cd//manifests/cluster-install
                ```
                vs
                ```
                bases:
                - github.com/argoproj/argo-cd//manifests/cluster-install?ref=v0.11.1
                ```