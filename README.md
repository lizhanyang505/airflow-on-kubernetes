# Guide to Airflow on Kubernetes

Recently I spend quite some time diving into [Airflow](https://airflow.incubator.apache.org/) and [Kubernetes](https://kubernetes.io). While there are reports of people using them together, I could not find any comprehensive guide or tutorial. Also, there are many forks and abandoned scripts and repositories. So you need to do some research.

Here I write down what I've found, in the hope that it is helpful to others. Because things move quickly, I've decided to put this on Github rather than in a blog post, so it can be easily updated. Please do not hesitate to provide updates, suggestions, fixes etc.

## Decide what you want
There are some related, but different scenarios:

1. Using Airflow to schedule jobs on Kubernetes
2. Running Airflow itself on Kubernetes
3. Do both at the same time

You can actually replace Airflow with X, and you will see this pattern all the time. E.g. you can use Jenkins or Gitlab (buildservers) on a VM, but use them to deploy on Kubernetes. Or you can host them on Kubernetes, but deploy somewhere else, like on a VM. And of course you can run them in Kubernetes and deploy to Kubernetes as well.

The reason that I make this distinction is that you typically need to perform some different steps for each scenario. And you may not need both.

### [1] Scheduling jobs on Kubernetes

The simplest way to achieve this right now, is by using the kubectl commandline utility (in a BashOperator) or the [python sdk](https://github.com/kubernetes-client/python).
However, you can also deploy your Celery workers on Kubernetes. The Helm chart mentioned below does this.

#### Native Kubernetes integration
Work is in progress that should lead to native support by Airflow for scheduling jobs on Kubernetes. The wiki contains a [discussion](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=71013666) about what this will look like, though the pages haven't been updated in a while. Progress can be tracked in Jira ([AIRFLOW-1314](https://issues.apache.org/jira/browse/AIRFLOW-1314)).

Development is being done in a fork of Airflow at [bloomberg/airflow](https://github.com/bloomberg/airflow). This is still work in progress so deploying it should probably not be done in production.



#### KubernetesPodOperator (coming in 1.10)
A subset of functionality will be released earlier, according to [AIRFLOW-1517](https://issues.apache.org/jira/browse/AIRFLOW-1517).

In the next release of Airflow (1.10), a new Operator will be introduced that leads to a better, native integration of Airflow with Kubernetes. The cool thing about this Operator will be that you can define custom Docker images per task. 
Previously, if your task requires some python library or other dependency, you'll need to install that on the workers. So your workers end up hosting the combination of all dependencies of all your DAGs.
I think eventually this can replace the CeleryExecutor for many installations.

Documentation for the new Operator [can be found here](https://github.com/apache/incubator-airflow/blob/master/docs/kubernetes.rst).



### [2] Running Airflow on Kubernetes
Airflow is implemented in a modular way. The main components are the scheduler, the webserver, and workers.
You also probably want a database.
If you want to distribute workers, you may want to use the CeleryExecutor. In that case, you'll probably want Flower (a UI for Celery) and you need a queue, like RabbitMQ or Redis.



## Docker images
First thing you need is a Docker image that packages Airflow. Most people seem to use [puckel/docker-airflow](https://github.com/puckel/docker-airflow). This is a robust Docker image that is up to date and also has an automated build set up, pushing images to Dockerhub. This means that you do not need to build your own image when you are first starting out. The image has an [entrypoint script](https://github.com/puckel/docker-airflow/blob/master/script/entrypoint.sh) that allows the container to fulfill the role of scheduler, webserver, flower, or worker.

While `puckel/docker-airflow` is widely used, it isn't updated very often anymore, so it is lagging behind Airflow releases a bit. Many people are forking this repo and updating it themselves. At this point the Airflow community is lacking a canonical Docker image.

* [puckel/docker-airflow](https://hub.docker.com/r/puckel/docker-airflow)

Another option may be the image by astronomer.io, though I cannot find the Dockerfile source, so at this point I'm hesitant to run it:

* [astronomerinc/ap-airflow](https://hub.docker.com/r/astronomerinc/ap-airflow)


## Helm
Next you need to create some Kubernetes manifests, or a Helm chart, to deploy the Docker image on Kubernetes. There is some work in this area, but it is not completely finished yet. 

There are several helm charts to install Airflow on kubernetes:

* [Official Helm chart](https://github.com/helm/charts/tree/master/stable/airflow)
* [astronomer.io chart](https://github.com/astronomer/helm.astronomer.io/tree/master/charts/airflow)
* [GoDataDriven chart](https://github.com/godatadriven/airflow-helm)

It's a bit unfortunate that the community has not yet arrived at a canonical Chart, so you'll have to try your luck. The 'official' chart probably covers most deployment options, including Celery and non-Kubernetes options, while the others may be more opinionated (and focused).

To install the official chart: 

```bash
helm install --namespace "airflow" --name "airflow" stable/airflow
```  

## Kubernetes Operator

There is work by Google on a [Kubernetes Operator for Airflow](https://github.com/GoogleCloudPlatform/airflow-operator). This name is quite confusing, as *operator* here refers to a [controller for an application on Kubernetes](https://coreos.com/blog/introducing-operators.html), not an Airflow Operator that describes a task. So you would use this operator instead of using the Helm chart to deploy Kubernetes itself.




## Deploying your DAGs
There are several ways to deploy your DAG files when running Airflow on Kubernetes.

1. git-sync
2. Persistent Volume
3. Embedding in Docker container

### Mounting Persistent Volume
You can store your DAG files on an external volume, and mount this volume into the relevant Pods
(scheduler, web, worker). In this scenario, your CI/CD pipeline should update the DAG files in the 
PV.
Since all Pods should have the same collection of DAG files, it is recommended to create just one PV
that is shared. This ensures that the Pods are always in sync about the DagBag. 

To share a PV with multiple Pods, the PV needs to have accessMode 'ReadOnlyMany' or 'ReadWriteMany'.
If you are on AWS, you can use [Elastic File System (EFS)](https://aws.amazon.com/efs/). 
If you are on Azure, you can use [Azure File Storage (AFS)](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv).

If you mount the PV as ReadOnlyMany, you get reasonable security because the DAG files cannot be manipulated from within the Kubernetes cluster. You can then update the DAG files via another channel, for example, your build server.

## RBAC
If your cluster has [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) turned on, and you want to launch Pods from Airflow, you will need to bind the appropriate roles to the serviceAccount of the Pod that wants to schedule other Pods. Typically, this means that the Workers (when using CeleryExecutor) or the Scheduler (using LocalExecutor or the new KubernetesPodExecutor) need extra permissions. You'll need to grant the 'watch/create' verbs on Pods.

## Relevant links
* [Awesome Apache Airflow](https://github.com/jghoman/awesome-apache-airflow)
* [Airflow on AKS](https://blog.godatadriven.com/airflow-on-aks) (end-to-end guide)
* [Airflow scheduler healthcheck](https://roland.de-boo.nl/airflow-healthcheck/)

k8s install uri
https://hub.kubeapps.com/charts/stable/airflow
