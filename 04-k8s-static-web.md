# Static Website on Kubernetes Workshop

## 01-Preparation Repository
Create Repository and push repository

1. Create `opsta-web` repository on your own GitHub
2. gen ssh-key
```bash
ssh-keygen
cat ~/.ssh/id_rsa.pub
```
3. Add SSH Key to your GitHub account on <https://github.com/settings/keys> ดูวิธีทำได้ที่ [Ref](https://github.com/opsta/devsecops-workshop/blob/master/docs/02-git.md#push-repository-to-github)
4. Initial Git CLI
````
git config --global user.email "62130500[xxx]@mail.kmutt.ac.th"
git config --global user.name "[MYFIRSTNAME MYLASTNAME]"
````
5. Initial Opsta HTML Static Website
````
wget https://github.com/opsta/opsta-www.github.io/archive/gh-pages.zip
unzip gh-pages.zip
mkdir opsta-web
mv opsta-www.github.io-gh-pages opsta-web/src
cd opsta-web
ls -l
git init
git add .
git commit -m "First Initial"
git remote add origin git@github.com:[GITHUB_USER]/opsta-web.git
git branch -M main
git push -u origin main
````
## 02-Docker
### Create Dockerfile and docker-compose.yml (Build)

1. Create `Dockerfile`

```Dockerfile
FROM nginx:1.19.8-alpine
COPY src /usr/share/nginx/html
```

2. Create `docker-compose.yml`

```yaml
services:
  web:
    build: .
    image: ghcr.io/[GITHUB_USER]/opsta-web:dev
    ports:
      - "8080:80"
```

3. Run `docker-compose up` !!Dont't forget to update [Docker-Compose](https://github.com/opsta/devsecops-workshop/blob/master/docs/04-docker-compose.md#install-docker-compose)
* Web Preview on port 8080 to test

### Push Docker Image to GitHub Docker Registry (Ship)

1. ใน https://github.com/ คลิกเมนูที่รูปโปรไฟล์แล้วเลือก `Feature preview` > `Improved container support` > แล้วกดเพื่อ Enable
2. ไปที่ Setting > Developer Setting > Personal access tokens หรือกดที่ <https://github.com/settings/tokens> แล้วเลือก Generate new personal token โดยตั้งชื่อ Notes เพื่อบอกว่าใช้ทำอะไรเช่น INT209 โดยเลือก
  * repo
  * write:packages
3. Copy token ที่ได้แล้วนำไปใช้ในข้อ 4 !! ถ้าทำหายจะต้อง delete แล้วทำสร้างใหม่เท่านั้น
4. กลับไปที่ Cloudshell แล้วพิมพ์คำสั่งดังนี้
```bash
export TOKEN=[TOKEN]
export GITHUB_USER=[GITHUB_USER]
echo $TOKEN | docker login ghcr.io --password-stdin --username $GITHUB_USER
docker push ghcr.io/$GITHUB_USER/opsta-web:dev
```
หมายเหตุ 2 คำสั่งล่างไม่ต้องเปลี่ยน command

## 03-Kubernetes
### Preparing
```bash
gcloud container clusters get-credentials k8s --project zcloud-cicd --zone asia-southeast1-a
kubectl create namespace student[X]-opsta-dev
kubectl config set-context $(kubectl config current-context) --namespace=student[X]-opsta-dev
```

### Create Secret to pull Docker Image from GitHub Docker Private Registry
```bash
# See the Docker credentials file
cat ~/.docker/config.json
# Show secret
kubectl get secret
# Create Docker credentials Kubernetes Secret
kubectl create secret generic registry-github \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
```
<b>อธิบาย</b> ลักษณะการเก็บของ secret เป็น key และ value
```bash
# See newly created secret
kubectl get secret
kubectl describe secret registry-github
```
โดยจะมี Data ใน Secret
```
Name:         registry-github
Namespace:    student168-opsta-dev
Labels:       <none>
Annotations:  <none>
Type:  kubernetes.io/dockerconfigjson
Data
====
.dockerconfigjson:  307 bytes
```

##  04 - Create Kubernetes Manifest File (Run)
1. สร้าง Folder k8s ภายใต้ directory เพื่อเก็บ manifest file โดยใช้คำสั่ง `mkdir k8s`
2. สร้างไฟล์ 3 ไฟล์
* Create `opsta-deployment.yaml` file inside `k8s` directory with below content

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opsta-dev-web
  namespace: student[X]-opsta-dev
  labels:
    app: opsta-dev-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: opsta-dev-web
  template:
    metadata:
      labels:
        app: opsta-dev-web
    spec:
      containers:
      - name: opsta-dev-web
        image: ghcr.io/[GITHUB_USER]/opsta-web:dev
        imagePullPolicy: Always
        ports:
        - containerPort: 80
          name: web-port
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: 80
            scheme: HTTP
        readinessProbe:
          httpGet:
            path: /
            port: 80
            scheme: HTTP
      imagePullSecrets:
      - name: registry-github
```
<b>อธิบาย</b>
> imagePullSecrets นั้นจะดึงที่เชื่อมต่อ registry-github
> livenessProbe เป็น Healthcheck

* Create `opsta-service.yaml` file inside `k8s` directory with below content

```yaml
apiVersion: v1
kind: Service
metadata:
  name: opsta-dev-web
  namespace: student[X]-opsta-dev
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: opsta-dev-web
```

* Create `opsta-ingress.yaml` file inside `k8s` directory with below content

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  name: opsta-dev-web
  namespace: student[X]-opsta-dev
spec:
  rules:
  - host: dev.opsta.net
    http:
      paths:
      - path: /student[X]/opsta(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: opsta-dev-web
            port:
              number: 80
```
<b>อธิบาย</b>
* สร้าง Ingress แล้วเข้าผ่าน domain dev.opsta.net ที่เป็น load balancer ที่ใช้ร่วมกัน โดยต้องผ่าน path /student[X]/opsta ซึ่งจะเชื่อมต่อกับ service ที่ชื่อว่า opsta-dev-web
* ซึ่ง path (/|$)(.*) จะเชื่อมกับ nginx.ingress.kubernetes.io/rewrite-target: /$2

3. Create deployment resource
```bash
kubectl apply -f k8s/
```

4. Check status of each resource
```bash
kubectl get deployment,service,ingress
```

* Try to access <https://dev.opsta.net/student[X]/opsta> to check the deployment
* Commit and push your code
