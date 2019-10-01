# Modifying Actions and Deploying to a local cluster

This article describes how to perform the following on a Linux box:
* Deploy a local cluster through minikube
* (Optional) deploy the default Actions to the cluster
* Make local changes to the Actions codebase and deploy them to the cluster

First install minikube and kubectl.  This page has instructions for both: https://kubernetes.io/docs/tasks/tools/install-minikube/.

Note: depending on your environment, you may need to use `sudo` for all docker commands below.

## Set Up a Minikube Cluster
1.  Start your minikube cluster.  
```
minikube start --vm-driver=kvm2 --cpus=4 --memory=4096 --kubernetes-version=1.14.6 --extra-config=apiserver.authorization-mode=RBAC
```

*Note: if you stop the cluster, you must start it again with the args above.  A common mistake is to start it again with simply:*
```
[wrong] minikube start
```

2.
Create clusterrole.yaml somewhere locally with these contents.  This is to enable RBAC (in some cases enabling RBAC via #1 may not work):
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
```

3.
Now run the command below to apply it
```
kubectl create -f clusterrole.yaml
```
You may get `AlreadyExists`, that is ok.

4.
Install helm if you haven't already using
```
sudo snap install helm --classic
```

5.
Create the tiller service account
```
kubectl apply -f https://raw.githubusercontent.com/Azure/helm-charts/master/docs/prerequisities/helm-rbac-config.yaml
```

6. 
Run the following to install tiller into the cluster
```
helm init --service-account tiller --history-max 200
```

7.
To confirm the step above worked, you should see a pod with a name starting with "tiller-deploy".
```
kubectl get pods -n kube-system
```

If you do *not* see one, try running these two commands:
```
kubectl create serviceaccount -n kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

Then repeat step 6.

8.
If you still don't see a tiller pod, you may have to debug your environment.  Try running the following, and look for errors under "Events" at the bottom of the output:
```
kubectl describe deployment tiller-deploy --namespace kube-system
```

9.
If you haven't already cloned the code, clone https://github.com/actionscore/actions.

## Deploy Actions (unmodified) to the Cluster

*This section describes how to deploy Actions, without changes, to your cluster.  It may be useful to go through to see how things should look, but it is optional.  To skip to the part where you make changes to Actions and deploy them, skip to the section below named "Modify Actions and Deploy to the Cluster".*

Now let's deploy Actions.  From the location you cloned to, go to 
```
[location you cloned to]/actionscore/actions/charts/actions-operator 
```

and run

```
helm install . --name actions --namespace actions-system
```

This will install Actions to your cluster from the default repository.

1.
To make sure it deployed correctly, use the dashboard:
```
minikube dashboard
```

or run the following:
```
kubectl get pods -n actions-system
```

You should see three pods and they should have a status of "Running".  Note the namespace.

2.  
Now we're going to delete Actions, make some code edits to Actions, and redeploy.

Run the following to delete Actions:
```
helm del --purge actions
```

Use the following to confirm it's gone: 
```
kubectl get pods -n actions-system
```

## Modify Actions and Deploy to the Cluster

1.
Open the following file and add a print statement of your choice in main()
```
[location you cloned to]/actionscore/actions/cmd/operator/main.go
```

Save the file.

2. 
Go up two levels to 
```
[location you cloned to]/actionscore/actions
```

and build:
```
make build GOOS=linux GOARCH=amd64
```

You should see the binaries in ./dist/linux_amd64/release.

3.
We're going to now build a container with our changes and push it to Docker Hub.

Log into Docker Hub:
```
docker login
```

4.
Open the Dockerfile in the current directory and replace the last line, the `ADD`, with:

```
ADD ./dist/linux_amd64/release/. /
```

5.
Build and tag your image
```
docker build -t [Docker Hub repo]:[tag] .
```

For example:
```
docker build -t user123/actions_dev:latest .
```

6.
Push the image to Docker Hub
```
docker push user123/actions_dev:latest
```

7.
If you deployed Actions before, delete it now using 
```
helm del --purge actions
```

8.
Now we'll redeploy actions with your changes.

Go to 
```
[place you cloned to]/actionscore/actions/charts/actions-operator
```

and run the following to deploy:
```
helm install --name=actions --namespace=actions-system --set-string global.registry=docker.io/[your Docker Hub id] global.tag=[the tag of the image you just built] .
```


21.
Once Actions is deployed, you should see the print statement you added by running:
```
kubectl logs [your actions-operator pod name]
```

For example:
```
kubectl logs actions-operator-123-456
```
 