# PERSISTING-DATA-IN-KUBERNETES
## INTRODUCTION


It is important to recognize that containers are inherently designed to be stateless, meaning that data does not persist within them. This applies even when running containers in Kubernetes pods, unless specific configurations are implemented to support statefulness.

In order to achieve statefulness in Kubernetes, it is crucial to have a clear understanding of the concepts and functionality of volumes, persistent volumes, and persistent volume claims. These components play a vital role in managing and providing persistent storage to Kubernetes applications, allowing them to store and retrieve data across container restarts and rescheduling.


## STEP 1: Setting Up Kubernetes Service With kOps

I utilized the kOps tool to deploy and manage a Kubernetes cluster for this project. https://kops.sigs.k8s.io/getting_started/aws/

![p1](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/1abe3869-1003-4e26-9996-6df0729d72c8)


To deploy the Nginx application, I generated a manifest file specifying the desired configuration and subsequently apply it to the Kubernetes cluster.

```
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF
```

- Used the `kubectl apply -f nginx-pod.yaml` to run the pod 

![p3](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/27c390cb-f8e6-4db4-a6ba-8567705235e1)

- Checked the status of the Pods 

![p2](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/9cfb8a1a-7fe0-4394-84e7-62fe0f71365a)


- In order to ensure compatibility, when creating a volume, it is important to ensure that it exists within the same region and availability zone as the EC2 instance hosting the pod. To check the Availability Zone where the node is running:`kubectl describe node i-a47fda919bdbd90`

![p4](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/e7a16912-4870-407c-a409-13ee3839f581)


To create a volume within the AWS Elastic Block Storage section, it is necessary to create it in the same availability zone (AZ) as the node responsible for running the Nginx pod. This volume will be subsequently mounted into the Nginx pod, enabling persistent storage for the application.

![p5](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/30fb0bc5-38e1-4a65-9d2d-4d0df1d56387)


![p6](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/18778d29-4cc2-4e16-b8b5-50d31272100d)

- Updating the deployment configuration with the volume spec and volume mount


![p7](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/33ee7667-df1d-4c86-89a2-77c268441d9d)


- Run the command `kubectl create -f nginx-pod.yaml`

![p8](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/3fcecd7c-4d1d-472b-a4ec-6c7585a91278)

![p9](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/43aeff2d-7adf-4d0a-a6fc-8a02e8ca1dc9)



- When attempting to access the endpoint by port forwarding the service, you may encounter a 403 error. This occurs because mounting a volume onto a filesystem that already contains data will result in the automatic deletion of all existing data. This approach to achieving statefulness is recommended when the mounted volume already contains the desired data that needs to be accessible within the container.


## STEP 3: Managing Volumes Dynamically With PV and PVCs

- PVs (Persistent Volumes) are cluster resources, while PVCs (Persistent Volume Claims) serve as requests for those resources and act as claim checks. K0ps allows for the configuration and management of Kubernetes clusters. By default, in a k0ps-managed cluster there is a default storageClass defined that enables the dynamic creation of PVs. These PVs, in turn, create volumes that can be utilized by Pods within the cluster.

- Verifying that there is a storageClass in the cluster:`$ kubectl get storageclass`

![p14](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/8aa04822-4a00-4dec-996d-988ec4a2fdad)

- Create a manifest file for a PVC, and based on the gp2 storageClass a PV will be dynamically created

```
apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nginx-volume-claim
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 2Gi
      storageClassName: gp2
```

![p15](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/01985736-340c-4540-9ec2-0adb42bc7f8a)

- Checking the setup:`$ kubectl get pvc`

![p16](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/b5be7d69-2aa4-45f2-b85e-335a7b619b6c)

- Create a yaml file nginx-deployment-with-pvc 

![p18](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/bafc1183-5602-4e65-b191-837fa7acec74)

![p19](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/a26327ff-f857-4168-ad9e-d6ba7e6b582d)

- With the new deployment manifest, the `/tmp/dare` directory will be persisted, and any data written in there will be sotred permanetly on the volume, which can be used by another Pod if the current one gets replaced.


- Another approach is to Create two yaml file PV-Claim.yaml and pv-pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
       persistentVolumeClaim:
        claimName: task-pv-claim

  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi

```

![p23](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/0ce184c3-7f51-4dab-85fe-a08bd6541f01)

## STEP 4: Use Of ConfigMap As A Persistent Storage

- A ConfigMap in Kubernetes is used to manage configuration files and other non-confidential data as key-value pairs. ConfigMaps are designed to ensure that the configuration data is decoupled from the Pod's definition, allowing for easier configuration management and flexibility. Importantly, ConfigMaps help ensure that the configuration data is not lost when a Pod is replaced or rescheduled.

- To demonstrate this, the HTML file that came with Nginx will be used.
- Exec into the container and copy the HTML file
-   cat /usr/share/nginx/html/index.html
-  Use configMap to create a file in a volume. 
```

cat <<EOF | tee ./nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: website-index-file
data:
  # file to be mounted inside a volume
  index-file: |
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to Nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to Nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
EOF
```
![p32](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/566db043-229f-45ff-896e-144d3b5a4eb4)

- Updating the deployment file to use the configmap in the volumeMounts section
```
cat <<EOF | tee ./nginx-pod-with-cm.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
          - name: config
            mountPath: /usr/share/nginx/html
            readOnly: true
      volumes:
      - name: config
        configMap:
          name: website-index-file
          items:
          - key: index-file
            path: index.html
EOF
```

- By utilizing a ConfigMap that has been mounted onto the filesystem, the index.html file is no longer ephemeral. This becomes evident when accessing the pod through the `exec` command and listing the contents of the /usr/share/nginx/html directory. The presence of the file in that directory indicates that the index.html content is now sourced from the mounted ConfigMap, ensuring its persistence and availability within the pod.

![p27](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/d65b2ae3-79da-46bb-8613-ee2390cdbd5a)

- Accessing the site will not change anything at this time because the same html file is being loaded through configmap.

- But if you make any change to the content of the html file through the configmap, and restart the pod, all your changes will persist.

- List the available configmaps using command `kubectl get cm`

![p28](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/6eb53142-f112-4b71-b324-fa653713b527)

- Will Edit the configmap manifest file, using this command `kubectl edit cm website-index-file`

![p29](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/4a8cf39c-ea4a-4ddd-a03e-855e677f44b1)


- Without restarting the pod, the site should be loaded automatically.
- Performing the Port forwarding command and accessing it through the browser:

![p31](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/c8c2f412-9a43-4886-a3c9-60a3a45eca4e)

![p30](https://github.com/busolagbadero/PERSISTING-DATA-IN-KUBERNETES/assets/94229949/16374518-9096-4bed-b535-47c4978e3ae8)








