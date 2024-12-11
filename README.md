# FluxCD Gitops using image automation and image reflector controller

[Automatic Image Update to Git using Flux and GitHub Actions Blog](https://infracloud.io/blogs/automatic-image-update-to-git-using-flux-github-actions/) has a detailed walkthrough with explanation and this README.md is design to guide you through the setup and things you need look out for after you fork it.

### Prerequisites
1. **Flux CLI** - which can be downloaded from the [official docs](https://fluxcd.io/docs/cmd/)
2. **Kubernetes cluster** - For demo, minikube works.
3. GitHub account - which has GitHub Actions enabled.

### Environment setup details

1. **Hello Env Flask application**: A simple application displaying the environment name and release number.
2. **Two environments**: Staging and production, housed in separate namespaces.
3. **Continuous deployment**: Source code changes trigger automated build and deployment to the staging environment.
4. **Release tagging**: Tagging a release initiates automatic build processes and creates a pull request for production deployment.


#### Git repositories

Let's take a look at two Git repositories we will be using.

1. **Application repository** - [https://github.com/infracloudio/flux-helloenv-app](https://github.com/infracloudio/flux-helloenv-app)  - code and Kustomization.
   1. **Kustomization for environment separation**: Utilizing Kustomize to segregate staging and production environments. 
   2. **Structured repository**: The repository combines source code and Kubernetes manifests for a unified demo setup. There are many possible ways to [structure your git repositories.](https://fluxcd.io/flux/guides/repository-structure/)
   3. **CI and automation**: Leveraging GitHub Actions for [continuous integration](/ci-cd-consulting/) and automation.
2. **Management Repository**-  [https://github.com/infracloudio/flux-gitops-helloenv](https://github.com/infracloudio/flux-gitops-helloenv) - GitOps manifests.

### Flux bootstrap configuration

Beginning with an empty cluster, our initial task is to [bootstrap Flux itself](https://fluxcd.io/flux/cmd/flux_bootstrap/). Flux serves as the foundation upon which we'll bootstrap all other components.

In the following sections, we'll take a look at what things we need to change it get it working for you.
 
#### git-repo.yaml

This instructs Flux on how to interact with the Git repository where your application's source code resides. It contains the following configuration:

```yaml
  ref:
    branch: main
  secretRef:
    name: ssh-credentials
  url: ssh://git@github.com/infracloudio/flux-helloenv-app
```

`url:` Indicates the URL of the Git repository, **Change this and make sure it points to your forked repo url `url: ssh://git@github.com/<your_github_username>/flux-helloenv-app`**  

#### image-repo.yaml

This file is responsible for scanning the Docker image registry and fetching image tags based on the defined policy. Here's the configuration:  

```yaml
image: docker.io/shapai/helloenv
interval: 1m0s
```

`image:` Specifies the Docker image repository ( e.g docker.io/shapai/helloenv) to scan for image tags. Change this to have your relevant Docker image repository, where the CI job will build and push the image. This same registry needs to be updated in your forked CI file, where the CI build will push images.  

#### image-policy-staging.yaml 

This file defines the image tagging policy for the staging environment. Here's the configuration:  

```yaml
  filterTags:
    extract: $ts
    pattern: ^main-[a-f0-9]+-(?P<ts>[0-9]+)
  imageRepositoryRef:
    name: helloenv
  policy:
    numerical:
      order: asc
```
`imageRepositoryRef:` Refers to the image repository named helloenv. If you wish to be different you need to make sure the workflows within .github folder also points to same image repository.

#### image-policy-prod.yaml 

This file defines the image tagging policy for the production environment:  

```yaml
  imageRepositoryRef:
    name: helloenv
  policy:
    semver:
      range: '>=1.0.0'
```

`imageRepositoryRef:` Refers to the image repository named helloenv.  

### Setting up Flux

Now that we've seen all the configurations, we can proceed to the bootstrap command.

Note the `--owner` and `--repository` switches here: we are explicitly looking for the `${GITHUB_USER}/flux-gitops-helloenv` repo. Make sure you fork both repos under your user and follow on.

```sh
flux bootstrap github \ 
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=flux-gitops-helloenv \
  --path=./clusters/my-cluster/ \
  --branch=main \
  --read-write-key \
  --personal --private=false
```

The process of bootstrapping everything may take some time. To monitor the progress and ensure everything is proceeding as expected, we can utilize the `--watch` switch.

```sh
flux get kustomizations --watch
```

Verify that all Flux pods are in running state by running get pods.
```sh
$ kubectl -n flux-system get pods

NAME                                           READY   STATUS    RESTARTS   AGE
image-automation-controller-6c4fb698d4-zrp78   1/1     Running   0          29s
image-reflector-controller-5dfa39212d-hnnvj    1/1     Running   0          29s
kustomize-controller-424f5ab2a2-u2hwb          1/1     Running   0          29s
source-controller-2wc41z892-axkr1              1/1     Running   0          29s
```

Since our flux-helloenv-app repository is public, application will get deployed as part of the bootstrap step. You can check both staging and prod environments with following commands.
```sh
kubectl rollout status -n helloenv-staging deployments
watch kubectl get pods -n helloenv-staging

kubectl rollout status -n helloenv-prod deployments
watch kubectl get pods -n helloenv-prod
```

#### Setup Git authentication for Flux

After bootstrapping Flux, we will grant it write access to our GitHub repositories. This will allow Flux to update image tags in manifests, create pull requests etc.

The [`flux create secret git`](https://fluxcd.io/flux/cmd/flux_create_secret_git/) command creates an SSH key pair for the specified host and puts it into a named Kubernetes secret in Flux's management namespace (by default flux-system). The command also outputs the public key, which should be added to the forked repo's "Deploy keys" in GitHub.


```sh
GITHUB_USER=<your_github_username>
flux create secret git ssh-credentials \
  --url=ssh://git@github.com/${GITHUB_USER}/flux-helloenv-app
```

If you need to retrieve the public key later, you can extract it from the secret as follows:

```sh
kubectl get secret ssh-credentials -n flux-system -ojson \
  | jq -r '.data."identity.pub"' | base64 -d
```

Use the public key as a Deploy key in your fork of the flux-helloenv-app repo. Browse to the following URL, replacing `<your_github_username>` with your GitHub username: `https://github.com/<your_github_username>/flux-helloenv-app/settings/keys`.   

Click "Add deploy key" and paste the key data (starts with `ssh-<alg>...` ) into the contents. The name is arbitrary, we use helloenv-app-secret here.

### Accessing the application

The default image version can be checked by accessing our application using curl command. If you are using minikube, make sure you enable the ingress addon so the ingress functionality works.  

```sh
minikube addons enable ingress
```  

If you are running on the local cluster, you will have to add the minikube IP to your `/etc/hosts` file. The command `minikube ip` will give the IP of your cluster.  

For example, the entry in `/etc/hosts` file will look like this:

```sh
192.168.49.2 helloenv.prod.com helloenv.stage.com
```

You can check the ingress that is available for stage and prod.

```
$ kubectl get ingress --all-namespaces

NAMESPACE          NAME       CLASS    HOSTS                ADDRESS   PORTS   AGE
helloenv-prod      helloenv   <none>   helloenv.prod.com              80      9m22s
helloenv-staging   helloenv   <none>   helloenv.stage.com             80      9m21s
```

Now, you can access the application with following commands.

```sh
curl helloenv.prod.com

curl helloenv.stage.com
```

The result of these `curl` commands will show the current default image versions that the deployment is using.
