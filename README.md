# Vertical Pod Autoscaler with Nodeless Kubernetes

### Step 1: Create 1-worker Nodeless Kubernetes cluster

[Follow instructions in this repo](https://github.com/elotl/kubeadm-aws) to create a {1 master, 1 Milpa worker} Nodeless Kubernetes cluster.

Grab k8s master and Milpa worker IPs from terraform output:
```
Outputs:

master_ip = 35.170.81.41
milpa_worker_ips = [
  "3.214.216.137",
]
```

### Step 2: Deploy Kubernetes Metrics Server

Log on to Kubernetes master, verify cluster is up, then deploy Kubernetes Metrics Server:

```
$ ssh -i "myechuri-key2.pem" ubuntu@35.170.81.41
ubuntu@ip-10-0-19-166:~$
```

```
$ kubectl get nodes -o wide
NAME                          STATUS   ROLES          AGE     VERSION   INTERNAL-IP   EXTERNAL-IP     OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-10-0-16-14.ec2.internal    Ready    milpa-worker   3m36s   v1.15.3   10.0.16.14    3.214.216.137   Ubuntu 16.04.6 LTS   4.4.0-1090-aws   containerd://1.2.6
ip-10-0-19-166.ec2.internal   Ready    master         4m      v1.15.3   10.0.19.166   35.170.81.41    Ubuntu 16.04.6 LTS   4.4.0-1090-aws   docker://18.9.7
```

```
git clone https://github.com/mattkelly/metrics-server.git
cd metrics-server
git checkout bugfix/deploy-1.8+
kubectl apply -f deploy/1.8+/
```

### Step 3: Install VPA

[Install VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#installation).

```
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/
./hack/vpa-up.sh
```

Verify VPA and metrics-server are up before proceeding to testing them out.

```
$ kubectl get deployments -n kube-system
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
coredns                    2/2     2            2           10m
kube-proxy                 1/1     1            1           10m
metrics-server             1/1     1            1           5m31s
vpa-admission-controller   1/1     1            1           102s
vpa-recommender            1/1     1            1           102s
vpa-updater                1/1     1            1           103s
```
### Step 4: [Test VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#test-your-installation)

Create `hamster` Deployment with two pods, each running a single container that requests 100 millicores and tries to utilize slightly above 500 millicores. 

```
git clone https://github.com/elotl/nodeless-vpa-tutorial.git
cd nodeless-vpa-tutorial/
kubectl create -f hamster-deployment.yaml
```

Wait for `hamster` Deployment to be `Ready`.
```
$ kubectl get deployment hamster

NAME      READY   UP-TO-DATE   AVAILABLE   AGE
hamster   2/2     2            2           77s
```

Verify that the two `hamster` pods are visible as Kiyot Cells.

```
$ kubectl get cells

NAME                                   POD NAME                                  KUBELET                      LAUNCH TYPE   INSTANCE TYPE   INSTANCE ID           IP
6d3794ec-5532-4637-8219-05dc527fab08   default_hamster-66577ccf9c-lb2ng          ip-10-0-16-14.ec2.internal   On-Demand     t3.nano         i-0f54a5be16bb2c7e5   10.0.25.172
c5624c6e-5665-4030-a97d-59aff59f8fab   default_hamster-66577ccf9c-bk5lg          ip-10-0-16-14.ec2.internal   On-Demand     t3.nano         i-05985098d7c785b6b   10.0.27.164
e9f1c472-0561-4e75-862a-203d7237eea3   kube-system_kube-proxy-76bbf9cff4-7sp4t   ip-10-0-16-14.ec2.internal   On-Demand     c5.large        i-0b8c858d0301300b9   10.0.19.155
```
Notice that we got `t3.nano` instances as launch type for `hamster` pods.

### Step 5: Create VPA for `hamster` deployment

Create VPA with default (`auto`) mode (`recreate` pod upon change in resource recommendations).
```
$ kubectl create -f hamster-vpa-default-auto.yaml 
verticalpodautoscaler.autoscaling.k8s.io/hamster-vpa created
```

Verify VPA provided resource recommendations for `hamster` pods.

```
$ kubectl describe vpa
Name:         hamster-vpa
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  autoscaling.k8s.io/v1beta2
Kind:         VerticalPodAutoscaler
Metadata:
  Creation Timestamp:  2019-09-16T18:02:41Z
  Generation:          2
  Resource Version:    807148
  Self Link:           /apis/autoscaling.k8s.io/v1beta2/namespaces/default/verticalpodautoscalers/hamster-vpa
  UID:                 1756fc5b-847a-44bd-908c-d86defa56bda
Spec:
  Target Ref:
    API Version:  apps/v1
    Kind:         Deployment
    Name:         hamster
Status:
  Conditions:
    Last Transition Time:  2019-09-16T18:02:43Z
    Status:                True
    Type:                  RecommendationProvided
  Recommendation:
    Container Recommendations:
      Container Name:  hamster
      Lower Bound:
        Cpu:     181m
        Memory:  262144k
      Target:
        Cpu:     224m
        Memory:  262144k
      Uncapped Target:
        Cpu:     224m
        Memory:  262144k
      Upper Bound:
        Cpu:     240m
        Memory:  262144k
Events:          <none>
```

### Step 6: Verify HPA recommendations

In 5 minutes, `hamster` pods should be utilizing around 500m CPU.

```
$ kubectl top pod
NAME                       CPU(cores)   MEMORY(bytes)   
hamster-66577ccf9c-h9k2j   501m         1Mi             
hamster-66577ccf9c-jqkwh   502m         0Mi   
```

### Teardown

Follow [teardown instructions from kubeadm repo](https://github.com/elotl/kubeadm-aws#teardown).
