# OpenShift example for StateFulSets

Creating a ZooKeeper Ensemble
Creating an ensemble is as simple as using kubectl create to generate the objects stored in the manifest.

   $ oc create -f http://k8s.io/docs/tutorials/stateful-application/zookeeper.yaml
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
   



