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
$ ./ci-mock.sh -r webmakaka/podinfo -b dev
```

<br/>

```
$ helm -n dev history podinfo-dev
REVISION	UPDATED                 	STATUS  	CHART        	APP VERSION	DESCRIPTION     
1       	Thu Apr 16 22:07:28 2020	deployed	podinfo-3.1.5	3.1.5      	Install complete
```

<br/>

```bash
$ ./ci-mock.sh -r webmakaka/podinfo -b stg
```

<br/>

```bash
$ ./ci-mock.sh -r webmakaka/podinfo -v 0.4.10
```

<br/>

```bash
$ ./ci-mock.sh -r webmakaka/podinfo -v 0.4.11
```

<br/>

```
$ kubectl get pods -n prod
NAME                           READY   STATUS    RESTARTS   AGE
podinfo-prod-6d7bd6bc8-2dgzb   1/1     Running   0          4m22s
podinfo-prod-6d7bd6bc8-2kzv2   1/1     Running   0          4m22s
podinfo-prod-6d7bd6bc8-zslq5   1/1     Running   0          4m10s
```

<br/>

```
$ kubectl -n prod describe pod podinfo-prod-6d7bd6bc8-2dgzb | grep Image
    Image:         webmakaka/podinfo:0.4.11

```

---

<a href="https://marley.org"><strong>Marley</strong></a>
