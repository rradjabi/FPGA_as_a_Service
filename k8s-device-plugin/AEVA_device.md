Instructions for the experimental environment for AEVA Kubernetes Device Plugin. 

# K3s setup
Install a micro cluster (worker can be master) using K3s.
```
curl -sfL https://get.k3s.io | sh -
```

## K3s management
Stop k3s service
```
sudo systemctl stop k3s
```
Forcefully stop k3s
```
sudo /usr/local/bin/k3s-killall.sh
```
Start k3s
```
sudo systemctl start k3s
```
See pods
```
sudo kubectl get nodes
```
Delete fpga-device-plugin daemonset
```
kubectl delete daemonset fpga-device-plugin-daemonset -n kube-system
```

# Building from FPGA_as_a_Service K8s repo

Build within a golang docker
```
docker run -ti golang:1.18.3 bash
```

Setup inside golang docker
```
apt update
apt install vim -y
apt-get install -y ca-certificates openssl
```

Clone repo
```
git clone https://github.com/Xilinx/FPGA_as_a_Service.git
```
or
```
git clone https://github.com/rradjabi/FPGA_as_a_Service.git
```

Build plugin
```
cd FPGA_as_a_Service/k8s-device-plugin
go mod init k8s-device-plugin
./build
```

Copy plugin at `bin/k8s-device-plugin` to host and run binary directly with `sudo ./k8s-device-plugin`.

You should be able to see the device as a resource in the cluster as `amd.com/aeva_vmss_e1.s_v0-0` when you run `sudo kubectl describe nodes`:
```
...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource                     Requests    Limits
  --------                     --------    ------
  cpu                          200m (2%)   0 (0%)
  memory                       140Mi (0%)  170Mi (0%)
  ephemeral-storage            0 (0%)      0 (0%)
  hugepages-1Gi                0 (0%)      0 (0%)
  hugepages-2Mi                0 (0%)      0 (0%)
  amd.com/aeva_vmss_e1.s_v0-0  0           0
...
```
If you stop the `k8s-device-plugin` binary from running, you will notice that `Allocatable: amd.com/aeva_vmss_e1.s_v0-0` value goes from `1` to `0`. This is an easy way to confirm it is working.

## Launch a pod that requires the resource
Next we want to deploy a pod (container app) that requires a resource.

k8s-device-plugin repo has a sample pod 'dp-pod.yml'
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: xilinxatg/fpga-verify:latest
    resources:
      limits:
        amd.com/MA35: 1
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
```
In this, it finds the docker from dockerhub `xilinxatg/fpga-verify:latest`, and it requires a resource `amd.com/MA35` with a quantity of `1`.

Modify this to use `amd.com/aeva_vmss_e1.s_v0-2`.
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: ubuntu:20.04
    resources:
      limits:
        amd.com/aeva_vmss_e1.s_v0-2: 1
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
```

Create a pod requiring the AEVA resource with `sudo kubectl apply -f dp-pod.yml`. And check status of running pod with `sudo kubectl describe pods`.
```
Name:             ubuntu-aeva-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             aeva-prod0/10.23.81.232
Start Time:       Tue, 12 Mar 2024 16:41:29 -0700
Labels:           <none>
Annotations:      <none>
Status:           Running
IP:               10.42.0.14
IPs:
  IP:  10.42.0.14
Containers:
  ubuntu-aeva-pod:
    Container ID:  containerd://8d46f179de04318a2def892f209e3cdb03cc2de4e1832300c17c18c0e068e478
    Image:         ubuntu:20.04
    Image ID:      docker.io/library/ubuntu@sha256:80ef4a44043dec4490506e6cc4289eeda2d106a70148b74b5ae91ee670e9c35d
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
    Args:
      -c
      while true; do echo hello; sleep 10;done
    State:          Running
      Started:      Tue, 12 Mar 2024 16:41:30 -0700
    Ready:          True
    Restart Count:  0
    Limits:
      amd.com/aeva_vmss_e1.s_v0-2:  1
    Requests:
      amd.com/aeva_vmss_e1.s_v0-2:  1
    Environment:                    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-gsvpw (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-gsvpw:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  6m44s  default-scheduler  Successfully assigned default/ubuntu-aeva-pod to aeva-prod0
  Normal  Pulled     6m43s  kubelet            Container image "ubuntu:20.04" already present on machine
  Normal  Created    6m43s  kubelet            Created container ubuntu-aeva-pod
  Normal  Started    6m43s  kubelet            Started container ubuntu-aeva-pod
```
