# Deploying Containers to a Kubernetes Cluster

In this lab, you will deploy a containerized application to a Kubernetes cluster.

> This lab assumes you have a project set up and a Kubernetes cluster up and running.
> If you don't have a cluster yet, please [complete this lab first](../lab-02/README.md).

## Step 1 &mdash; Use a Kubernetes Cluster

Configure the use of a [StackReference][stack-refs] to the Kubernetes cluster
stack to extract and use the kubeconfig. This can be found in the [last
section](../lab-04/README.md#next-steps) of the previous lab.

From your project's root directory, install the Pulumi and Kubernetes package:

```
pip3 install pulumi pulumi-kubernetes
```

Next, add these imports to your `__main__.py` file:

```python
from pulumi import export, StackReference, Output, ResourceOptions
from pulumi_kubernetes import Provider
import pulumi
```

Create a StackReference to the Kubernetes cluster stack using the config
`clusterStackRef` stack setting.

```python
config  = pulumi.Config()
stackRef = config.require("clusterStackRef");
infra = StackReference(f"{stackRef}")
```

Next, we need to declare a new Kubernetes provider based on the kubeconfig created in 
the cluster stack. To do this, add this to your `__main__.py` file

```python
k8s_provider = Provider("k8s-provider", kubeconfig=infra.get_output("kubeconfig"))
```

> :white_check_mark: After this change, your `__main__.py` should [look like this](./code/step1.py).

[stack-refs]: https://www.pulumi.com/docs/intro/concepts/organizing-stacks-projects/#inter-stack-dependencies

## Step 2 &mdash; Import the Kubernetes Resources And Create a Namespace

First, add these imports to your `__main__.py` file:

```python
from pulumi_kubernetes.apps.v1 import Deployment
from pulumi_kubernetes.core.v1 import Service, Namespace
```

Next, declare a [namespace object](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/). This will scope your objects to a name of your choosing, so that in this workshop you won't accidentally interfere with other participants.

Append this to your `__main__.py` file, replacing `joe-duffy` with your own name and referencing the Provider created in step1:

```python
# Create a Namespace
ns = Namespace("app-ns",
    metadata={
        "name": "joe-duffy",
    },
    opts=ResourceOptions(provider=k8s_provider)
)
```

> :white_check_mark: After this change, your `__main__.py` should [look like this](./code/step2.py).

## Step 3 &mdash; Declare Your Application's Deployment Object

You'll now declare a [deployment object](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), which deploys a specific set of containers to the cluster and scales them. In this case, you'll deploy the pre-built `gcr.io/google-samples/kubernetes-bootcamp:v1` container image with only a single replica.

Append this to your `__main__.py` file:

```python
# ...
app_labels = {
    "app": "iac-workshop"
}
app_deployment = Deployment("app-dep",
    metadata={
        "namespace": ns.metadata["name"]
    },
    spec={
        "selector": {
            "match_labels": app_labels,
        },
        "replicas": 1,
        "template": {
            "metadata": {
                "labels": app_labels,
            },
            "spec": {
                "containers": [{
                    "name": "iac-workshop",
                    "image": "gcr.io/google-samples/kubernetes-bootcamp:v1",
                }],
            },
        },
    },
    opts=ResourceOptions(provider=k8s_provider)
)
```

> :white_check_mark: After this change, your `__main__.py` should [look like this](./code/step3.py).

## Step 4 &mdash; Declare Your Application's Service Object

Next, you'll declare a [service object](https://kubernetes.io/docs/concepts/services-networking/service/), which enables networking and load balancing across your deployment replicas.

Append this to your `__main__.py` file:

```python
# ...
service = Service("app-service",
    metadata={
        "namespace": ns.metadata["name"],
        "labels": app_labels
    },
    spec={
        "ports": [{
            "port": 80,
            "target_port": 8080,
        }],
        "selector": app_labels,
        "type": "LoadBalancer",
    },
    opts=ResourceOptions(provider=k8s_provider)
)
```

Afterwards, add these lines to export the resulting, dynamically assigned endpoint for the resulting load balancer:

```python
# ...
export('url', Output.all(service.status['load_balancer']['ingress'][0]['hostname'], service.spec['ports'][0]['port'])\
    .apply(lambda args: f"http://{args[0]}:{round(args[1])}"))
```

> :white_check_mark: After these changes, your `__main__.py` should [look like this](./code/step4.py).

## Step 5 &mdash; Deploy Everything

First, add the `StackReference` to the cluster stack, which is used to get the kubeconfig
from its stack output.

> This can be found in the [last section](../lab-04/README.md#next-steps) of the previous lab.

```bash
pulumi config set iac-workshop-apps:clusterStackRef joeduffy/iac-workshop-cluster/dev
```

Deploy Everything:

```bash
pulumi up
```

This will show you a preview and, after selecting `yes`, the application will be deployed:

```
Updating (dev):

     Type                             Name                                    Status
 +   pulumi:pulumi:Stack              iac-workshop-apps                       created
 +   ├─ pulumi:providers:kubernetes   k8s-provider                            created
 +   ├─ kubernetes:core:Namespace     app-ns                                  created
 +   ├─ kubernetes:core:Service       app-service                             created
 +   └─ kubernetes:apps:Deployment    app-dep                                 created

Outputs:
    url: "http://ae7c37b7c510511eab4540a6f2211784-521581596.us-west-2.elb.amazonaws.com:80"

Resources:
    + 5 created

Duration: 32s

Permalink: https://app.pulumi.com/joeduffy/iac-workshop-apps/dev/updates/1
```

List the pods in your namespace, again replacing `joe-duffy` with the namespace you chose earlier:

```bash
kubectl get pods --namespace joe-duffy
```

And you should see a single replica:

```
NAME                                READY   STATUS    RESTARTS   AGE
app-dep-8r1febnu-66bffbf565-vx9kv   1/1     Running   0          0m15s
```

Curl the resulting endpoint to view the application:

```bash
curl $(pulumi stack output url)
```

You should see something like the following:

```
Hello Kubernetes bootcamp! | Running on: app-dep-8r1febnu-66bffbf565-vx9kv | v=1
```

> Kubernetes does not wait until the AWS load balancer is fully initialized, so it may take a few minutes before it becomes available.

## Step 6 &mdash; Update Your Application Configuration

Next, you'll make two changes to the application:

* Scale out to 3 replicas, instead of just 1.
* Update the version of your application by changing its container image tag

Note that the application says `Demo application version v0.10.0-blue` in the banner. After deploying, it will change to version `v0.10.0-green`.

First update your deployment's configuration's replica count:

```
# ...
        replicas=3,
# ...
```

And then update its image to:

```
# ...
                    image="jocatalin/kubernetes-bootcamp:v2",
# ...
```

> :white_check_mark: After this change, your `__main__.py` should [look like this](./code/step6.py).

Deploy your updates:

```bash
pulumi up
```

This will show you that the deployment has changed:

```
Previewing update (dev):

     Type                           Name              Plan       Info
     pulumi:pulumi:Stack            iac-workshop-apps-dev
 ~   └─ kubernetes:apps:Deployment  app-dep           update     [diff: ~spec]

Resources:
    ~ 1 to update
    3 unchanged

Do you want to perform this update?
  yes
> no
  details
```

Selecting `details` will reveal the two changed made above:

```
  pulumi:pulumi:Stack: (same)
    [urn=urn:pulumi:dev::iac-workshop-apps::pulumi:pulumi:Stack::iac-workshop-apps-dev]
    ~ kubernetes:apps/v1:Deployment: (update)
        [id=joe-duffy/app-dep-8r1febnu]
        [urn=urn:pulumi:dev::iac-workshop-apps::kubernetes:apps/v1:Deployment::app-dep]
        [provider=urn:pulumi:dev::iac-workshop-apps::pulumi:providers:kubernetes::default_1_2_3::c2145624-bf5a-4e9e-97c6-199096da4c67]
      ~ spec: {
          ~ replicas: 1 => 3
          ~ template: {
              ~ spec: {
                  ~ containers: [
                      ~ [0]: {
                              ~ image: "gcr.io/google-samples/kubernetes-bootcamp:v1" => "jocatalin/kubernetes-bootcamp:v2"
                            }
                    ]
                }
            }
        }

Do you want to perform this update?
  yes
> no
  details
```

And selecting `yes` will apply them:

```
Updating (dev):

     Type                           Name              Status      Info
     pulumi:pulumi:Stack            iac-workshop-apps-dev
 ~   └─ kubernetes:apps:Deployment  app-dep           updated     [diff: ~spec]

Outputs:
    url: "http://ae33950ecf82111e9962d024411cd1af-422878052.eu-central-1.elb.amazonaws.com:80"

Resources:
    ~ 1 updated
    3 unchanged

Duration: 16s

Permalink: https://app.pulumi.com/joeduffy/iac-workshop-apps/dev/updates/2
```

Query the pods again using your chosen namespace from earlier:

```bash
kubectl get pods --namespace joe-duffy
```

Check that there are now three:

```
NAME                               READY   STATUS    RESTARTS   AGE
app-dep-8r1febnu-6cd57d964-c76rx   1/1     Running   0          8m45s
app-dep-8r1febnu-6cd57d964-rdpn6   1/1     Running   0          8m35s
app-dep-8r1febnu-6cd57d964-tj6m4   1/1     Running   0          8m56s
```

Finally, curl the endpoint again:

```bash
curl $(pulumi stack output url)
```

And verify that the output now ends in `v=2`, instead of `v=1` (the result of the new container image):

```
Hello Kubernetes bootcamp! | Running on: app-dep-8r1febnu-6cd57d964-c76rx | v=2
```

If you'd like, do it a few more times, and observe that traffic will be load balanced across the three pods:

```bash
for i in {0..10}; do curl $(pulumi stack output url); done
```

## Step 7 &mdash; Destroy Everything

Finally, destroy the resources and the stack itself:

```
pulumi destroy
pulumi stack rm
```

## Next Steps

Congratulations! 