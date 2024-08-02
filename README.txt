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
* Kubernetes manifests can be specified in several ways:
    * kustomize applications
    * helm charts
    * jsonnet files
    * Plain directory of YAML/json manifests
    * Any custom config management tool configured as a config management plugin
* automates the deployment of the desired application states in the specified target environments
    * can track updates to branches, tags, or pinned to a specific version of manifests at a Git commit
* implemented as a Kubernetes controller
    * continuously monitors running applications and compares the current, live state against the desired target state (Git repo)
* deployed application whose live state deviates from the target state is considered `OutOfSync`
    * argoCD provides facilities to automatically or manually sync the live state back to the desired target state
* helm
    * values
        * As of v2.6, values files can be sourced from a separate repository than the Helm chart by taking advantage of multiple sources for Applications
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