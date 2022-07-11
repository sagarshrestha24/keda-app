# KEDA: Kubernetes Based Event-driven Autoscaling with SpringBoot and RabbitMQ

KEDA is a Kubernetes-based Event Driven Autoscaler. With KEDA, you can drive the scaling of any container in Kubernetes based on the number of events needing to be processed.

KEDA is a single-purpose and lightweight component that can be added into any Kubernetes cluster. KEDA works alongside standard Kubernetes components like the Horizontal Pod Autoscaler and can extend functionality without overwriting or duplication. With KEDA you can explicitly map the apps you want to use event-driven scale, with other apps continuing to function. This makes KEDA a flexible and safe option to run alongside any number of any other Kubernetes applications or frameworks.

## High-level Architecture
At a high level, KEDA provides below components in order to control the auto scaling process:
1. Operator / Agent : Activates and deactivates Kubernetes Deployments to scale to and from zero on no events.
2. Scalers : Connects to an external event source and feed custom metrics for a specific event source. A current list of scalers is available on the KEDA home page. ( https://keda.sh/#scalers )
3. Metrics: Acts as a Kubernetes metrics server that exposes rich event data like queue length or stream lag to the Horizontal Pod Autoscaler to drive scale out.

![This is an image](https://miro.medium.com/max/1400/1*TB2xTS5qIq7ps-qy5NiSvA.png)


In This Project,There are two SpringBoot applications/services. Both applications are communicating to RabbitMQ via queue. Sender application is publishing some message to queue and Receiver Application is listening to that queue and consumes all the messages present in that queue.

## Install KEDA on kubernetes cluster

KEDA can be installed and configured in a various way to your cluster. Please follow the official documents for KEDA.

https://keda.sh/docs/2.0/deploy/

I have used YAML approach for installing KEDA. If would like to follow the same approach then please execute below command.

``` kubectl apply -f https://github.com/kedacore/keda/releases/download/v2.0.0/keda-2.0.0.yaml ```

Once the above command executed successfully, you can verify the installation.

``` kubectl get all -n keda ```

Now, you can see all resources are up and running well. Look at the pods keda-operator-xxx and keda-metrics-apiserver-xxx those are main components for KEDA to be working properly.

![image](https://user-images.githubusercontent.com/76894861/178245252-4d0b756f-562c-4782-a562-a183d588245e.png)


## RabbitMQ installation on kubernetes cluster

There are various approaches by which you can install RabbitMQ in Kubernetes cluster. Here I am using Kubernetes Operator approach. For that I have used RabbitMQ cluster operator. You can simply execute the below command.

``` kubectl apply -f "https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml" ```

You can check the successful deployment of RabbitMQ cluster operator by below command.


``` kubectl get all -n rabbitmq-system ```

![image](https://user-images.githubusercontent.com/76894861/178245788-a6221f3d-6953-4db2-b052-da1cc6cbcb0c.png)

Once the Operator has been installed successfully, we are good to go with creating RabbitMQ cluster in our cluster. For that we will create the below YAML file ‘rabbitmq-keda-cluster.yaml’. 

``` 

apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-keda-cluster
spec:
  replicas: 1
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1
      memory: 2Gi
  rabbitmq:
          additionalConfig: |
                  log.console.level = info
                  channel_max = 1500
                  default_user= guest 
                  default_pass = guest
                  default_user_tags.administrator = true
  service:
    type: ClusterIP
    
 ```

Please execute the below command. You can find the actual file in Sender Application source code


``` kubectl apply -f rabbitmq-keda-cluster.yaml ```

Let’s verify the deployment of rabbitMq cluster. 


``` kubectl get all -l app.kubernetes.io/part-of=rabbitmq ```





