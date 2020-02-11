# kube-istio-cluster

In order to prepare a fully configured cluster for our experiments that needs
to run [Kubernetes](https://kubernetes.io Kubernete's Home Page) and
[Istio](https://istio.io/ Istio's HomePage), we grouped all
[Ansible](https://www.ansible.com/ Ansible's Home Page) scripts used to
configure the available nodes.

The nodes used in the experiment are provisioned by the
[Emulab](https://www.emulab.net Emulab's Home Page) project, which provides
configurable types of nodes and different network topologies for researchers to
experiement with.

In this repository are the main scripts used for the configuration of the
nodes using _Ansible_ that runs commands in all nodes declared within a group in
the [hosts][h] file.

For an example of an experiment we've used 3 nodes: 1 master and 2 workers. For
that we created in the [hosts][h] file two groups, the master and workers group.
If you'd like to have more worker nodes, for example, you only have to add more
hosts to the group. We also define environment variables for the native group
_all_, which represents every node. The example file is as follows:

```
[masters]
master ansible_host=hos1.example.net ansible_user=hostuser

[workers]
worker1 ansible_host=hos2.example.net ansible_user=hostuser
worker2 ansible_host=hos3.example.net ansible_user=hostuser

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

Here we defined the groups and added named hosts to them. We also defined the
environment variable _ansible_python_interpreter_, which tells _Ansible_ which
_Python_ interpreter to use.

[h]: ./hosts

## Usage

Let's configure our cluster with _Kubernetes_ and _Istio_ then. Inthe case of
_Emulab_'s hosts, I recomend logging in via `ssh` so your identity is added to
the machine's allowed hosts, if you do not do this so, the scripts may not be
able to run in the machines, only in the first one wich can capture your input
to add the identity.

`ssh -p 22 myhost@host1.example.net`

And so on! Now you can clone this repo with
`git clone https://github.com/r3-musketeers/kube-istio-cluster.git` and then
change to the cloned directory with `cd kube-istio-cluster`. Now we are ready to
proceed to the next steps.

_PS:_ How to install _Ansible_?

```shell
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

### 01-setup.yaml

`ansible-playbook -i hosts 01-setup.yaml`

In this script we are setting up authorized keys for the current _Ansible_ user
and disabling swap in all nodes because _Kubernetes_ demands so. We are also
deleting the entry in the _fstab_ file so it will not turn swap back on when the
machine is rebooted.

### 02-dependencies.yaml

`ansible-playbook -i hosts 02-dependencies.yaml`

This script install in all nodes the dependencies necessary for _Kubernetes_
like: _docker_, _kubelet_ and _kubeadm_, and install the _kubectl_ in the master
node only, because _kubectl_ is the cluster management tool which can only run
in a master node.

### 03-init-master.yaml

`ansible-playbook -i hosts 03-init-master.yaml`

Like the name says, this script will init the master node running the necessary
steps to this node assume the _ROLE_ master. The master node is the first one to
be initialized because in it you will install the _SDN_ (Software Defined Network),
in this case [Flannel](https://github.com/coreos/flannel Flannel's GitHub Page),
and generate the _join cluster_ command that the workers node will use with a
generated token to join the cluster.

_PS:_ For some reason the command to apply the _Flannel_ definition to the
cluster state doesn't work when in the ansible script. For this to work I had to
run the command manually in the master node:

```
ssh -p 22 hostuser@host1.example.net
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml >> pod_network_setup.txt
```

### 04-join-workers.yaml

`ansible-playbook -i hosts 04-join-workers.yaml`

Basically this scripts gets the join command in the master node using _kubectl_
and executes it in the workers nodes.

At this point, it is possible to access the master node and check our cluster
status. With the command `kubectl get nodes` we should get the following output:

```stdout
NAME                                      STATUS   ROLES    AGE     VERSION
node0.kube-istio.scalablesmr.emulab.net   Ready    master   13m     v1.17.2
node1.kube-istio.scalablesmr.emulab.net   Ready    <none>   7m54s   v1.17.2
node2.kube-istio.scalablesmr.emulab.net   Ready    <none>   7m55s   v1.17.2
```

### 05-download-istio.yaml

`ansible-playbook -i hosts 05-download-istio.yaml`

This script downloads the _Istio_ download script, runs and removes it. We also
append the istio binaries path to the environment variable PATH so we can run
`istioctl` without informing the executable location everytime.

### 06-configure-instio.yaml

`ansible-playbook -i hosts 06-configure-istio.yaml`

Now we are going to install the _istio demo profile_ which is designed to
instantiate pods and services to showcase high levels of tracing and logging.
Such profile is not suitable for performance evaluation.

Right now we can access the master node and check the services and pods
instantiated in our cluster by _Istio_ demo profile, all of these resources are
grouped in a _namespace_ 'istio-system':

`kubectl get services -n istio-system`

```stdout
NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                        AGE
grafana                  ClusterIP      172.21.211.123   <none>          3000/TCP                                       2m
istio-citadel            ClusterIP      172.21.177.222   <none>          8060/TCP,15014/TCP                             2m
istio-egressgateway      ClusterIP      172.21.113.24    <none>          80/TCP,443/TCP,15443/TCP                       2m
istio-galley             ClusterIP      172.21.132.247   <none>          443/TCP,15014/TCP,9901/TCP                     2m
istio-ingressgateway     LoadBalancer   172.21.144.254   <pending>       15020:31831/TCP,80:31380/TCP,443:31390/TCP,... 2m
istio-pilot              ClusterIP      172.21.105.205   <none>          15010/TCP,15011/TCP,8080/TCP,15014/TCP         2m
istio-policy             ClusterIP      172.21.14.236    <none>          9091/TCP,15004/TCP,15014/TCP                   2m
istio-sidecar-injector   ClusterIP      172.21.155.47    <none>          443/TCP,15014/TCP                              2m
istio-telemetry          ClusterIP      172.21.196.79    <none>          9091/TCP,15004/TCP,15014/TCP,42422/TCP         2m
jaeger-agent             ClusterIP      None             <none>          5775/UDP,6831/UDP,6832/UDP                     2m
jaeger-collector         ClusterIP      172.21.135.51    <none>          14267/TCP,14268/TCP                            2m
jaeger-query             ClusterIP      172.21.26.187    <none>          16686/TCP                                      2m
kiali                    ClusterIP      172.21.155.201   <none>          20001/TCP                                      2m
prometheus               ClusterIP      172.21.63.159    <none>          9090/TCP                                       2m
tracing                  ClusterIP      172.21.2.245     <none>          80/TCP                                         2m
zipkin                   ClusterIP      172.21.182.245   <none>          9411/TCP                                       2m
```

`kubectl get podes -n istio-system`

```stdout
NAME                                      READY   STATUS    RESTARTS   AGE
grafana-6b65874977-n75rs                  1/1     Running   0          9m3s
istio-citadel-f78ff689-x4srt              1/1     Running   0          9m22s
istio-egressgateway-7b6b69ddcd-bhtzb      1/1     Running   0          9m18s
istio-galley-69674cb559-l5lzh             1/1     Running   0          9m17s
istio-ingressgateway-649f9646d4-pfs65     1/1     Running   0          9m9s
istio-pilot-7989874664-mstgd              1/1     Running   0          9m16s
istio-policy-5cdbc47674-pftgp             1/1     Running   5          9m12s
istio-sidecar-injector-7dd87d7989-kxl92   1/1     Running   0          9m16s
istio-telemetry-6dccd56cf4-cx978          1/1     Running   4          9m14s
istio-tracing-c66d67cd9-n8ms8             1/1     Running   0          9m26s
kiali-8559969566-lgk9w                    1/1     Running   0          9m19s
prometheus-66c5887c86-cr4gm               1/1     Running   0          9m13s
```

## Conclusion

After all these steps it is possible to follow any of the tasks in the istio
documentation. The following tasks are a good place for beginners to start:

- [Request routing](https://istio.io/docs/tasks/traffic-management/request-routing/)
- [Fault injection](https://istio.io/docs/tasks/traffic-management/fault-injection/)
- [Traffic shifting](https://istio.io/docs/tasks/traffic-management/traffic-shifting/)
- [Querying metrics](https://istio.io/docs/tasks/observability/metrics/querying-metrics/)
- [Visualizing metrics](https://istio.io/docs/tasks/observability/metrics/using-istio-dashboard/)
- [Collecting logs](https://istio.io/docs/tasks/observability/logs/collecting-logs/)
- [Rate limiting](https://istio.io/docs/tasks/policy-enforcement/rate-limiting/)
- [Ingress gateways](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/)
- [Accessing external services](https://istio.io/docs/tasks/traffic-management/egress/egress-control/)
- [Visualizing your mesh](https://istio.io/docs/tasks/observability/kiali/)

=)
