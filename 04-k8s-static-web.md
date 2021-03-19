# Static Website on Kubernetes Workshop

## 01-Prepare Repository

* Create `opsta-web` repository on your own GitHub
* Add SSH Key to your GitHub account on <https://github.com/settings/keys>

```bash
ssh-keygen
```

* Initial Opsta HTML Static Website

```bash
wget https://github.com/opsta/opsta-www.github.io/archive/gh-pages.zip
unzip gh-pages.zip
mkdir opsta-web
find opsta-www.github.io-gh-pages -type f \
  -exec sed -i 's!https://www.opsta.co.th/!https://dev.opsta.co.th/student150/opsta/!g' {} \;
mv opsta-www.github.io-gh-pages opsta-web/src
cd opsta-web
ls -l
git init
git add .
git commit -m "First Initial"
git remote add origin git@github.com:winggundamth/opsta-web.git
git branch -M main
git push -u origin main
```

## Create Dockerfile and docker-compose.yml

## Install Docker Compose

```bash
sudo CRYPTOGRAPHY_DONT_BUILD_RUST=1 pip3 install docker-compose
sudo curl -L https://raw.githubusercontent.com/docker/compose/1.28.5/contrib/completion/bash/docker-compose \
  -o /etc/bash_completion.d/docker-compose
docker-compose version
```

* Create Dockerfile

```Dockerfile
FROM nginx:1.19.8-alpine
COPY src /usr/share/nginx/html
```

* Create docker-compose.yml

```yaml
services:
  web:
    build: .
    image: ghcr.io/[GITHUB_USER]/opsta-web:dev
    ports:
      - "8080:80"
```

* Run `docker-compose up`
* Web Preview to test

## Push Docker Image to GitHub Docker Registry

* Enable `Feature preview` > `Improved container support`
* Generate new personal token from <https://github.com/settings/tokens>
  * repo
  * write:packages
* Copy token

```bash
export TOKEN=CHANGEME
export GITHUB_USER=CHANGEME
echo $TOKEN | docker login ghcr.io --password-stdin --username $GITHUB_USER
docker push ghcr.io/$GITHUB_USER/opsta-web:dev
```

## Prepare Kubernetes Environment

```bash
gcloud container clusters get-credentials k8s --project zcloud-cicd --zone asia-southeast1-a
kubectl create namespace student[X]-opsta-dev
kubectl config set-context $(kubectl config current-context) --namespace=student[X]-opsta-dev
```

## Create Secret to pull Docker Image from GitHub Docker Private Registry

```bash
# See the Docker credentials file
cat ~/.docker/config.json
# Show secret
kubectl get secret
# Create Docker credentials Kubernetes Secret
kubectl create secret generic registry-github \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
# See newly created secret
kubectl get secret
kubectl describe secret registry-github
```

## Create Kubernetes Manifest File

* `mkdir k8s/` to make a directory to store manifest file
* Create `opsta-deployment.yaml` file inside `k8s/` directory with below content

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

* Create `opsta-service.yaml` file inside `k8s/` directory with below content

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

* Create `opsta-ingress.yaml` file inside `k8s/` directory with below content

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

```bash
# Create deployment resource
kubectl apply -f k8s/

# Check status of each resource
kubectl get deployment
kubectl get service
kubectl get ingress
```

* Try to access <https://dev.opsta.net/student[X]/opsta> to check the deployment
* Commit and push your code

## Assignment

* Delete your own namespace and start over
* Choose any HTML static from <https://www.free-css.com/free-css-templates>
* Create Dockerfile, docker-compose.yml and Kubernetes Manifest File with
  * repository name: `static-web`
  * docker image name: `ghcr.io/[GITHUB_USER]/static-web:dev`
  * path name: `static`
