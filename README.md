# Managing Helm releases the GitOps way


<br/>

### Run in minikube


```
$ {
minikube --profile my-profile config set memory 4096
minikube --profile my-profile config set cpus 2

// $ minikube --profile my-profile config set vm-driver virtualbox
minikube --profile my-profile config set vm-driver docker

minikube --profile my-profile config set kubernetes-version v1.16.1
minikube start --profile my-profile
}
```

<br/>

    // To delete
    // $ minikube --profile my-profile stop && minikube --profile my-profile delete

<br/>

    $ kubectl version --short
    Client Version: v1.18.1
    Server Version: v1.16.1

<br/>

    // FluxCtl Installation
    $ curl -sL https://fluxcd.io/install | sh && chmod +x $HOME/.fluxcd/bin/fluxctl && sudo mv $HOME/.fluxcd/bin/fluxctl /usr/local/bin/


<br/>

### With HELM3

    $ helm repo add fluxcd https://charts.fluxcd.io
    
    $ kubectl create namespace fluxcd
    
    $ helm upgrade -i flux fluxcd/flux --wait \
    --namespace fluxcd \
    --set git.url=git@github.com:webmakaka/helm-operator-get-started \
    --set git.timeout=3m \
    --set git.pollInterval=1m \
    --set resources.requests.cpu=500m \
    --set resources.requests.memory=500Mi \
    --set sync.timeout=3m \
    --set helm.versions=v3


    $ kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml


    $ helm upgrade -i helm-operator fluxcd/helm-operator --wait \
    --namespace fluxcd \
    --set git.ssh.secretName=flux-git-deploy \
    --set helm.versions=v3
    
    $ watch kubectl -n fluxcd get pods
    
    $ fluxctl --k8s-fwd-ns fluxcd identity 
     
     GITHUB --> Project (flux-get-started) --> SETTINGS --> DELPLOY KEYS

    + allow write access
    
<br/>

    $ fluxctl --k8s-fwd-ns=fluxcd sync

    $ kubectl -n fluxcd logs deployment/flux -f


<br/>


```
$ docker login
$ cd hack 
$ ./ci-mock.sh -r stefanprodan/podinfo -b dev

Sending build context to Docker daemon  4.096kB
Step 1/15 : FROM golang:1.13 as builder
....
Step 9/15 : FROM alpine:3.10
....
Step 12/15 : COPY --from=builder /go/src/github.com/stefanprodan/k8s-podinfo/podinfo .
....
Step 15/15 : CMD ["./podinfo"]
....
Successfully built 71bee4549fb2
Successfully tagged stefanprodan/podinfo:dev-kb9lm91e
The push refers to repository [docker.io/stefanprodan/podinfo]
36ced78d2ca2: Pushed
```

Inside the *charts* directory there is a podinfo Helm chart.
Using this chart I want to create a release in the `dev` namespace with the image I've just published to Docker Hub.
Instead of editing the `values.yaml` from the chart source, I create a `HelmRelease` definition (located in /releases/dev/podinfo.yaml):

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: podinfo-dev
  namespace: dev
  annotations:
    fluxcd.io/automated: "true"
    filter.fluxcd.io/chart-image: glob:dev-*
spec:
  releaseName: podinfo-dev
  chart:
    git: git@github.com:fluxcd/helm-operator-get-started
    path: charts/podinfo
    ref: master
  values:
    image:
      repository: stefanprodan/podinfo
      tag: dev-kb9lm91e
    replicaCount: 1
```

Flux Helm release fields:

* `metadata.name` is mandatory and needs to follow Kubernetes naming conventions
* `metadata.namespace` is optional and determines where the release is created
* `spec.releaseName` is optional and if not provided the release name will be $namespace-$name
* `spec.chart.path` is the directory containing the chart, given relative to the repository root
* `spec.values` are user customizations of default parameter values from the chart itself

The options specified in the HelmRelease `spec.values` will override the ones in `values.yaml` from the chart source.

With the `fluxcd.io/automated` annotations I instruct Flux to automate this release.
When a new tag with the prefix `dev` is pushed to Docker Hub, Flux will update the image field in the yaml file,
will commit and push the change to Git and finally will apply the change on the cluster.

![gitops-automation](https://github.com/stefanprodan/openfaas-flux/blob/master/docs/screens/flux-helm-image-update.png)

When the `podinfo-dev` HelmRelease object changes inside the cluster,
Kubernetes API will notify the Flux Helm Operator and the operator will perform a Helm release upgrade.

```
$ helm -n dev history podinfo-dev

REVISION	STATUS    	CHART        	DESCRIPTION
1       	superseded	podinfo-0.2.0	Install complete
2       	deployed  	podinfo-0.2.0	Upgrade complete
```

The Flux Helm Operator reacts to changes in the HelmRelease collection but will also detect changes in the charts source files.
If I make a change to the podinfo chart, the operator will pick that up and run an upgrade.

![gitops-chart-change](https://github.com/stefanprodan/openfaas-flux/blob/master/docs/screens/flux-helm-chart-update.png)

```
$ helm -n dev history podinfo-dev

REVISION	STATUS    	CHART        	DESCRIPTION
1       	superseded	podinfo-0.2.0	Install complete
2       	superseded	podinfo-0.2.0	Upgrade complete
3       	deployed  	podinfo-0.2.1	Upgrade complete
```

Now let's assume that I want to promote the code from the `dev` branch into a more stable environment for others to test it.
I would create a release candidate by merging the podinfo code from `dev` into the `stg` branch.
The CI would kick in and publish a new image:

```bash
$ cd hack && ./ci-mock.sh -r stefanprodan/podinfo -b stg

Successfully tagged stefanprodan/podinfo:stg-9ij63o4c
The push refers to repository [docker.io/stefanprodan/podinfo]
8f21c3669055: Pushed
```

Assuming the staging environment has some sort of automated load testing in place,
I want to have a different configuration than dev:

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: podinfo-rc
  namespace: stg
  annotations:
    fluxcd.io/automated: "true"
    filter.fluxcd.io/chart-image: glob:stg-*
spec:
  releaseName: podinfo-rc
  chart:
    git: git@github.com:fluxcd/helm-operator-get-started
    path: charts/podinfo
    ref: master
  values:
    image:
      repository: stefanprodan/podinfo
      tag: stg-9ij63o4c
    replicaCount: 2
    hpa:
      enabled: true
      maxReplicas: 10
      cpu: 50
      memory: 128Mi
```

With Flux Helm releases it's easy to manage different configurations per environment.
When adding a new option in the chart source make sure it's turned off by default so it will not affect all environments.

If I want to create a new environment, let's say for hotfixes testing, I would do the following:
* create a new namespace definition in `namespaces/hotfix.yaml`
* create a dir `releases/hotfix`
* create a HelmRelease named `podinfo-hotfix`
* set the automation filter to `glob:hotfix-*`
* make the CI tooling publish images from my hotfix branch to `stefanprodan/podinfo:hotfix-sha`

### Production promotions with sem ver

For production, instead of tagging the images with the Git commit, I will use [Semantic Versioning](https://semver.org).

Let's assume that I want to promote the code from the `stg` branch into `master` and do a production release.
After merging `stg` into `master` via a pull request, I would cut a release by tagging `master` with version `0.4.10`.

When I push the git tag, the CI will publish a new image in the `repo/app:git_tag` format:

```bash
$ cd hack && ./ci-mock.sh -r stefanprodan/podinfo -v 0.4.10

Successfully built f176482168f8
Successfully tagged stefanprodan/podinfo:0.4.10
```

If I want to automate the production deployment based on version tags, I would use `semver` filters instead of `glob`:

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: podinfo-prod
  namespace: prod
  annotations:
    fluxcd.io/automated: "true"
    filter.fluxcd.io/chart-image: semver:~0.4
spec:
  releaseName: podinfo-prod
  chart:
    git: git@github.com:fluxcd/helm-operator-get-started
    path: charts/podinfo
    ref: master
  values:
    image:
      repository: stefanprodan/podinfo
      tag: 0.4.10
    replicaCount: 3
```

Now if I release a new patch, let's say `0.4.11`, Flux will automatically deploy it.

```bash
$ cd hack && ./ci-mock.sh -r stefanprodan/podinfo -v 0.4.11

Successfully tagged stefanprodan/podinfo:0.4.11
```

![gitops-semver](https://github.com/stefanprodan/openfaas-flux/blob/master/docs/screens/flux-helm-semver.png)

### Managing Kubernetes secrets

In order to store secrets safely in a public Git repo you can use the Bitnami [Sealed Secrets controller](https://github.com/bitnami-labs/sealed-secrets)
and encrypt your Kubernetes Secrets into SealedSecrets.
The SealedSecret can be decrypted only by the controller running in your cluster.

The Sealed Secrets Helm chart is available on [Helm Hub](https://hub.helm.sh/charts/stable/sealed-secrets),
so I can use the Helm repository instead of a git repo. This is the sealed-secrets controller release:

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: sealed-secrets
  namespace: adm
spec:
  releaseName: sealed-secrets
  chart:
    repository: https://kubernetes-charts.storage.googleapis.com/
    name: sealed-secrets
    version: 1.6.1
```

Note that this release is not automated, since this is a critical component I prefer to update it manually.

Install the kubeseal CLI:

```bash
brew install kubeseal
```

At startup, the sealed-secrets controller generates a RSA key and logs the public key.
Using kubeseal you can save your public key as `pub-cert.pem`,
the public key can be safely stored in Git, and can be used to encrypt secrets without direct access to the Kubernetes cluster:

```bash
kubeseal --fetch-cert \
--controller-namespace=adm \
--controller-name=sealed-secrets \
> pub-cert.pem
```

You can generate a Kubernetes secret locally with kubectl and encrypt it with kubeseal:

```bash
kubectl -n dev create secret generic basic-auth \
--from-literal=user=admin \
--from-literal=password=admin \
--dry-run \
-o json > basic-auth.json

kubeseal --format=yaml --cert=pub-cert.pem < basic-auth.json > basic-auth.yaml
```

This generates a custom resource of type `SealedSecret` that contains the encrypted credentials:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: basic-auth
  namespace: adm
spec:
  encryptedData:
    password: AgAR5nzhX2TkJ.......
    user: AgAQDO58WniIV3gTk.......
```

Delete the `basic-auth.json` file and push the `pub-cert.pem` and `basic-auth.yaml` to Git:

```bash
rm basic-auth.json
mv basic-auth.yaml /releases/dev/

git commit -a -m "Add basic auth credentials to dev namespace" && git push
```

Flux will apply the sealed secret on your cluster and sealed-secrets controller will then decrypt it into a
Kubernetes secret.

![SealedSecrets](https://github.com/fluxcd/helm-operator-get-started/blob/master/diagrams/flux-helm-operator-sealed-secrets.png)

To prepare for disaster recovery you should backup the sealed-secrets controller private key with:

```bash
kubectl get secret -n adm sealed-secrets-key -o yaml --export > sealed-secrets-key.yaml
```

To restore from backup after a disaster, replace the newly-created secret and restart the controller:

```bash
kubectl replace secret -n adm sealed-secrets-key -f sealed-secrets-key.yaml
kubectl delete pod -n adm -l app=sealed-secrets
```

### <a name="help"></a>Getting Help

If you have any questions about Helm Operator and continuous delivery:

- Read [the Helm Operator docs](https://docs.fluxcd.io/projects/helm-operator/en/latest/).
- Read [the Flux integration with the Helm operator docs](https://docs.fluxcd.io/en/latest/references/helm-operator-integration.html).
- Invite yourself to the <a href="https://slack.cncf.io" target="_blank">CNCF community</a>
  slack and ask a question on the [#flux](https://cloud-native.slack.com/messages/flux/)
  channel.
- To be part of the conversation about Helm Operator's development, join the
  [flux-dev mailing list](https://lists.cncf.io/g/cncf-flux-dev).
- [File an issue.](https://github.com/fluxcd/flux/issues/new)

Your feedback is always welcome!



---

<strong>Marley</strong>

<a href="https://webmakaka.com"><strong>WebMakaka</strong></a>