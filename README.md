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

Used the `kubectl apply -f nginx-pod.yaml` to run the pod 

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

- PVs (Persistent Volumes) are cluster resources, while PVCs (Persistent Volume Claims) serve as requests for those resources and act as claim checks. K0ps alows for the configuration and management of Kubernetes clusters. By default, in a k0ps-managed clusterthere may be a default storageClass defined that enables the dynamic creation of PVs. These PVs, in turn, create volumes that can be utilized by Pods within the cluste







