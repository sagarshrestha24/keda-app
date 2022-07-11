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


To login to the RabbitMQ we will do port forward 

``` kubectl port-forward service/rabbitmq-keda-cluster 15672 ```

Now try to access the Management console with below address and login with ‘guest/guest’.

http://localhost:15672

![image](https://user-images.githubusercontent.com/76894861/178247021-cbe3dc98-521b-4c22-a047-fbc9ea9468e7.png)

Now create a queue ‘kedaQ’ from the console. You can give any name of the queue.

![image](https://user-images.githubusercontent.com/76894861/178247213-07fc8cdd-fb01-4bc7-b1b0-a7ea51fec3e7.png)


## Deploy SpringBoot Sender Application

```
git clone https://github.com/koushikmgithub/springboot-rmq-sender.git

mvn clean install

docker build -t username/springboot-rmq-sender:1.0 .

docker push username/springboot-rmq-sender:1.0

kubectl apply -f kubernetes/app-deployment.yaml

```

Now verify the application deployment by using below command.

``` kubectl get pods ```

![image](https://user-images.githubusercontent.com/76894861/178249681-de852c4f-e47f-44cb-a78c-34df2862b371.png)


Once the pod is up and running, let’s access the application’s endpoint and publish some messages to queue.


``` kubectl port-forward svc/springboot-rmq-sender-service 9095 ```


After port forwarding, you can access the application locally. For accessing the endpoint, I have used postman.


Sending some sample messages to RMQ. You can verify the message in ‘kedaQ’ from RabbitMQ console.

![image](https://user-images.githubusercontent.com/76894861/178251973-b485910b-2218-470a-ac19-38f827e28c22.png)


## Deploy SpringBoot Receiver Application

Now let’s deploy the receiver application. The receiver application is listening to the ‘kedaQ’ and store all the messages to H2 DB. It’s also exposed an endpoint by which you can fetch all the messages from DB itself.

```
git clone https://github.com/koushikmgithub/springboot-rmq-receiver.git

mvn clean install

docker build -t username/springboot-rmq-receiver:1.0 .

docker push username/springboot-rmq-receiver:1.0

kubectl apply -f kubernetes/app-deployment.yaml

```

Now verify the application deployment by using the below command.

``` kubectl get pods ```

![image](https://user-images.githubusercontent.com/76894861/178252442-312b9756-9ec2-4811-8530-3585c54b6743.png)


Do the port forward for accessing the application’s endpoint locally.


``` kubectl port-forward svc/springboot-rmq-receiver-service 9096 ```


Use the postman and access the /receiveMessage endpoint


![image](https://user-images.githubusercontent.com/76894861/178252979-2bac1f32-9517-43c3-b3d5-5e6c9a3cf1c0.png)

For Load Testing we used seige tools for load testing 

To install seige use this command 

``` 
sudo apt update -y

sudo apt install siege -y

```

Once siege is installed successfully, let’s open a terminal and execute the below command.


``` siege -d1 -c10 "http://localhost:9095/sender/sendMessage?msg=Sending Hello Message to RMQ" ```

Here we have create a ScaledObject which is the custom resource definition, where you can define the source of metrics, as well as autoscaling criteria.


``` 
‘app-deployment-keda.yaml’

apiVersion: v1
kind: Secret
metadata:
  name: keda-rabbitmq-secret
data:
  host: YW1xcDovL2d1ZXN0Omd1ZXN0QHJhYmJpdG1xLWtlZGEtY2x1c3Rlci5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsOjU2NzI=  # base64 encoded value of format amqp://guest:password@localhost:5672/vhost
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-rabbitmq-conn
  namespace: default
spec:
  secretTargetRef:
    - parameter: host
      name: keda-rabbitmq-secret
      key: host
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: springboot-rmq-receiver-deployment
  pollingInterval: 20 # Optional. Default: 30 seconds
  cooldownPeriod: 120 # Optional. Default: 300 seconds
  maxReplicaCount: 20 # Optional. Default: 100  
  triggers:
  - type: rabbitmq
    metadata:
      protocol: amqp
      queueName: kedaQ #queue name
      mode: QueueLength
      value: "5"      
    authenticationRef:
      name: keda-trigger-auth-rabbitmq-conn
      
```

Note: host is base64 encode of : amqp://guest:guest@rabbitmq-keda-cluster.default.svc.cluster.local:5672

Lets Apply the abone configuration file

``` kubectl apply -f app-deployment-keda.yaml ```


Once the file is applied successfully, the KEDA will start the action. Incase there is no message in the queue, the receiver application will be terminated automatically.

Let’s send some high load again to the queue and see the actions.


``` siege -d1 -c10 ‘http://localhost:9095/sender/sendMessage?msg=Sending Hello Message to RMQ’ ```

![image](https://user-images.githubusercontent.com/76894861/178254611-eee51c12-0314-4d6e-ac10-c7f65228760b.png)


This time we have sent 89 transactions/ messages to the kedaQ and let’s see how KEDA is working in the backend. 

![image](https://user-images.githubusercontent.com/76894861/178255416-d63c7b7a-4145-4799-a21b-149b3b5d2f71.png)


After sometimes, there is no message/events in ‘kedaQ’ since it is already consumed by receiver apps , so the KEDA deactivates Kubernetes deployments (here Receiver App Deployment ) to scale to zero. If you check the pods for receiver apps , you can find it is in terminating state.

And finally there is no receiver pods.


![image](https://user-images.githubusercontent.com/76894861/178256094-8f81b254-5bef-4079-bf1a-27ea062f55ac.png)






