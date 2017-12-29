## Running a python Application on Kubernetes

Kubernetes is an opensource platform that features deployment, maintenance and scaling mechanisms that help us simplify the 
management of containerized python applications while giving us the portability, extensibility and self healing capabilities for 
our applications. 

Whether you want to run a simple python application or a complex one, kubernetes can quickly and efficiently help
you deploy and scale your applications, seamlessly roll out new features while limiting resources to only required resources.
In this blog, I will cover a holistic process of deploying a simple python application to kubernetes. Among the topics i will cover 
include:

+ Creating python container images
+ Publishing the container images to an image registry
+ Working with Persistent Volume
+ Deploying the Python Application to kubernetes

### Requirements

To seamlessly follow through, you will need the following:

+ docker

Docker is an open platform to build and ship distributed applications. To install docker on Debian/Ubuntu:

    sudo apt-get install docker.io
    
Verify that docker now runs:

    $ docker info
    Containers: 0
    Images: 289
    Storage Driver: aufs
     Root Dir: /var/lib/docker/aufs
     Dirs: 289
    Execution Driver: native-0.2
    Kernel Version: 3.16.0-4-amd64
    Operating System: Debian GNU/Linux 8 (jessie)
    WARNING: No memory limit support
    WARNING: No swap limit support

+ kubectl

kubectl is a command line interface for executing commands against a kubernetes cluster. Run the shell script below to install 
kubectl.

**install_kubectl.sh**
  curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release
  /release/stable.txt)/bin/linux/amd64/kubectl

+ Get the [source code](https://github.com/jnanjekye/k8s_python_sample_code/tree/master)

One of the requirements to deploy to kubernetes, is a containerised application. I will quickly start with a discussion on 
containerizing python Applications.

## Containerization at a glance

Containerization involves enclosing an application in a container with its own operating system. This is an option to full machine 
virtualization giving us the advantage of being able to run an application on any machine without worrying about dependencies.Roman Gaponov has a good [article](https://hackernoon.com/docker-tutorial-getting-started-with-python-redis-and-nginx-81a9d740d091) 
that you can reference. We will start with creating a container image for our python code.

## Creating a python container image

To create these images, we will use docker. Docker is popular software solution that allows us to deploy applications inside isolated 
Linux software containers. Docker is able to automatically build images using instructions from a dockerfile.

**Docker file**

This is a docker file for our python application.

    FROM python:3.6
    MAINTAINER XenonStack

    # Creating Application Source Code Directory
    RUN mkdir -p /k8s_python_sample_code/src

    # Setting Home Directory for containers
    WORKDIR /k8s_python_sample_code/src

    # Installing python dependencies
    COPY requirements.txt /k8s_python_sample_code/src
    RUN pip install --no-cache-dir -r requirements.txt

    # Copying src code to Container
    COPY . /k8s_python_sample_code/src/app

    # Application Environment variables
    ENV APP_ENV development

    # Exposing Ports
    EXPOSE 5035

    # Setting Persistent data
    VOLUME ["/app-data"]

    # Running Python Application
    CMD ["python", "app.py"]
 
 This is a dockerfile with instructions to run our sample python code. It use the python 3.5 development environment. 
 
 **Build a python docker image**
 
 We can now build the docker image from these instructions using this command.
 
    docker build -t k8s_python_sample_code .
  
 This command creates a docker image for our python application living in src/
 
 ## Publishing the container images
 
 We can publish our python container image to different private/public cloud repositories like dockerhub,  AWS ECR, Google Container Registry,
 etc. For purposes of this tutorial, we shall use dockerhub.
 
 Before publising the image, we need to tag it to a version.
 
     docker tag k8s_python_sample_code:latest k8s_python_sample_code:0.1
     
  Once this is done, push the image to the cloud repository.
  
  **Push the image to cloud repository**
  
  Using a docker registry other than dockerhub to store images requires you to add that container registry to the local docker 
  daemon and kubernetes Docker daemons. You can look up this information for the different cloud registries. We shall use dockerhub in
  this blog.
  
  Execute this docker command to push the image.
  
      docker push k8s_python_sample_code
      
  ## Persistent Storage
  
  Kubernetes supports many persistent storage like AWS EBC, CephFS, GlusterFS, Azure Disk, NFS, etc. I will cover kubernetes 
  persistence storage with cephfs.
  
  To use cephfs for persistent data to kubernetes containers, we will create two file:
  
  + persistent-volume.yml
  
      ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: app-disk1
      namespace: k8s_python_sample_code
    spec:
      capacity:
      storage: 50Gi
      accessModes:
      - ReadWriteMany
      cephfs:
      monitors:
        - "172.17.0.1:6789"
      user: admin
      secretRef:
        name: ceph-secret
      readOnly: false
      
  + persistent_volume_claim.yaml
  
      ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: appclaim1
      namespace: k8s_python_sample_code
    spec:
      accessModes:
      - ReadWriteMany
      resources:
      requests:
        storage: 10Gi
 
 We can now use kubetctl to add the persitent volume and claim to the kubernetes cluster.
 
    $ kubectl create -f persistent-volume.yml
    $ kubectl create -f persistent-volume-claim.yml
 
 We are nowready to deploy to kubernetes.
 
 ## Deploying the Application to Kubernetes
 
To manage our last mile of deploying the application to kubernetes, we will create two important file:

+ Service file

Create a file and name it k8s_python_sample_code.service.yml with the following content.

    apiVersion: v1
    kind: Service
    metadata:
      labels:
      k8s-app: k8s_python_sample_code
      name: k8s_python_sample_code
      namespace: k8s_python_sample_code
    spec:
      type: NodePort
      ports:
      - port: 5035
      selector:
      k8s-app: k8s_python_sample_code
      
  + Deployment file
  
  Create a file and name it k8s_python_sample_code.deployment.yml with the following content.
  
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: k8s_python_sample_code
      namespace: k8s_python_sample_code
    spec:
      replicas: 1
      template:
      metadata:
        labels:
        k8s-app: k8s_python_sample_code
      spec:
        containers:
        - name: k8s_python_sample_code
          image: k8s_python_sample_code:0.1
          imagePullPolicy: "IfNotPresent"
          ports:
          - containerPort: 5035
          volumeMounts:
            - mountPath: /app-data
              name: k8s_python_sample_code
         volumes: 
             - name: <name of application>
               persistentVolumeClaim:
                 claimName: appclaim1

We can now use kubectl to deploy the application to kubernetes:

  $ kubectl create -f k8s_python_sample_code.deployment.yml
  $ kubectl create -f k8s_python_sample_code.service.yml
  
Yaay, Your app was successfully deployed to kubernetes.

## Verification

You can verify whether your  appis running by  inspecting the running services.

    kubectl get services

May kubernetes set you free!!!

You can checkout my recent book on [python 2 and 3 Compatibility](https://www.amazon.com/Python-Compatibility-Six-Python-Future-Libraries/dp/1484229541)by Apress for my other content.

