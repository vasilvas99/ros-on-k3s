This is an abridged version (with minor modifications for k3s) of the great article by Canonical:

### [ROS 2 and Kubernetes Basics](https://ubuntu.com/blog/exploring-ros-2-with-kubernetes) 

ROS 2 uses a DDS implenetation under the hood for node-to-node communication and relies on UDP multicasting to do node-discovery. This is not much of an issue when running ROS 2-nodes in single docker docker container, but is in a somewhat of a conflict with how k8s does networking - every pod gets acces to the same [eth0](https://ubuntu.com/blog/multus-how-to-escape-the-kubernetes-eth0-prison) i.f. 

To resolve the issue we can use [macvlan](https://backreference.org/2014/03/20/some-notes-on-macvlanmacvtap/) . This allows us to have multiple (as many as we need) virtual NICs with their own unique MAC-addresses that are "slaves" (bridged to) the host eth0 interface. That way a seperate virtual NIC can be assigned to each pod, avoiding conflicts when it comes to UDP multicast.

Since k3s does not support **macvlan**  by default we need a k8s-CNI plugin to extend it's capabilities.
For this reason we need the [Multus-CNI plugin](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md). 

# Installing the Canonical Demo on a Leda-QEMU image

The Canonical article provides a sample kubectl .yaml file that can be used as a great starting point.

### [Talker-listener demo](https://github.com/canonical/robotics-blog-k8s/blob/main/ros-talker-listener-demo.yaml)  

The config itself is very readable, but in summary, it sets-up the **macvlan** network, two "talker pods" that publish to the "microk8s_chatter" ROS2-topic and one "listener pod" that subscribes to that ROS2-topic, logging the messages it reads there.

To install on Leda __manually__, keeping in mind that the Leda image does not have __git__ :

1) Run a Leda QEMU image that has access to the internet.
3) Install Multus CNI:
```bash
wget https://github.com/k8snetworkplumbingwg/multus-cni/archive/refs/heads/master.zipunzip master.zip 
cd multus-cni-master/
cat ./deployments/multus-daemonset-thick.yml | kubectl apply -f -
```
3) Obtain and apply the ros talker-listener demo yaml:
```bash
wget https://raw.githubusercontent.com/canonical/robotics-blog-k8s/main/ros-talker-listener-demo.yaml
kubectl apply -f ros-talker-listener-demo.yaml
```
4) Wait for all container images to be pulled and for them to start-up.
5) Look at the listener logs, there should be entries from two seperate talkers:
```bash 
root@leda-3502:~/multus-cni-master# kubectl logs --follow -l app=ros-listener
[INFO] [1666863960.859851649] [minimal_subscriber]: listener:ros-listener-deployment-575bfddd-jvm5j:1: "talker:ros-talker-deployment-6c447f496c-s274q:1: 3362"
[INFO] [1666863962.177914492] [minimal_subscriber]: listener:ros-listener-deployment-575bfddd-jvm5j:1: "talker:ros-talker-deployment-6c447f496c-8d9v9:1: 3333"
```

# Important notes:
- Multus CNI is nice :)
- The macvlan setup in the yaml file should be modified to fit our network requirements
- Every ROS2 node should be in its own container, built `FROM ros:<release>`
- Every pod should only have a **single container inside it** since having multiple containers in a pod (even if they are for the same ROS2 node) leads to unpredictable multicasting issues
- This workflow (Multus + macvlan) can prorably be easily transferred to other DDS-based applications that rely on UDP multicasting 

# [Eclipse Muto](https://github.com/eclipse-muto/agent) 


 - It seems that Muto is based on ROS(1) which uses some custom stuff and not DDS as middleware for inter-node comms
 - This means that concrete testing with ROS(1) should be done.
 - ROSv1 uses a roscore "master" which serves as a rendezvous server.
 - The general architecture for running roscore-talker-listener in k3s should then be pretty similar to how we've implemented the carsim-driversim-kuksa communication 

Example ROS1 talker and listener:

```
https://raw.githubusercontent.com/ros/ros_tutorials/kinetic-devel/rospy_tutorials/001_talker_listener/listener.py
```

```
https://raw.githubusercontent.com/ros/ros_tutorials/kinetic-devel/rospy_tutorials/001_talker_listener/talker.py
```

- Have a  `FROM ros:noetic` container that runs the `roscore` command with a known hostname
- All other containers are with nodes are `FROM ros:noetic` should have the `ROS_MASTER_URI=http://<core_hostname>:11311` environmental variable exported
- _Note_: similar to the carsim demo the containers should be able to resolve the roscore-container hostname (custom network or host networking)