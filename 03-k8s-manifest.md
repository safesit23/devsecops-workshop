# Kubernetes Manifest Files Workshop

Original from [Opstra](https://github.com/opsta/devsecops-workshop/blob/master/docs/06-k8s-manifest.md)

## Prepare Kubernetes Environment

```bash
gcloud container clusters get-credentials k8s --project zcloud-cicd --zone asia-southeast1-a
kubectl create namespace student[X]-opsta-dev
kubectl config set-context $(kubectl config current-context) --namespace=student[X]-opsta-dev
```

## Deploy Pod with Manifest File

* `mkdir ~/k8s` to create folder for Kubernetes Manifest File
* Create `01-pod.yaml` file with below content

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: student[X]-opsta-dev
spec:
  containers:
  - name: busybox
    image: busybox
    command:
    - sleep
    - "3600"
```

* Create pod from manifest file

```bash
cd ~/k8s
# Create resources as configured in manifest file
kubectl apply -f 01-pod.yaml
kubectl get pod

# Try to get inside pod
kubectl exec -it busybox -- sh
ping www.google.com
exit
```

## How to check syntax for manifest file

```bash
# Show all api resources in Kubernetes Cluster
kubectl api-resources
# Show manifest syntax for Kind = pod
kubectl explain pod
# Show all manifest syntax for Kind = pod
kubectl explain pod --recursive
# Show manifest spec syntax for Kind = pod
kubectl explain pod.spec
# Show manifest syntax for Kind = deployment
kubectl explain deployment
```

## Deployment and Service Manifest File

* Create `02-apache-deployment.yaml` file with below content

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache
  namespace: student[X]-opsta-dev
  labels:         //ผูกกันระหว่าง services กับ deployment
    app: apache
spec:
  replicas: 1
  selector:      //ผูกกันระหว่าง pod กับ deployment
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - image: httpd:2.4.43-alpine
        name: apache
```
### อธิบาย 
ส่วนนี้บอกถึง Pod
````
spec:
      containers:
      - image: httpd:2.4.43-alpine
        name: apache
````

* Create `02-apache-service.yaml` file with below content

```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache
  namespace: student[X]-opsta-dev
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: apache 
```
### อธิบาย 
- ใน Services จะมี selector ไปที่ app: apache ซึ่งจะไป map กับ deployment ที่ label ว่า app: apache

```bash
kubectl create -f 02-apache-deployment.yaml -f 02-apache-service.yaml --record
kubectl get deployments
kubectl get services
kubectl proxy --port=8080
# Click preview port 8080
# access via /api/v1/namespaces/student[xxx]-opsta-dev/services/apache:80/proxy/
```
### create vs apply
- apply เพื่อ check change ของ pod เมื่อมีการเปลี่ยนแปลง

Try to change replicas to 1 in `02-apache-deployment.yaml`
แล้ว run คำสั่ง
````
kubectl apply -f 02-apache-deployment.yaml -f 02-apache-service.yaml
````


## Clean every deployment and service

```bash
# Delete from files in this folders
kubectl delete -f .
# Show all pod,deployments,services
kubectl get po,deploy,svc
```

Next: [Static Website on Kubernetes Workshop](docs/07-k8s-static-web.md)
