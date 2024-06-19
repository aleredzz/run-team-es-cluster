# Solutions 

Bellow I am answering the questions sent as part of the challenge.

### Deliberate on the deployment method—whether it be through plain YAML, Helm, Kustomize, or any other suitable mechanism.

For this solution, I opt for the Operator pattern offered by Elastic to install Elastic Search clusters in Kubernetes. It is also the recommended way to by Elastic to lift and run the platform, known as `ECK` (Elastic Cloud on Kubernetes).

I thought it would be important for the Run Team to know what technologies they running on, hence why the paragraph above is also found the in README.md file at the root of this project, without going into too much details.

Plain YAMLs are hard to manage when it comes to customizations. In organizations where people are not familiar with Kubernetes resource files, specially a large amount of them, changing a specific annotation or an amount of CPU for a deployment, can be difficult. Similar case happens to Kustomize since it offers a customization abstraction on top of these YAML files.

If we think of an unify way to see these configurations or customizations, you could think using a Helmchart. However, if you go to the official Elastic Search Helmchart, you will find they point you at the ECK (Operator) option.

Another point in favor of the Operator is that it handles many tasks automatically, for instance, upgrades. So leveraging this burdens off on to the Operator, makes it easier to maintaining the platform.

Lastly, I would stay away from anything that is not recommended by the vendor. Any other path might introduce more implementation time, complexity and lack of support or available information.

### Outline how the solution will handle potential failures to maintain continuous service availability.

In my experience, High Availability was made with at least 3 nodes. Every time I used to spin up K8s control plane nodes at Intel, we always set for no less than three. The argument behind it was not only cost-effective but also that if you set sail with 2 nodes, you can only manage to use less than 50% of resources available on each to be able to sustain a node failure. Having more resources meant that rollin updates and moving workloads around, is faster and has less processing overhead when you have more than 1 node.

Node anti-affinity is part of this solution too. The whole premise of High Availability needs to have congruent and sound implementation. The idea behind is that "I want 3 ECK nodes, _in 3 different cluster nodes_". This last part I can achieve it with node anti-affinity.

Now, in the event that something does goes down, persistent storage can help. This is why I included a storage template in the solution since EBS based storage was available in the environment and also PV was set to `retain`.

```
kubectl get storageclass
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  5d11h

kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                 STORAGECLASS   REASON   AGE
pvc-0c3b3a38-8314-4d74-94c7-9720f3adf931   10Gi       RWO            Retain           Bound    eks/elasticsearch-data-run-team-es-all-roles-node-0   gp2                     3h5m
pvc-2975cd2c-89b2-4f49-ac05-a2bf4b63bfb9   10Gi       RWO            Retain           Bound    eks/elasticsearch-data-run-team-es-all-roles-node-2   gp2                     3h5m
pvc-a4cfd067-44f7-49e1-87e9-1b9ff908f418   10Gi       RWO            Retain           Bound    eks/elasticsearch-data-run-team-es-all-roles-node-1   gp2                     3h5m

```

### Elaborate on authentication, authorization, encryption, and any other security practices/concerns used in this solution

The Elasticsearch operator implementation of this solution has default user `elastic` but a randomly generated password. This helps for initial configuration of the platform, as is not an easily gessable password.
In order to access the password, the user must posses access the the `eks` namespace where the name reside. I also included some good practices in the `Readme.md` file when it comes to this default account and least-privilege approaches. 

Another aspect of this solution is that the all the ES services use SSL. This makes the channels encrypted and secure. I pointed out also, in the case of Kibana, that the SSL termination was not being used. And removed any unpractical and unneeded risks. 

By using EBS for persisting storage, we are one a few clicks away of having encryption at rest of the elastic search data. Which is also is a good practice, specially if there are concerns with the data when its deemed Personal Information.  

So at this point we have covered grater areas of security - Authentication, Authorization, Data at rest and in transit.


### Plan for ongoing operational tasks such as system upgrades, cluster scaling, disaster recovery procedures, etc. Explain your approach to handling these day-to-day operational challenges

One of the benefits of the Operator is that it takes care of many complex tasks. In the case of the upgrades. The ELK documentation website offers guides on [upgrading ECK](https://www.elastic.co/guide/en/cloud-on-k8s/2.8/k8s-upgrading-eck.html#k8s-upgrading-eck). So the strategy would go around creating team's upgrade guides off the documentation for each case. Upgrades procedures for specific versions for instance "Upgrading ELK to 2.8" or "Upgrading ELK to 2.9" so the team has a handy, known, tested, revised procedure to go on.

In the case of disaster recovery, guides for specific events and occasional drill runs, can prepare the team on time for eventual emergency cases. Scenarios like, cluster down, lost backups, can create actionable to improve our disaster recovery stand point, for example secondary disc backups in a secondary region.

Part of the operational tasks, I believe involves active monitoring and alerting. Ideally I would centralize the monitoring of the ES instances we have installed. Creating alert groups on specific metrics like response times, CPU and Memory consumption and logging would be useful too. Here we can have a more proactive attitude towards events that could cause disruption on the service.

Documenting each event addressed by team with an standard approach such as "Symptoms, checks, solutions" case can also help towards managing these events faster. It can also help towards a self-healing strategy. 

### Draft a strategy for system observability, specifying which metrics are critical for monitoring and analyzing the system's health and performance.

I would like to take this incremental phases. From a platform perspective, I would tap into observability with the services available known endpoints like basic `/health`  and response times. This step is easy to implement with a service meshes. `Linkerd` or `Istio` could be a very cost-effective first step on to understanding what services are being used the most.

I would first develop a plan where there is an service mesh platform like linkerd, or Istio installed successfully in the cluster. Hopefully we also have different environments rather than the "production" cluster given for this example. Here, the process can tested to a good extent.

Linkerd and Istio are both supported by ECK, so any option is feasable:
- https://www.elastic.co/guide/en/cloud-on-k8s/2.7/k8s-service-mesh-istio.html
- https://www.elastic.co/guide/en/cloud-on-k8s/2.7/k8s-service-mesh-linkerd.html

Now, I would follow and document these integration guide procedures so that the process is streamlined as much as possible. (first step towards automation)

My next stage is to drill deeper. I would like to have an overview on the pods internal processes and perhaps understand more about system calls. Many tools offer this type of insight such as Datadog or New Relic. Other option that can be cheaper and Open Source and can also offer this information Prometheus and Grafana. These 2 are kind of the defacto tooling for this purpose.

With both of these steps, we cover the networking part, understanding the moving packets, latencies, ports, usage, and also the application call stack and pod's resource management.

_Note: I used to use New Relic at Intel and saw Datadog's product at KubeCon Amsterdam 2023_

**Additional Future Improvements. We value your way of thinking, so try to avoid copying and pasting from other solutions or other resources/LLM models. If you want to use them, please put a reference.**

The first task I would propose is to _automate_. We have at this point, good documentation and knowledge of the process. Since I have worked with AWX and Ansible more, my proposal would go towards creating recipes that will deploy ECK clusters, and add observability on top. Whenever the request comes in for another environment, we will have automated the task. The tasks for my team now can revolve around improving the recipes and upgrade the platforms we install in an automatic way too.
Moreover, adding encryption at rest for the ElasticSearch data store is good step forward, as it encrypts the data in filesystem and EBS, as an AWS service, also offers backups. However this points might got away from the Kubenetes space. 
Lastly, the certificate strategy: could be in better shape. meaning that we can include certificate rotation and include documentation around that for the end user could be very handy.
