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
If you are running OpenShift without SystemD, the systemd cgroup must be created manually in order to make SystemD based containers work

~~~
mkdir -p /sys/fs/cgroup/systemd
mount -t cgroup -o none,name=systemd cgroup /sys/fs/cgroup/systemd
mkdir -p /sys/fs/cgroup/systemd/user
echo $$ > /sys/fs/cgroup/systemd/user/cgroup.procs
~~~

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