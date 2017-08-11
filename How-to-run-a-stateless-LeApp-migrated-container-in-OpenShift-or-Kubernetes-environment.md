  By stateless container we understand a container which can be created and destroyed without loosing any important data related to its state. Such kind of data might be a file created and saved within the container file system, or database records etc. This doesn't mean, that there can't be a database or file server running in the container, but all changes done during the existence of the container will be gone once the container is destroyed.

  This allows to run multiple instances of the very same container as long as they are not sharing any external storage (write access required).

## Example applications 
* VPN server
* Static HTTP/FTP server
* Dynamic HTTP server of which state is stored in external database
* In-memory database

## LeApp to OpenShift/Kubernetes migration
LeApp migrated machine can be migrated to OpenShift/Kubernetes either as one macro-image, or as an image mounting multiple ephemeral volumes (multivolume). For now, we only support "multiple step" migration which involve leapp-tool and helper script. Leapp-tool is used for primary migration of the machine to a container. The helper script will generate required templates for creating an OpenShift/Kubernetes pod and service and in case of macro-image, a Dockerfile will be generated as well.

### Requirements
**Macro-image**
* Private registry in order to pull image on any node or access to a node where the container will be running
* Privileges to mount hostPath (/sys/fs/cgroup) and in memory emptyDir in case of SystemD based container

**Multi-volume**
* HTTPD server accessible from the nodes for distributing the image data
* Privileges to create and mount volumes (type emptyDir)
* Privileges to mount hostPath (/sys/fs/cgroup) and in memory emptyDir in case of SystemD based container

### Migration process
**Step 1) - Migrate machine using leapp tool**
~~~
# leapp-tool migrate-machine --container-name leapp-migrated-container <source-ip-or-hostname> 
~~~

**Step 2. b) - Stateless macro image example (local image only)**

Run [helper tool](https://github.com/GAZDOWN/leapp/tree/kubernetes-stateless/kubernetes/template-generator) and generate macro-image templates
~~~
# ./cli.py macroimage \
    --tcp 9022:22 80:80 \
    --local-image \
    -c leapp-migrated-container \
    --ip 192.168.100.1 \
    --dest ~/templates
~~~

*Example script output*:
~~~
Storing templates to: /home/marcel/templates
Create a tar image of migrated machine:
  tar -czf /home/marcel/templates/leapp-migrated-container.tar.gz -C /var/lib/leapp/macrocontainers/leapp-migrated-container .

Move it to the machine where you want to build the container 
along with generated Dockerfile and execute: 
  docker build -t leapp/leapp-migrated-container .
~~~

~~~
# ls -1 ~/templates

Dockerfile
leapp-migrated-container-pod.yaml
leapp-migrated-container-svc.yaml
~~~

**Step 2. b) Stateless multi-volume image example**

Run [helper tool](https://github.com/GAZDOWN/leapp/tree/kubernetes-stateless/kubernetes/template-generator) and generate multi-volume image templates
~~~
# ./cli.py multivolume \
    --tcp 9022:22 80:80 \
    --image-url http://local.web.server/leapp-migrated.tar.gz \
    -c leapp-migrated-container \
    --ip 192.168.100.1 \
    --dest ~/templates
~~~

*Example script output*:
~~~
Storing templates to: /home/marcel/templates
Create a tar image of migrated machine:
  tar -czf /home/marcel/templates/leapp-migrated.tar.gz -C /var/lib/leapp/macrocontainers/leapp-migrated-container .

Move it to the web server, where it will be available under image-url you've specified 
~~~

~~~
# ls -1 ~/templates

leapp-migrated-container-pod.yaml
leapp-migrated-container-svc.yaml
~~~

**Step 3) Preparing image data**

Follow the steps given by the helper script

**Step 4) Create the pod and service**

At this point, the templates, images and image data should be ready. Proceed with following commands and create pod and service in your OpenShift environment. 

_**NOTE**_: Please edit the pre-generated service template in case you need specific configuration like i.e. load balancer etc.
~~~
oc create -f leapp-migrated-container-pod.yaml
oc create -f leapp-migrated-container-svc.yaml
~~~

## Known issues

**SystemD containers in non-systemd environment**

If you are running OpenShift without SystemD, the systemd cgroup must be created manually in order to make SystemD based containers work

~~~
mkdir -p /sys/fs/cgroup/systemd
mount -t cgroup -o none,name=systemd cgroup /sys/fs/cgroup/systemd
mkdir -p /sys/fs/cgroup/systemd/user
echo $$ > /sys/fs/cgroup/systemd/user/cgroup.procs
~~~

**Replacing of /etc/hosts file**

*/etc/hosts* file will be always replaced by underlying container daemon

## Debugging

It might happen, that the container will be failing to start. The best way of finding what is the issue is checking the logs of each failing pod, respectively container in the pod. 

### Debugging steps:

**Step 1) List the pods**

*CMD:*
~~~
oc get pods  
~~~
*Example*:
~~~
# oc get pods                                                                   
NAME                                                  READY     STATUS       RESTARTS   AGE
leapp-container-leapp-tests-centos7-guest-httpd-pod   0/1       Init:Error   0          6s
test-systemd-1-tt3zh                                  1/1       Running      0          1m
~~~
We can see, that leapp-container-leapp-tests-centos7-guest-httpd-pod pod has some issues during Init start.

**Step 2) Get information about the pod**

*CMD*:
~~~
oc describe pod <pod-name>
~~~
*Example*:
~~~
# oc describe pod leapp-container-leapp-tests-centos7-guest-httpd-pod
name:                   leapp-container-leapp-tests-centos7-guest-httpd-pod
Namespace:              myproject
Security Policy:        privileged
Node:                   192.168.42.190/192.168.42.190
Start Time:             Fri, 11 Aug 2017 13:55:31 +0200
Labels:                 name=leapp-container-leapp-tests-centos7-guest-httpd-label
Status:                 Pending
IP:                     172.17.0.2
Controllers:            <none>
Init Containers:
  container-leapp-tests-centos7-guest-httpd-init:
    Container ID:       docker://14e7a1606baa985e5ff600344daf2633ef8173af4224051b2eb123c169e58025
    Image:              busybox
    Image ID:           docker-pullable://busybox@sha256:2605a2c4875ce5eb27a9f7403263190cd1af31e48a2044d400320548356251c4
    Port:
    Command:
      sh
      -c
      mkdir -p /work-dir && wget -O /work-dir/image.tar http://10.10.10.89:8080/test.tar.gz &&cd /work-dir && tar -xf image.tar &&mkdir -p /work-dir/var/log/journal/$(cat /work-dir/etc/machine-id)
    State:              Waiting
      Reason:           CrashLoopBackOff
    Last State:         Terminated
      Reason:           Error
      Exit Code:        1
      Started:          Fri, 11 Aug 2017 13:56:32 +0200
      Finished:         Fri, 11 Aug 2017 13:56:32 +0200
    Ready:              False
    Restart Count:      3
    Volume Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-8r5ng (ro)
      /work-dir/boot from container-leapp-tests-centos7-guest-httpd-boot-vol (rw)
      ...
    Environment Variables:      <none>
Containers:
  container-leapp-tests-centos7-guest-httpd:
    Container ID:
    Image:              gazdown/leapp-scratch:7
    Image ID:
    Ports:              80/TCP, 22/TCP
    State:              Waiting
      Reason:           PodInitializing
    Ready:              False
    Restart Count:      0
    Volume Mounts:
      /boot from container-leapp-tests-centos7-guest-httpd-boot-vol (rw)
      ...
    Environment Variables:
      container:        docker
Conditions:
  Type          Status
  Initialized   False 
  Ready         False 
  PodScheduled  True 
Volumes:
  container-leapp-tests-centos7-guest-httpd-cgroup-vol:
    Type:       HostPath (bare host directory volume)
    Path:       /sys/fs/cgroup
  container-leapp-tests-centos7-guest-httpd-run-vol:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
  container-leapp-tests-centos7-guest-httpd-tmp-vol:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
  container-leapp-tests-centos7-guest-httpd-boot-vol:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
  ...
  default-token-8r5ng:
    Type:       Secret (a volume populated by a Secret)
    SecretName: default-token-8r5ng
QoS Class:      BestEffort
Tolerations:    <none>
Events:
  FirstSeen     LastSeen        Count   From                            SubObjectPath                                                           Type            Reason          Message
  ---------     --------        -----   ----                            -------------                                                           --------        ------          -------
 ...
  1m            1m              2       {kubelet 192.168.42.190}                                                                                Warning         FailedSync      Error syncing pod, skipping: [failed to "InitContainer" for "container-leapp-tests-centos7-guest-httpd-init" with RunInitContainerError: "init container \"container-leapp-tests-centos7-guest-httpd-init\" exited with 1"
, failed to "StartContainer" for "container-leapp-tests-centos7-guest-httpd-init" with CrashLoopBackOff: "Back-off 10s restarting failed container=container-leapp-tests-centos7-guest-httpd-init pod=leapp-container-leapp-tests-centos7-guest-httpd-pod_myproject(f9967b88-7e8b-11e7-bbe5-4a4b56578cb1)"
]
  57s   57s     1       {kubelet 192.168.42.190}        spec.initContainers{container-leapp-tests-centos7-guest-httpd-init}     Normal  Started         Started container with docker id 662bbe40346a
  57s   57s     1       {kubelet 192.168.42.190}        spec.initContainers{container-leapp-tests-centos7-guest-httpd-init}     Normal  Created         Created container with docker id 662bbe40346a; Security:[seccomp=unconfined]
  55s   40s     2       {kubelet 192.168.42.190}                                                                                Warning FailedSync      Error syncing pod, skipping: [failed to "InitContainer" for "container-leapp-tests-centos7-guest-httpd-init" with RunInitContainerError: "init container \"container-leapp-tests-centos7-guest-httpd-init\" exited with 1"
, failed to "StartContainer" for "container-leapp-tests-centos7-guest-httpd-init" with CrashLoopBackOff: "Back-off 20s restarting failed container=container-leapp-tests-centos7-guest-httpd-init pod=leapp-container-leapp-tests-centos7-guest-httpd-pod_myproject(f9967b88-7e8b-11e7-bbe5-4a4b56578cb1)"
]
  1m    27s     4       {kubelet 192.168.42.190}        spec.initContainers{container-leapp-tests-centos7-guest-httpd-init}     Normal  Pulling         pulling image "busybox"
  24s   24s     1       {kubelet 192.168.42.190}        spec.initContainers{container-leapp-tests-centos7-guest-httpd-init}     Normal  Started         Started container with docker id 14e7a1606baa
  1m    24s     3       {kubelet 192.168.42.190}                                                                                Warning FailedSync      Error syncing pod, skipping: failed to "InitContainer" for "container-leapp-tests-centos7-guest-httpd-init" with RunInitContainerError: "init container \"container-leapp-tests-centos7-guest-httpd-init\" exited with 1"

  1m    24s     4       {kubelet 192.168.42.190}        spec.initContainers{container-leapp-tests-centos7-guest-httpd-init}     Normal  Pulled          Successfully pulled image "busybox"
  24s   24s     1       {kubelet 192.168.42.190}        spec.initContainers{container-leapp-tests-centos7-guest-httpd-init}     Normal  Created         Created container with docker id 14e7a1606baa; Security:[seccomp=unconfined]
  1m    11s     6       {kubelet 192.168.42.190}        spec.initContainers{container-leapp-tests-centos7-guest-httpd-init}     Warning BackOff         Back-off restarting failed docker container
  23s   11s     2       {kubelet 192.168.42.190}                                                                                Warning FailedSync      Error syncing pod, skipping: [failed to "InitContainer" for "container-leapp-tests-centos7-guest-httpd-init" with RunInitContainerError: "init container \"container-leapp-tests-centos7-guest-httpd-init\" exited with 1"
, failed to "StartContainer" for "container-leapp-tests-centos7-guest-httpd-init" with CrashLoopBackOff: "Back-off 40s restarting failed container=container-leapp-tests-centos7-guest-httpd-init pod=leapp-container-leapp-tests-centos7-guest-httpd-pod_myproject(f9967b88-7e8b-11e7-bbe5-4a4b56578cb1)"
]
~~~
_**NOTE**_: The above output was shortened

What is in our concern is the error message, returned by container daemon and since we are facing issues during init stage, the logs of init container. To get them, we need to extract the init container name from the description of the pod.

*Container daemon error*
~~~
Error syncing pod, skipping: [failed to "InitContainer" for "container-leapp-tests-centos7-guest-httpd-init" with RunInitContainerError: "init container \"container-leapp-tests-centos7-guest-httpd-init\" exited with 1"
~~~
*Init container name*
~~~
...
Init Containers:
  container-leapp-tests-centos7-guest-httpd-init:
...
~~~

**Step 4) Get logs from the container**

*CMD*:
~~~
oc logs <pod-name> -c <container-name>
~~~
*Example*:
~~~
# oc logs leapp-container-leapp-tests-centos7-guest-httpd-pod -c container-leapp-tests-centos7-guest-httpd-init
Connecting to 10.10.10.89:8080 (10.10.10.89:8080)
wget: can't connect to remote host (10.10.10.89): Connection refused
~~~
From the above we can see, that the file we are trying to download is not available on the server.

## Template examples

### Macro-image
Dockerfile:
~~~
FROM gazdown/leapp-scratch:6
ADD leapp-migrated-container.tar.gz /
~~~

leapp-migrated-container-pod.yaml:
~~~
apiVersion: v1
kind: Pod
metadata:
  annotations: {}
  labels:
    name: leapp-leapp-migrated-container-label
  name: leapp-leapp-migrated-container-pod
spec:
  containers:
  - image: leapp/leapp-migrated-container
    imagePullPolicy: Never
    name: leapp-migrated-container
    ports:
    - containerPort: 22
    - containerPort: 80
    volumeMounts: []
  volumes: []
~~~

leapp-migrated-container-svc.yaml:
~~~
apiVersion: v1
kind: Service
metadata:
  labels:
    name: leapp-leapp-migrated-container-label
  name: leapp-leapp-migrated-container-service
spec:
  externalIPs:
  - 192.168.100.1
  ports:
  - name: leapp-leapp-migrated-container-9022-22-port
    port: 9022
    protocol: TCP
    targetPort: 22
  - name: leapp-leapp-migrated-container-80-80-port
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    name: leapp-leapp-migrated-container-label
~~~

### Multi-volume

leapp-migrated-container-pod.yaml:
~~~
apiVersion: v1
kind: Pod
metadata:
  annotations:
    pod.beta.kubernetes.io/init-containers: '[{"command": ["sh", "-c", "mkdir -p /work-dir
      && wget -O /work-dir/image.tar http://local.web.server/leapp-migrated.tar.gz
      &&cd /work-dir && tar -xf image.tar &&mkdir -p /work-dir/var/log/journal/$(cat
      /work-dir/etc/machine-id)"], "image": "busybox", "name": "leapp-migrated-container-init",
      "volumeMounts": [{"mountPath": "/work-dir/etc", "name": "leapp-migrated-container-etc-vol"}]}]'
  labels:
    name: leapp-leapp-migrated-container-label
  name: leapp-leapp-migrated-container-pod
spec:
  containers:
  - image: gazdown/leapp-scratch:6
    name: leapp-migrated-container
    ports:
    - containerPort: 22
    - containerPort: 80
    volumeMounts:
    - mountPath: /etc
      name: leapp-migrated-container-etc-vol
  volumes:
  - emptyDir: {}
    name: leapp-migrated-container-etc-vol
~~~

leapp-migrated-container-svc.yaml:
~~~
apiVersion: v1
kind: Service
metadata:
  labels:
    name: leapp-leapp-migrated-container-label
  name: leapp-leapp-migrated-container-service
spec:
  externalIPs:
  - 192.168.100.1
  ports:
  - name: leapp-leapp-migrated-container-9022-22-port
    port: 9022
    protocol: TCP
    targetPort: 22
  - name: leapp-leapp-migrated-container-80-80-port
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    name: leapp-leapp-migrated-container-label
~~~