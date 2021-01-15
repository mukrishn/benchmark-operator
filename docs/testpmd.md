# TestPMD 

[TestPMD](https://doc.dpdk.org/guides/testpmd_app_ug/index.html) is anapplication  used to test the DPDK in a packet forwarding mode, here this application run inside a pod with SRIOV VF.

## Pre-requisites

This workload assumes a healthy cluster is runing with SR-IOV and Performance Add-On operators. 

* [SR-IOV](https://docs.openshift.com/container-platform/4.6/networking/hardware_networks/installing-sriov-operator.html) operator is required to create/assign VFs to the pods
* [PAO](https://docs.openshift.com/container-platform/4.6/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes.html) is required to prepare the worker node with Hugepages for DPDK application and isolate CPUs for workloads. 

### Sample SR-IOV configuration

These are used to create VFs and SRIOV networks, interface might vary

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: testpmd-policy
  namespace: openshift-sriov-network-operator
spec:
  deviceType: vfio-pci
  nicSelector:
    pfNames:
     - ens2f1
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  numVfs: 20
  resourceName: intelnics
```

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetwork
metadata:
  name: testpmd-sriov-network
  namespace: openshift-sriov-network-operator
spec:
  ipam: |
    {
      "type": "host-local",
      "subnet": "10.57.1.0/24",
      "rangeStart": "10.57.1.100",
      "rangeEnd": "10.57.1.200",
      "routes": [{
        "dst": "0.0.0.0/0"
      }],
      "gateway": "10.57.1.1"
    }
  spoofChk: "on"
  trust: "on"
  resourceName: intelnics
  networkNamespace: my-ripsaw

```

### Sample PAO configuration

The CPU and socket information might vary, use the below as an example. 

Add label to the MCP pool before creating the performance profile.
```sh
# oc label mcp worker machineconfiguration.openshift.io/role=worker
```

```yaml
---
apiVersion: performance.openshift.io/v1
kind: PerformanceProfile
metadata:
   name: testpmd-performance-profile-0
spec:
   cpu:
      isolated: "6,8,10,12,14,16,18,20,22,24,26,28"
      reserved: "0-5,7,9,11,13,15,17,19,21,23,25,27,29"
   hugepages:
      defaultHugepagesSize: "1G"
      pages:
         - size: "1G"
           count: 16
           node: 0
   numa:
      topologyPolicy: "best-effort"
   nodeSelector:
      node-role.kubernetes.io/worker: ""
   realTimeKernel:
     enabled: false
   additionalKernelArgs:
   - nosmt
   - tsc=reliable
```

## Running standalone TestPMD

A minimum configuration to run testpmd with in a pod, 

```yaml
apiVersion: ripsaw.cloudbulldozer.io/v1alpha1
kind: Benchmark
metadata:
  name: testpmd-benchmark
  namespace: my-ripsaw
spec:
  clustername: myk8scluster
  workload:
    name: testpmd
    args:
      privileged: true
      pin: false
      pin_testpmd: "node-0"      
      networks:
        testpmd:
          - name: testpmd-sriov-network
            count: 2  # Interface count, Min 2
            mac: # List length should match interface count, Optional input. 
              - 60:60:00:f4:47:06 # Random MAC address for SRIOV-VF interface
              - 60:60:00:f4:47:07 # Random MAC address for SRIOV-VF interface
      peer_mac: # List length should match testpmd network interface count, Mandatory input.
        - 50:50:00:f4:47:08 # MAC address of Traffic generator interface
        - 50:50:00:f4:47:09 # MAC address of Traffic generator interface
```

Additional advanced pod and testpmd specification can also be supplied in CR definition yaml
for more description refer [here](https://manpages.debian.org/experimental/dpdk/testpmd.1.en.html)

```yaml
  workload:
    ...
    pod_hugepage_1gb_count: 4Gi 
    pod_memory: "1000Mi"
    pod_cpu: 6
    socket_memory: 1024
    memory_channels: 4
    forwarding_cores: 4
    rx_queues: 1
    tx_queues: 1
    rx_descriptors: 1024
    tx_descriptors: 1024
    forward_mode: "mac"
    stats_period: 1
```

## Container Images

Pod uses this [image](https://catalog.redhat.com/software/containers/openshift4/dpdk-base-rhel8/5e32be6cdd19c77896004a41?container-tabs=dockerfile&tag=latest&push_date=1610346978000)
it can be overriden by supplying additional inputs in CR definition yaml, provided `testpmd` command will be executable from the image `WORKDIR` with additional environment variables

```yaml
  workload:
    ...
    image_testpmd: "registry.redhat.io/openshift4/dpdk-base-rhel8:v4.6"
    image_pull_policy: "Always"
    environments:
      testpmd_mode: "direct" ## Just an example
```

## Result

Current implemenation, just creates a pod and run testpmd application. At this state, interfaces are ready for traffic IO indefinitely. 
Traffic generation is not part of this workload yet, it will be a future implementation. Manually use [trafficgen](https://github.com/atheurer/trafficgen) for packets generation.
Note down the MAC addresses of testpmd pod for traffic generation.

At the end of this CR, these will be created

```sh
# oc get benchmark 
NAME                TYPE      STATE      METADATA STATE   CERBERUS        UUID                                   AGE
testpmd-benchmark   testpmd   Complete   not collected    not connected   ebe5d7ab-1c88-57bd-b789-b3374a59c4b9   58s
```
```sh
# oc get pods
NAME                                     READY   STATUS    RESTARTS   AGE
testpmd-application-pod-ebe5d7ab-ts9hp   1/1     Running   0          35s
```

Traffic IO can be seen in logs, if any external generator is launched - for example trex or trafficgen

```sh
# oc logs testpmd-application-pod-ebe5d7ab-ts9hp
EAL: Detected 32 lcore(s)
EAL: Detected 2 NUMA nodes
EAL: Auto-detected process type: PRIMARY
EAL: Multi-process socket /tmp/dpdk/pg/mp_socket
EAL: Selected IOVA mode 'VA'
EAL: Probing VFIO support...
EAL: VFIO support initialized
EAL: PCI device 0000:62:0b.1 on NUMA socket 0
EAL:   probe driver: 8086:154c net_i40e_vf
EAL:   using IOMMU type 1 (Type 1)
EAL: PCI device 0000:62:0c.2 on NUMA socket 0
EAL:   probe driver: 8086:154c net_i40e_vf
Auto-start selected
Set mac packet forwarding mode
CLI commands to be read from /tmp/testpmd-cmdline.txt
testpmd: create a new mbuf pool <mbuf_pool_socket_0>: n=187456, size=2176, socket=0
testpmd: preferred mempool ops selected: ring_mp_mc
Configuring Port 0 (socket 0)
Port 0: 60:60:00:F4:47:07
Configuring Port 1 (socket 0)
Port 1: 60:60:00:F4:47:06
Checking link statuses...
Done
Read CLI commands from /tmp/testpmd-cmdline.txt
No commandline core given, start packet forwarding
mac packet forwarding - ports=2 - cores=2 - streams=2 - NUMA support enabled, MP allocation mode: native
Logical Core 8 (socket 0) forwards packets on 1 streams:
  RX P=0/Q=0 (socket 0) -> TX P=1/Q=0 (socket 0) peer=50:50:00:F4:47:09
Logical Core 10 (socket 0) forwards packets on 1 streams:
  RX P=1/Q=0 (socket 0) -> TX P=0/Q=0 (socket 0) peer=50:50:00:F4:47:08

  mac packet forwarding packets/burst=32
  nb forwarding cores=4 - nb forwarding ports=2
  port 0: RX queue number: 1 Tx queue number: 1
    Rx offloads=0x0 Tx offloads=0x0
    RX queue: 0
      RX desc=1024 - RX free threshold=32
      RX threshold registers: pthresh=8 hthresh=8  wthresh=0
      RX Offloads=0x0
    TX queue: 0
      TX desc=1024 - TX free threshold=32
      TX threshold registers: pthresh=32 hthresh=0  wthresh=0
      TX offloads=0x0 - TX RS bit threshold=32
  port 1: RX queue number: 1 Tx queue number: 1
    Rx offloads=0x0 Tx offloads=0x0
    RX queue: 0
      RX desc=1024 - RX free threshold=32
      RX threshold registers: pthresh=8 hthresh=8  wthresh=0
      RX Offloads=0x0
    TX queue: 0
      TX desc=1024 - TX free threshold=32
      TX threshold registers: pthresh=32 hthresh=0  wthresh=0
      TX offloads=0x0 - TX RS bit threshold=32

Port statistics ====================================
  ######################## NIC statistics for port 0  ########################
  RX-packets: 28         RX-missed: 0          RX-bytes:  1792
  RX-errors: 0
  RX-nombuf:  0         
  TX-packets: 28         TX-errors: 0          TX-bytes:  1680

  Throughput (since last show)
  Rx-pps:            2          Rx-bps:         1456
  Tx-pps:            2          Tx-bps:         1368
  ############################################################################

  ######################## NIC statistics for port 1  ########################
  RX-packets: 28         RX-missed: 0          RX-bytes:  1792
  RX-errors: 0
  RX-nombuf:  0         
  TX-packets: 28         TX-errors: 0          TX-bytes:  1680

  Throughput (since last show)
  Rx-pps:            2          Rx-bps:         1456
  Tx-pps:            2          Tx-bps:         1368
  ############################################################################
```
