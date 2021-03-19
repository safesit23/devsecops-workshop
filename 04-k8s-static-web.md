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
