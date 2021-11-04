# Deploying-a-Memcached-service-on-kubernetes
#In Cloud Shell, create a new Kubernetes Engine cluster of three nodes:
gcloud container clusters create demo-cluster --num-nodes 3 --zone us-central1-f
#Configure Helm
#Add Helm's stable chart repository:
helm repo add stable https://charts.helm.sh/stable
#Update
helm repo update
#Install a new Memcached Helm chart release with three replicas, one for each node
helm install mycache stable/memcached --set replicaCount=3
#wait for few seconds
kubectl get pods
#
Discovering Memcached service endpoints
Verify that the deployed service is headless
#

kubectl get service mycache-memcached -o jsonpath="{.spec.clusterIP}" ; echo

It is the client's responsibility to discover the Memcached service endpoints. To do that:

Retrieve the endpoints' IP addresses:

kubectl get endpoints mycache-memcached

Implement the service discovery logic

kubectl run -it --rm python --image=python:3.6-alpine --restart=Never sh
pip install pymemcache
python
import socket
from pymemcache.client.hash import HashClient
_, _, ips = socket.gethostbyname_ex('mycache-memcached.default.svc.cluster.local')
servers = [(ip, 11211) for ip in ips]
client = HashClient(servers, use_pooling=True)
client.set('mykey', 'hello')
client.get('mykey')

Enabling connection pooling

Deploy Mcrouter
To deploy Mcrouter, run the following commands in Cloud Shell.

Uninstall the previously installed mycache Helm chart release:

helm delete mycache

helm install mycache stable/mcrouter --set memcached.replicaCount=3
Reducing latency

Configure Application Pods to Expose the Kubernetes Node Name as an Environment Variable

cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-application-py
spec:
  replicas: 5
  selector:
    matchLabels:
      app: sample-application-py
  template:
    metadata:
      labels:
        app: sample-application-py
    spec:
      containers:
        - name: python
          image: python:3.6-alpine
          command: [ "sh", "-c"]
          args:
          - while true; do sleep 10; done;
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
EOF

kubectl get pods






POD=$(kubectl get pods -l app=sample-application-py -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it $POD -- sh -c 'echo $NODE_NAME'
