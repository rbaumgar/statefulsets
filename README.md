# OpenShift example for StateFulSets
Tested mit OpenShift and Minishift v3.5.5.31.

This new controller allows for the deployment of application types that require changes to their configuration or deployment count (instances) to be done in a specific and ordered manner.

Supported:

- Declaration of the Ordinal Index.
- Stable network ID nomenclature.
- Controlled or manual handling of PVs.
- Sequence control at deployment time.
- Ordered control during scale up or scale down, based on instance status.

See https://docs.openshift.com/container-platform/3.5/release_notes/ocp_3_5_release_notes.html#ocp-35-statefulsets

# Prerequisits
Because the Zookeeper image needs root, so OpenShift admin has to set scc...
e.g. projects = statef

    $ oc project statef

Create a new service account

    $ oc create serviceaccount useroot 

Add service account to security context constraint anyuid(as admin!)

    $ oc adm policy add-scc-to-user anyuid -z useroot -n statef

# Creating a ZooKeeper Ensemble
Creating an ensemble is as simple as using oc create to generate the objects stored in the manifest.

    $ oc create -f zookeeper.yaml
    service "zk-hs" created
    service "zk-cs" created
    statefulset "zk" created

When you create the manifest, the StatefulSet controller creates each Pod, with respect to its ordinal, and waits for each to be Running and Ready prior to creating its successor.

    $ oc get pod -w
    NAME      READY     STATUS              RESTARTS   AGE
    zk-0      0/1       ContainerCreating   0          10s
    zk-0      0/1       Running   0         32s
    zk-0      1/1       Running   0         51s
    zk-1      0/1       Pending   0         0s
    zk-1      0/1       Pending   0         0s
    zk-1      0/1       ContainerCreating   0         0s
    zk-1      0/1       Running   0         11s
    zk-1      1/1       Running   0         30s
    zk-2      0/1       Pending   0         0s
    zk-2      0/1       Pending   0         0s
    zk-2      0/1       ContainerCreating   0         0s
    zk-2      0/1       Running   0         9s
    zk-2      1/1       Running   0         21s
   
Examining the hostnames of each Pod in the StatefulSet, you can see that the Pods’ hostnames also contain the Pods’ ordinals.

    $ for i in 0 1 2; do oc exec zk-$i -- hostname; done
    zk-0
    zk-1
    zk-2
   
ZooKeeper stores the unique identifier of each server in a file called “myid”. The identifiers used for ZooKeeper servers are just natural numbers. For the servers in the ensemble, the “myid” files are populated by adding one to the ordinal extracted from the Pods’ hostnames.

    $ for i in 0 1 2; do echo "myid zk-$i";oc  exec zk-$i -- cat /var/lib/zookeeper/data/myid; done
    myid zk-0
    1
    myid zk-1
    2
    myid zk-2
    3

Each Pod has a unique network address based on its hostname and the network domain controlled by the zk-headless Headless Service.

    $ for i in 0 1 2; do oc exec zk-$i -- hostname -f; done
    zk-0.zk-hs.statef.svc.cluster.local
    zk-1.zk-hs.statef.svc.cluster.local
    zk-2.zk-hs.statef.svc.cluster.local

The combination of a unique Pod ordinal and a unique network address allows you to populate the ZooKeeper servers’ configuration files with a consistent ensemble membership.

    $ oc exec zk-0 -- cat /opt/zookeeper/conf/zoo.cfg
    #This file was autogenerated DO NOT EDIT
    clientPort=2181
    dataDir=/var/lib/zookeeper/data
    dataLogDir=/var/lib/zookeeper/data/log
    tickTime=2000
    initLimit=10
    syncLimit=5
    maxClientCnxns=60
    minSessionTimeout=4000
    maxSessionTimeout=40000
    autopurge.snapRetainCount=3
    autopurge.purgeInteval=12
    **server.1=zk-0.zk-hs.statef.svc.cluster.local:2888:3888**
    **server.2=zk-1.zk-hs.statef.svc.cluster.local:2888:3888**
    **server.3=zk-2.zk-hs.statef.svc.cluster.local:2888:3888**

StatefulSet lets you deploy ZooKeeper in a consistent and reproducible way. You won’t create more than one server with the same id, the servers can find each other via a stable network addresses, and they can perform leader election and replicate writes because the ensemble has consistent membership.


# Verify Zookeeper Ensemble
The simplest way to verify that the ensemble works is to write a value to one server and to read it from another. You can use the “zkCli.sh” script that ships with the ZooKeeper distribution, to create a ZNode containing some data.

    $ oc exec zk-0 zkCli.sh create /hello world
    Connecting to localhost:2181
    ...
    
    WATCHER::
    
    WatchedEvent state:SyncConnected type:None path:null
    Created /hello
    
You can use the same script to read the data from another server in the ensemble.
    
    $ oc  exec zk-1 zkCli.sh get /hello
    Connecting to localhost:2181
    ...
    
    WATCHER::
    
    WatchedEvent state:SyncConnected type:None path:null
    world
    ...
    
# Test Failover
Once you have all 3 nodes running, you can test failover by killing the leader.

    $ oc delete pod zk-2

    $ oc run --attach bbox --image=busybox --restart=Never -- sh -c 'while true; do for i in 0 1 2; do echo zk-$i $(echo stats | nc zk-$i.zk-hs 2181 | grep Mode); sleep 10; done; done'
    If you don't see a command prompt, try pressing enter.
    zk-0 Mode: follower
    zk-1 Mode: follower
    zk-2 Mode: leader
    ...
    zk-0 Mode: follower
    zk-1 Mode: leader
    nc: bad address 'zk-2.zk-hs'
    zk-2
    zk-0 Mode: follower
    zk-1 Mode: leader
    nc: bad address 'zk-2.zk-hs'
    zk-2
    zk-0 Mode: follower
    zk-1 Mode: leader
    zk-2 Mode: follower
    ...

# Delete Zookeeper Ensemble
You can take the ensemble down by deleting the zk StatefulSet.

    $ oc delete statefulset zk
    statefulset "zk" deleted
    $ oc delete svc zk-cs zk-hs
    service "zk-cs" deleted
    service "zk-hs" deleted

The cascading delete destroys each Pod in the StatefulSet, with respect to the reverse order of the Pods’ ordinals, and it waits for each to terminate completely before terminating its predecessor.

    $ oc get pods -w -l app=zk
    NAME      READY     STATUS    RESTARTS   AGE
    zk-0      1/1       Running   0          14m
    zk-1      1/1       Running   0          13m
    zk-2      1/1       Running   0          12m
    NAME      READY     STATUS        RESTARTS   AGE
    zk-2      1/1       Terminating   0          12m
    zk-1      1/1       Terminating   0         13m
    zk-0      1/1       Terminating   0         14m
    zk-2      0/1       Terminating   0         13m
    zk-2      0/1       Terminating   0         13m
    zk-2      0/1       Terminating   0         13m
    zk-1      0/1       Terminating   0         14m
    zk-1      0/1       Terminating   0         14m
    zk-1      0/1       Terminating   0         14m
    zk-0      0/1       Terminating   0         15m
    zk-0      0/1       Terminating   0         15m
    zk-0      0/1       Terminating   0         15m
    
