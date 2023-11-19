# Creating a Local Event-Driven Workflow: Leveraging Kafka as an Event Source
 
The primary aim of this article is to simplify the setup process of a local Kafka cluster and a local Argo Events ecosystem. This will prove immensely beneficial when conducting local testing, sparing you the need to enact changes and endure the anticipation of a response from the build server.
## Kafka cluster
Among the most straightforward methods to rapidly establish a local Kafka cluster is by adhering to the guidance provided in this informative article at [How to set up Kafka on Kubernetes with Strimzi in 5 minutes | b-nova](https://b-nova.com/en/home/content/heres-how-you-can-setup-kafka-with-strimzi-on-kubernetes-in-only-five-minutes/)
In a nutshell, to get started, you will be required to execute the following set of commands:
```
kubectl create namespace kafka
kubectl create -f https://strimzi.io/install/latest?namespace=kafka -n kafka
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-persistent-single.yaml -n kafka
```
Next, you can confirm that everything is functioning as intended by running the following command (ensure the 'Ready' column displays 'true'):
```
kubectl get kafka -n kafka
```
## Argo Events ecosystem
After successfully launching Kafka, the next crucial step is to create all the Argo Event-specific objects. I will provide a comprehensive, step-by-step guide to ensure the simplest and most pristine setup possible.
In summary, we will begin by establishing a dedicated namespace for all our objects. Then, we'll proceed to generate multiple files for RBAC, Event Bus, Event Source, Sensor, and Workflow Template. Please note that I won't delve into the intricate details of how these objects interconnect since there is a separate article on that topic within this wiki (You can find instructions on how to set up local Argo Workflow UI [here](https://github.com/luboganchev/DevWeek2023/blob/main/configure-argo-workflows-gui.md)).
So, let's get started with the following steps:
1. Create a namespace by running the following command:
```
kubectl create ns argo-kafka-local
```
2. Create a new physical folder called argo-kafka-local which will contain our local .yaml files.
3. Before we dive into crafting the different Argo objects, let's set up RBAC (Role-Based Access Control). This entails creating a Service Account, Role, and RoleBinding. To get started, create a file named **argo-kafka-local-rbac.yaml** and insert the following content:

```yaml
apiVersion: v1kind: ServiceAccountmetadata:  name: argo-kafka-local-sa  namespace: argo-kafka-local---apiVersion: rbac.authorization.k8s.io/v1kind: Rolemetadata:  name: argo-kafka-local-role  namespace: argo-kafka-localrules:  - apiGroups:      - ''    resources:      - pods    verbs:      - get      - watch      - patch  - apiGroups:      - ''    resources:      - pods/log    verbs:      - get      - watch  - apiGroups:      - argoproj.io    resources:      - workflows      - workflowtemplates      - workfloweventbindings      - clusterworkflowtemplates      - cronworkflows      - sensors      - workflowtaskresults      - eventsources    verbs: - '*' --- apiVersion: rbac.authorization.k8s.io/v1 kind: RoleBinding metadata: name: argo-kafka-local-role-binding namespace: argo-kafka-local roleRef: apiGroup: rbac.authorization.k8s.io kind: Role name: argo-kafka-local-role subjects: - kind: ServiceAccount name: argo-kafka-local-sa namespace: argo-kafka-local
```
To apply the RBAC file in your Kubernetes environment, execute the following command (*make sure you've navigated to the already created folder mentioned above argo-kafka-local*):
``` 
kubectl apply -f argo-kafka-local-rbac.yaml
```
4. Now that our RBAC setup is complete, we can proceed to create each of the Argo Event-specific objects one by one. Let's begin with the Event Bus. Please create a new file named **event-bus-default.yaml** and include the following content:
```
# This sets up the eventbus for argo events. You typically have one per environment per namespace.---apiVersion: argoproj.io/v1alpha1kind: EventBusmetadata:  name: event-bus-default  namespace: argo-kafka-localspec:  jetstream:    version: 2.8.1    persistence: accessMode: ReadWriteOnce volumeSize: 1G
```
Apply our newly created event bus to the k8s cluster by running the following command: 
```
kubectl apply -f event-bus-default.yaml
```
5. Next, let's create an Event Source responsible for subscribing to messages on a specific Kafka topic. To achieve this, create a file named **event-source-kafka.yaml** and input the following content:
```
apiVersion: argoproj.io/v1alpha1kind: EventSourcemetadata:  name: event-source-kafka  namespace: argo-kafka-localspec:  replicas: 1  eventBusName: event-bus-default  template:    serviceAccountName: argo-kafka-local-sa    container:      resources:        requests:          memory: '64Mi'          cpu: '100m'        limits:          memory: '128Mi'          cpu: '400m'  kafka:    kafka-broker:      url: my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092 topic: my-topic jsonBody: true partition: '0' connectionBackoff: duration: 10s steps: 5 factor: 2 jitter: 0.2
```
Apply the event source to the k8s cluster by running the following command 
``` 
kubectl apply -f event-source-kafka.yaml
``` 
6. After setting up the Event Source, it's essential to create a Sensor. To accomplish this, create a file named **sensor-new-kafka-message.yaml** and copy/paste the following content:
```
apiVersion: argoproj.io/v1alpha1kind: Sensormetadata:  name: sensor-new-kafka-message  namespace: argo-kafka-localspec:  eventBusName: event-bus-default  template:    serviceAccountName: argo-kafka-local-sa  dependencies:    - name: new-kakfa-message      eventSourceName: event-source-kafka      eventName: kafka-broker  triggers:    - template:        name: simple-container-template        argoWorkflow:          operation: submit          source:            resource:              apiVersion: argoproj.io/v1alpha1              kind: Workflow              metadata:                generateName: new-kafka-message-received-                namespace: argo-kafka-local              spec:                entrypoint: whalesay-template serviceAccountName: argo-kafka-local-sa arguments: parameters: - name: message value: 'hello world from sensor' workflowTemplateRef: name: workflow-template-simple-container parameters: - src: dependencyName: new-kakfa-message dataKey: body dest: spec.arguments.parameters.0.value
```
Afterward, execute this command in order to create the object in the cluster:
``` 
kubectl apply -f sensor-new-kafka-message.yaml
```
7. To complete the puzzle, the final step involves creating the Workflow Template, which will be utilized by the Sensor to generate workflows for each received Kafka message. Create a file named **workflow-template-simple-container.yaml** and insert the following code snippet:
```
apiVersion: argoproj.io/v1alpha1kind: WorkflowTemplatemetadata:  name: workflow-template-simple-container  namespace: argo-kafka-localspec:  entrypoint: whalesay-template  serviceAccountName: argo-kafka-local-sa  arguments:    parameters:      - name: message        value: hello world  templates:    - name: whalesay-template      inputs: parameters: - name: message container: image: docker/whalesay command: [cowsay] args: ['{{inputs.parameters.message}}']
```
Apply and the last argo events object which will be needed:
```
kubectl apply -f workflow-template-simple-container.yaml
``` 
## Test and verify
To comprehensively test and verify that everything is functioning as expected, follow these steps:
1. Open your Argo Workflow UI (you can find instructions on how to set up the [**Argo Workflow UI**](https://github.com/luboganchev/DevWeek2023/blob/main/configure-argo-workflows-gui.md)). 
2. Launch two new terminal tabs.
3. In the first tab, create a **Kafka producer**.
4. In the second tab, set up a **Kafka consumer**.
5. Execute the following commands, and try entering a **JSON** message in the producer tab. You should observe the same exact message appearing in the consumer tab.
**Kafka producer**:
```
kubectl -n kafka run kafka-producer -ti --image=quay.io/strimzi/kafka:0.23.0-kafka-2.8.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
```
**Kafka consumer**:
``` 
kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.23.0-kafka-2.8.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```
After the Kafka producer is operational, paste a test **JSON** message (since we've configured the event source to exclusively work with **JSON**). To inspect the logs within the Argo Workflow UI, follow these steps:
1. Start by checking the Event Source logs. You should expect to find the log entry: **"Succeeded to publish an event."**
2. Next, move on to the Sensor and verify if there is a log entry stating: **"Successfully processed trigger 'simple-container-template'."**
3. Lastly, go to the Workflows section and confirm that a new workflow has been triggered with the name: **"new-kafka-message-received-{generatedId}."**

### **Congratulations, you have successfully set up your local Kafka cluster and triggered an Argo workflow from a Kafka message!**
