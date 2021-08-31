# A-test-on-Pod-communication-between-different-vpcs
This repository is mainly created to test the issue#245

This can be done in 4 steps

Step 1: Create VPC using yaml code.(can named it vpc1.yaml)
<pre>
apiVersion: "mizar.com/v1"
kind: Vpc
metadata:
  name: vpc1
spec:
  ip: "12.0.0.0"
  prefix: "8"
  status: "Init"

</pre>
<b>You can edit the vpc vni to 1 which is used in subnet creation. Output can be like this</b>
<pre>
root@centaurus:~/task# kubectl get vpc
NAME   IP         PREFIX   VNI   DIVIDERS   STATUS        CREATETIME                   PROVISIONDELAY
vpc0   20.0.0.0   8        1     1          Provisioned   2021-08-31T06:42:43.103859   22.391985
vpc1   12.0.0.0   8        1     1          Provisioned

</pre>

Step 2: Create Subnet using yaml code. (can named it net1.yaml)
<pre>
apiVersion: "mizar.com/v1"
kind: Subnet
metadata:
  name: net1
spec:
  vpc: "vpc1"
  vni: "1"
  ip: "12.2.0.0"
  prefix: "16"
  status: "Init"
  bouncers: 1
</pre>
You can specify the ip range as per your choice . Output can be like this
<pre>
root@centaurus:~/task# kubectl get subnet
NAME   IP         PREFIX   VNI   VPC    STATUS        BOUNCERS   CREATETIME                   PROVISIONDELAY
net0   20.0.0.0   8        1     vpc0   Provisioned   1          2021-08-31T06:42:43.254071   42.269353
net1   12.2.0.0   16       1     vpc1   Provisioned   1

</pre>

Step 3: Pods creation in vpcs. For this, we need to add annotations  in yaml file and can refer this yaml code.(can named it test.yaml)
<pre>
apiVersion: v1
kind: Pod
metadata:
  name: pod-vpc0
  annotations:
    mizar.com/vpc: "vpc0"
    mizar.com/subnet: "net0"
spec:
  containers:
  - name: podvpc1net1
    image: mizarnet/testpod
    ports:
      - containerPort: 443
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-vpc1
  annotations:
    mizar.com/vpc: "vpc1"
    mizar.com/subnet: "net1"
spec:
  containers:
  - name: podvpc1net1
    image: mizarnet/testpod
    ports:
      - containerPort: 443
</pre>
You can verify the ip's of pods creted as per the vpcs
<pre>
root@centaurus:~/task# kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP           NODE                 NOMINATED NODE   READINESS GATES
mizar-daemon-4m4gv               1/1     Running   0          4h1m    172.18.0.2   kind-control-plane   <none>           <none>
mizar-daemon-76jzw               1/1     Running   0          4h1m    172.18.0.4   kind-worker          <none>           <none>
mizar-daemon-lc72c               1/1     Running   0          4h1m    172.18.0.3   kind-worker2         <none>           <none>
mizar-operator-54df7b55d-mrqbf   1/1     Running   0          4h      172.18.0.4   kind-worker          <none>           <none>
pod-vpc0                         1/1     Running   0          6s      20.0.0.85    kind-worker          <none>           <none>
pod-vpc1                         1/1     Running   0          6s      12.2.0.2     kind-worker2         <none>           <none>

</pre>

Step 4: Check if pods can ping each other
for this, you can use "kubectl exec pod_name -- ping ip_address" command.
The output can be like this.
<pre>
PING 12.2.0.2 (12.2.0.2) 56(84) bytes of data.
64 bytes from 12.2.0.2: icmp_seq=1 ttl=64 time=0.370 ms
64 bytes from 12.2.0.2: icmp_seq=2 ttl=64 time=0.249 ms
64 bytes from 12.2.0.2: icmp_seq=3 ttl=64 time=0.191 ms
^C
root@centaurus:~/task# kubectl exec pod-vpc1 -- ping 20.0.0.85
PING 20.0.0.85 (20.0.0.85) 56(84) bytes of data.
64 bytes from 20.0.0.85: icmp_seq=1 ttl=64 time=0.204 ms
64 bytes from 20.0.0.85: icmp_seq=2 ttl=64 time=0.212 ms
64 bytes from 20.0.0.85: icmp_seq=3 ttl=64 time=0.260 ms
^C

</pre>

Here, pod-vpc0 in vpc0 has ip 20.0.0.85 can successfully ping pod-vpc1 in vpc1 which has ip 12.2.0.2
