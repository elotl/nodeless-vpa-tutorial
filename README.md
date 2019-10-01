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

Create `hamster` Deployment. The Deployment has two pods, each running two containers with compute-intensive cpu `stress` workload. *Note that the Deployment does not specify cpu/mem requests/limits.*

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

```
$ kubectl get pods

hamster-849d779b4c-m78km   2/2     Running   0          109s
hamster-849d779b4c-rsqzc   2/2     Running   0          109s
```

Verify that the two `hamster` pods are visible as Kiyot Cells.

```
$ kubectl get cells

NAME                                   POD NAME                                  KUBELET                      LAUNCH TYPE   INSTANCE TYPE   INSTANCE ID           IP
cd6a652b-fdc8-4e9f-a792-2eb8f6d6d9b5   default_hamster-849d779b4c-rsqzc          ip-10-0-16-14.ec2.internal   On-Demand     t3.nano         i-0f369bc1feb858549   10.0.23.136
cf28a9ae-dda2-4313-a397-1fcc180cddbd   default_hamster-849d779b4c-m78km          ip-10-0-16-14.ec2.internal   On-Demand     t3.nano         i-06ca7024f6c658e30   10.0.22.6
```

*Notice that we got `t3.nano` (2 vpcu, 0.5GiB) on-demand launch type for `hamster` pods. Since we did not specify cpu/mem request/limit, Nodeless k8s gave us the most cost efficient compute for hamster pods.*

### Step 5: Create VPA for `hamster` deployment

Create VPA with `recreate` option. By using this option, we would like VPA to make recommendation for cpu/mem settings for our `hamster` pod and *stop and recreate* pods in the deployment with the new resource settings.
```
$ kubectl create -f hamster-vpa-recreate.yaml 

verticalpodautoscaler.autoscaling.k8s.io/hamster-vpa created
```

Wait for up to 5 minutes for VPA recommendations to be generated, applied, and `hamster` pods to be recreated.

```
$ kubectl describe vpa
Name:         hamster-vpa
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  autoscaling.k8s.io/v1beta2
Kind:         VerticalPodAutoscaler
Metadata:
  Creation Timestamp:  2019-10-01T22:19:31Z
  Generation:          3
  Resource Version:    2564046
  Self Link:           /apis/autoscaling.k8s.io/v1beta2/namespaces/default/verticalpodautoscalers/hamster-vpa
  UID:                 05a7a362-f51d-4df4-8e50-f779ccb0db53
Spec:
  Target Ref:
    API Version:  apps/v1
    Kind:         Deployment
    Name:         hamster
  Update Policy:
    Update Mode:  Recreate
Status:
  Conditions:
    Last Transition Time:  2019-10-01T22:19:43Z
    Status:                True
    Type:                  RecommendationProvided
  Recommendation:
    Container Recommendations:
      Container Name:  hamster2
      Lower Bound:
        Cpu:     61m
        Memory:  220472891
      Target:
        Cpu:     813m
        Memory:  225384266
      Uncapped Target:
        Cpu:     813m
        Memory:  225384266
      Upper Bound:
        Cpu:           14105m
        Memory:        2996930489
      Container Name:  hamster
      Lower Bound:
        Cpu:     124m
        Memory:  124962559
      Target:
        Cpu:     920m
        Memory:  511772986
      Uncapped Target:
        Cpu:     920m
        Memory:  511772986
      Upper Bound:
        Cpu:           7679m
        Memory:        4271737781
      Container Name:  hamster1
      Lower Bound:
        Cpu:     76m
        Memory:  242826547
      Target:
        Cpu:     1168m
        Memory:  248153480
      Uncapped Target:
        Cpu:     1168m
        Memory:  248153480
      Upper Bound:
        Cpu:     13909m
        Memory:  2955282352
Events:          <none>

```

### Step 6: Verify HPA recommendations are applied

`hamster` pods should be recreated with new resource settings.

```
$ kubectl get cells

NAME                                   POD NAME                                  KUBELET                      LAUNCH TYPE   INSTANCE TYPE   INSTANCE ID           IP
4359e92b-663f-49c7-b387-0f4d65324d66   default_hamster-849d779b4c-bp5qj          ip-10-0-16-14.ec2.internal   On-Demand     c5.large        i-07cf61a97580c3016   10.0.17.175
cb2f59d6-caf7-4c1f-829a-4559943cbdae   default_hamster-849d779b4c-rnrrl          ip-10-0-16-14.ec2.internal   On-Demand     c5.large        i-0149e74302de826fa   10.0.17.64
```

*Notice `hamster` pods are now running on `c5.large` (2 vcpu, 4GiB) launch type which is a compute optimized launch type that fits the VPA-issued resource recommendations.*

### Teardown

Follow [teardown instructions from kubeadm repo](https://github.com/elotl/kubeadm-aws#teardown).
