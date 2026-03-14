# Calico bgp configuration with eBPF
## Installtion
Make Talos or RKE2 kubernetes cluster
Create the k8s cluster with CNI none:
```
cluster:
  network:
    cni:
      name: none
```
Install tigera operator:

Please check the bug: https://github.com/aadhilam/k8-networking-calico-containerlab/blob/master/containerlab/08-calico-bgp-lb/calico-cni-config/custom-resources.yaml
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml
```

By default, Calico uses the /var directory to mount cgroups. However, since this path is not writable in Talos Linux, you need to change it to /sys/fs/cgroup.
Use the following command to update the cgroup mount path:
```
kubectl create -f -<<EOF
apiVersion: crd.projectcalico.org/v1
kind: FelixConfiguration
metadata:
  name: default
spec:
  cgroupV2Path: "/sys/fs/cgroup"
EOF
```
Then install calico cni:
```
kubectl create -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Enabled
    linuxDataplane: BPF
    bpfNetworkBootstrap: Enabled
    kubeProxyManagement: Enabled
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
  kubeletVolumePluginPath: None
EOF
```

Now nodes will be ready:
```
tanzu@TST-Tazu-BootSrap:~/talos/calico$ kubectl get nodes
NAME       STATUS   ROLES           AGE     VERSION
cp01       Ready    control-plane   5h58m   v1.35.0
worker01   Ready    <none>          5h24m   v1.35.0
tanzu@TST-Tazu-BootSrap:~/talos/calico$ 
```
also all pods:
```
tanzu@TST-Tazu-BootSrap:~/talos/calico$ kubectl get nodes
NAME       STATUS   ROLES           AGE     VERSION
cp01       Ready    control-plane   5h58m   v1.35.0
worker01   Ready    <none>          5h24m   v1.35.0
tanzu@TST-Tazu-BootSrap:~/talos/calico$
```
## Configuration
Config of bgp:

```
apiVersion: crd.projectcalico.org/v1
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  asNumber: 65420
  nodeToNodeMeshEnabled: false
  serviceLoadBalancerIPs:
    - cidr: 10.74.47.0/27
```
Config of bgp peer:
```
tanzu@TST-Tazu-BootSrap:~/talos/calico$ cat bgp-peer-calico.yml 
apiVersion: crd.projectcalico.org/v1 
kind: BGPPeer
metadata:
  name: bgppeer-arista
spec:
  peerIP: 10.74.45.1           # IP address of your Arista switch
  asNumber: 65120              # AS number of the Arista switch
  #nodeSelector: all()
  nodeSelector: role == "bgp"
  filters:
    - deny-pod-routes  
```

BGP filter to exculdes pod or service cidrs:
```
tanzu@TST-Tazu-BootSrap:~/talos/calico$ cat bgp-filter.yml 
apiVersion: crd.projectcalico.org/v1
kind: BGPFilter
metadata:
  name: deny-pod-routes
spec:
  exportV4:
  - action: Reject
    matchOperator: In
    cidr: 10.244.0.0/16
```
Configure calico LB:
```
tanzu@TST-Tazu-BootSrap:~/talos/calico$ cat calico-lb.yml 
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
 name: loadbalancer-ip-pool
spec:
 cidr: 10.74.47.0/27
 blockSize: 27
 natOutgoing: true
 disabled: false
 assignmentMode: Automatic
 allowedUses:
  - LoadBalancer
```
Nginx service having static IP type loadbalancer:
```
tanzu@TST-Tazu-BootSrap:~/talos/calico$ cat calico-nginx-lb-svc.yml 
apiVersion: v1
kind: Service
metadata:
  name: lb-nginx-service
  namespace: default
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      name: default
  type: LoadBalancer
  loadBalancerIP: 10.74.47.1
```
## Checking
Check the service status:
```
tanzu@TST-Tazu-BootSrap:~/talos/calico$ kubectl get svc
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP      10.96.0.1        <none>        443/TCP        6h8m
lb-nginx-service   LoadBalancer   10.111.237.174   10.74.47.1    80:30880/TCP   33m
tanzu@TST-Tazu-BootSrap:~/talos/calico$ 
```
BGP configurtation in router:
```
bgp 65120
 #
 ip vpn-instance VPN-QV-OM
  group talos-cillium external
  peer 10.74.45.100 as-number 65420
  peer 10.74.45.100 group talos-cillium
  peer 10.74.45.100 source-address 10.74.45.1
  peer 10.74.45.101 as-number 65420
  peer 10.74.45.101 group talos-cillium
  peer 10.74.45.101 source-address 10.74.45.1
  peer 10.74.45.102 as-number 65420
  peer 10.74.45.102 group talos-cillium
  peer 10.74.45.102 source-address 10.74.45.1
  #
  address-family ipv4 unicast
   balance 8
   peer 10.74.45.100 enable
   peer 10.74.45.101 enable
   peer 10.74.45.102 enable
```
Check BGP status from swtich:
```
<GZP-T_TOR1>display bgp peer ipv4 vpn-instance VPN-QV-OM

 BGP local router ID: 10.74.40.6
 Local AS number: 65120
 Total number of peers: 3                 Peers in established state: 1

  * - Dynamically created peer
  Peer                    AS  MsgRcvd  MsgSent OutQ PrefRcv Up/Down  State

  10.74.45.100         65420        0        0    0       0 00:16:19 Connect    
  10.74.45.101         65420       45       45    0       1 00:34:40 Established
  10.74.45.102         65420        0        0    0       0 0145h31m Connect 
```
Check BGP in calico pod:
```
tanzu@TST-Tazu-BootSrap:~/talos/calico$ kubectl exec -it -n calico-system calico-node-25s7q -- birdcl show protocols
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), ebpf-bootstrap (init), install-cni (init)
BIRD v0.3.3+birdv1.6.8 ready.
name     proto    table    state  since       info
static1  Static   master   up     13:34:55    
kernel1  Kernel   master   up     13:34:55    
device1  Device   master   up     13:34:55    
direct1  Direct   master   up     13:34:55    
Node_10_74_45_1 BGP      master   up     13:54:33    Established  
```
Check route:
```
<GZP-T_TOR1>display ip routing-table vpn-instance VPN-QV-OM protocol bgp

Summary count : 1

BGP Routing table status : <Active>
Summary count : 1

Destination/Mask   Proto   Pre Cost        NextHop         Interface
10.74.47.0/27      BGP     255 0           10.74.45.101    Vlan171

BGP Routing table status : <Inactive>
Summary count : 0
```

Check service reachability:
```
tanzu@TST-Tazu-BootSrap:~/talos/calico$ curl 10.74.47.1
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy, 
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional 
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

