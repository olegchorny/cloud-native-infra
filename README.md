# Cloud Native Infrastructure - from zero to hero

This sample shows:
- how to deploy the Kubernetes, Istio and Knative stack (KIK)
- how to build and deploy application on top of KIK stack
- how to scale application from 0 up to virtually unlimited number of instances
- how to orchestrate blue-green deployments

## Kubernetes
We are going to leverage GKE cluster as a tribute to https://github.com/GIFEE/GIFEE

Create cluster:
![Kubernetes](./img/1.png)

2 nodes, 2vCPUs, 7.5GB memory, latest Kubernetes version and additional thing in advanced edit..
![Kubernetes](./img/2.png)

Enable auto-scaling, preemtible nodes (they are almost 4 times cheaper!) and auto-upgrade
![Kubernetes](./img/3.png)

Save settings, push the Create button and when cluster is ready push the Connect button and Run In Cloud Shell 
![Kubernetes](./img/4.png)

Check the state of you cluster using `kubectl get nodes` and `kubectl get pods --all-namespaces` commands
![Kubernetes](./img/5.png)

If Nodes are Ready and Pods are Running, we are good to go to the next step.

## Istio

Grant cluster-admin permissions to the current user:
```
kubectl create clusterrolebinding cluster-admin-binding \
--clusterrole=cluster-admin \
--user=$(gcloud config get-value core/account)
```

And install Istio. `kubectl apply --filename https://raw.githubusercontent.com/knative/serving/v0.2.2/third_party/istio-1.0.2/istio.yaml`

![Istio](./img/6.png)

<!-- If you see "unable to recognize ..." errors, just apply the previous command again -->
<!-- ![Istio](./img/7.png) -->

Wait until all Istio Pods will be in the Running or Completed state
![Istio](./img/8.png)

And we are good to go to the Knative stage

### Knative
Install Knative: `kubectl apply --filename https://github.com/knative/serving/releases/download/v0.2.2/release.yaml`, wait until all the Pods are Running and label the default namespace to inject istio-proxy automatically: `kubectl label namespace default istio-injection=enabled`

![Knative](./img/9.png)

Create docker-secret.yaml to store your Docker Hub credentials
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-user-pass
  annotations:
    build.knative.dev/docker-0: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
data:
  # Use 'echo -n "username" | base64' to generate this string
  username: b2Nob3JueQ==
  # Use 'echo -n "password" | base64' to generate this string
  password: V0B0Y2hmdTE=
```
Create service-account.yaml to link the build process to the secret.
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
  - name: basic-user-pass
```

Install Kaniko template to build containers: `kubectl apply --filename https://raw.githubusercontent.com/knative/build-templates/master/kaniko/kaniko.yaml`

Apply Secret manifest: `kubectl apply --filename docker-secret.yaml`
Apply Service Account manifest: `kubectl apply --filename service-account.yaml`

![Knative](./img/10.png)

### Build and Run the Code

This sample uses `github.com/mchmarny/simple-app` as a basic Go application, but you could replace this GitHub repo with your own. The only requirements are that the repo must contain a Dockerfile with the instructions for how to build a container for the application.

Create the `service.yaml` to define how to build and deploy the code:

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: app-from-source
  namespace: default
spec:
  runLatest:
    configuration:
      build:
        apiVersion: build.knative.dev/v1alpha1
        kind: Build
        spec:
          serviceAccountName: build-bot
          source:
            git:
              url: https://github.com/mchmarny/simple-app.git
              revision: master
          template:
            name: kaniko
            arguments:
            - name: IMAGE
              value: docker.io/ochorny/app-from-source:latest
      revisionTemplate:
        spec:
          containerConcurrency: 1
          container:
            image: docker.io/ochorny/app-from-source:latest
            imagePullPolicy: Always
            env:
            - name: SIMPLE_MSG
              value: "Hello from the sample app!"
```

Apply the manifest: `kubectl apply -f service.yaml` and watch the result `kubectl get pods --watch`
Once you see the deployment pod switch to the running state, press Ctrl+C to escape the watch. Your container is now built and deployed!

![Build and Run](./img/11.png)

Jnative applications exposed via knative-ingressgateway, let's figure out the external IP: `kubectl get svc knative-ingressgateway --namespace istio-system` and expected URL for our service: `kubectl get ksvc app-from-source  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain`

Then, let's make the request to see the result: `curl -H "Host: app-from-source.default.example.com" http://{IP_ADDRESS}`


