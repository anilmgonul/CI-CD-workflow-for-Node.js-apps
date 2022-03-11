# How to setup CI/CD workflow for Node.js apps with Jenkins and Kubernetes

### Introduction

DevOps filozofisini sekillendiren pratiklerden ikisi CI/CD, Continuous Integration and Continuous Delivery yani surekli entegrasyon ve surekli dagitimdir. Basitce, bu olgu icinde uygulamali urun gelisimi ve entegrasyonu barindirir, ana repository'de surekli commit edilerek fonksiyonlarin calisip calismadigi test edilip, urunu deploy etmeye hazir hale getirir.

Bu is akisi ve calisma dongusu otomasyon mugendisligini kullanir. Bu yazi dizininde ise CI/CD calisma methodunu aciklayip, bir Node.js uygulamasini Kubernetes'de host edip inceleyecegiz. Otomasyon kismi icin Jenkins kullanacagiz.

Bu dizin icin Kubernetes cluster'i kurulmus ve Helm/Tiller server yuklenmis olmalidir. Bunun yani sira, Git ile ilgili bilgi sahibi olmak ve repository'ye sahip olmak da gerekmektedir.


### Workflow and Architecture

Asagida gosterilen is ilerleyisini takip edecegiz:

![alt](workflow.png)


Farkedilecegi uzere Kubernetes namespacase'leri deployment environment yani yukleme sahasi olarak kullanacagiz.

Bunun icin namespacase'leri olusturmamiz gerekiyor:

```
$ kubectl create namespace myapp-integration

    namespace/myapp-integration created

```

```
$ kubectl create namespace myapp-production

    namespace/myapp-production created

```

Namespacase'lerin olusturulup olusturulmadigini kontol etmek icin:


```
$ kubectl get namespaces

NAME                   STATUS   AGE
default                Active   11d
kube-public            Active   11d
kube-system            Active   11d
kubernetes-dashboard   Active   11d
myapp-integration      Active   2m7s
myapp-production       Active   106s

```


### Setting Up the Workflow

Erisim saglayacagimiz uygulama iki parcadan olusmakta:

* Nginx reverse proxy
* NodeJS app

Uygulama kodu Git repository'sinde host edilirken, asagidaki gibi yapiya sahip:

```
/
   -src/
     index.js
   -tests/
     integration-tests.sh
     production-tests.sh
   -deploy/
     nginx-reverseproxy.yaml
     nodejs.yaml
   Jenkinsfile
```

Dosyalari yakindan inceleyelim. Buna gore, **src** directory Node.js uygulamasini barindirir ve port 8080'de http server'i olusturur, ve ziyaret ettigi path'lere gore mesaj bildirimi saglar:

* “This is homepage” when visiting “/”
* “Welcome to dir1, how can I help you ?” when visiting “/dir1”
* “The information about person with id 1 is X” when visiting “/dir2/person/1”

```
// index.js

var http = require('http');
var url = require('url');
var server = http.createServer(function(req, res) {
 var page = url.parse(req.url).pathname;
 console.log(page);
 res.writeHead(200, {"Content-Type": "text/plain"});

 if (page == '/') {
   res.write('This is homepage');
 }
 else if (page == '/dir1') {
  res.write('Welcome to dir1, how can I help you ?');
 }
 else if (page == '/dir2/person/1') {
  res.write('The information about person with id 1 is X');
 }
 res.end();
});
server.listen(8080);
```

**Test** directory ise uygulamanin calistigindan emin olmak adina test edebilmesi icin iki kod dizininden olurusur. Bu yazi dizininde, konunun basitlestirilmesi adina ayni test integration ve production environment'de kullanilacak.

Integration tests.sh:

```
#!/bin/bash
# integration-tests.sh

echo "Starting integration tests..."
echo "Testing root path..."
res1=$(curl -s http://$1/)
if [ "$res1" != "This is homepage" ]; then
 echo "Path / test failed. Aborting..."
 exit 1
fi

echo "Testing path /dir1 ..."
res2=$(curl -s http://$1/dir1)
if [ "$res2" != "Welcome to dir1, how can I help you ?" ]; then
 echo "Path /dir1 test failed. Aborting..."
 exit 1
fi

echo "Testing root path /dir2/person/1 ..."
res3=$(curl -s http://$1/dir2/person/1)
if [ "$res3" != "The information about person with id 1 is X" ]; then
 echo "Path /dir1 test failed. Aborting..."
 exit 1
fi

echo "Integration tests succeeded."
```

Production tests.sh:

```
#!/bin/bash
# production-tests.sh

echo "Starting production tests..."
echo "Testing root path..."
res1=$(curl -s http://$1/)
if [ "$res1" != "This is homepage" ]; then
 echo "Path / test failed. Aborting..."
 exit 1
fi

echo "Testing path /dir1 ..."
res2=$(curl -s http://$1/dir1)
if [ "$res2" != "Welcome to dir1, how can I help you ?" ]; then
 echo "Path /dir1 test failed. Aborting..."
 exit 1
fi

echo "Testing root path /dir2/person/1 ..."
res3=$(curl -s http://$1/dir2/person/1)
if [ "$res3" != "The information about person with id 1 is X" ]; then
 echo "Path /dir1 test failed. Aborting..."
 exit 1
fi

echo "Production tests succeeded."
```

**Deploy** directory ise gerekli olan butun yaml dosyalarini barindirir uygulamamizi Kubernetes'de deploy edebilmek icin. Asagida gosterilen yaml dosyasi Nginx reverse proxy'yi deploy edebilmek icin Kubernetes kaynaklarini barindirir.

```
#nginx-reverseproxy.yaml

apiVersion: v1
kind: Service
metadata:
  name: nginx-reverseproxy-service
spec:
 selector:
    app:  nginx-reverseproxy
 type: LoadBalancer  #LB to expose the service and get an external IP address
 ports:
  - name: http
    port: 80
    protocol: TCP
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app:  nginx-reverseproxy
  name: nginx-reverseproxy-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-reverseproxy
    spec:
      containers:
      - image: nginx:1.13
        name: kubecont-nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d     
      volumes:
        - name: config-volume
          configMap:
            name: nginx-reverseproxy-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-reverseproxy-config
data:
  default.conf: |-
    server {
     server_name yourhostname.com;
     listen 80;
     #deny access to .htaccess files, if Apache's document root
     #concurs with nginx's one
     #
     location ~ /\.ht {
         deny  all;
     }
location / {
         proxy_pass http://nodejs-service:8080; #this is the service described in nodejs.yaml
     }
    }
```


Son olarak, asagidaki yaml dosyasi Kubernetes'de Node.js'e erisim saglar.

```
# nodejs.yaml

apiVersion: v1
kind: Service
metadata:
  name: nodejs-service
spec:
 selector:
    app:  nodejs
 ports:
  - name: http
    port: 8080
    protocol: TCP
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app:  nodejs
  name: nodejs-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nodejs
    spec:
      containers:
      - image: node:9.11
        name: kubecont-nodejs
        command: ["node", "/usr/src/app/index.js"]
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: app-volume
          mountPath: /usr/src/app     
      volumes:
        - name: app-volume
          configMap:
            name: nodejs-app
```                

Bu kurulumu sonlandirirken, Jenkins icerisinde CI/CD isleyis seklini betimleyen Jenkinsfile'imiz var. Bu uc asamadan olusur:

* Hazirlik asamasi: kubectl'in yuklenmis olmasi ve uygulama repository'nin klonlanmis olmasi.
* Entegrasyon asamasi: ConfigMap, Node.js uygulamasindan olusturuldu ve Kubernetes
kaynaklari da olusturuldu. Sonrasinda, ugulama test edildi ve environment temizlendi.
* Urun asamasi: entegrasyon asamasindaki adimlar uygulandi ancak, Kubernetes kaynaklari urun icinde barinildirilmayacagindan environment temizlik islemi yapilmaz.

Asagida Jenkinsfile'i inceleyebiliriz:

```
//Jenkinsfile

node {
stage('Preparation') {
      //Installing kubectl in Jenkins agent
      sh 'curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl'
  sh 'chmod +x ./kubectl && mv kubectl /usr/local/sbin'
//Clone git repository
  git url:'https://bitbucket.org/advatys/jenkins-pipeline.git'
   }
stage('Integration') {

      withKubeConfig([credentialsId: 'jenkins-deployer-credentials', serverUrl: 'https://104.155.31.202']) {

         sh 'kubectl create cm nodejs-app --from-file=src/ --namespace=myapp-integration -o=yaml --dry-run > deploy/cm.yaml'
         sh 'kubectl apply -f deploy/ --namespace=myapp-integration'
         try{
          //Gathering Node.js app's external IP address
          def ip = ''
          def count = 0
          def countLimit = 10

          //Waiting loop for IP address provisioning
          println("Waiting for IP address")        
          while(ip=='' && count<countLimit) {
           sleep 30
           ip = sh script: 'kubectl get svc --namespace=myapp-integration -o jsonpath="{.items[?(@.metadata.name==\'nginx-reverseproxy-service\')].status.loadBalancer.ingress[*].ip}"', returnStdout: true
           ip=ip.trim()
           count++                                                                              
          }

    if(ip==''){
     error("Not able to get the IP address. Aborting...")
        }
    else{
                //Executing tests
     sh "chmod +x tests/integration-tests.sh && ./tests/integration-tests.sh ${ip}"

     //Cleaning the integration environment
     println("Cleaning integration environment...")
     sh 'kubectl delete -f deploy --namespace=myapp-integration'
         println("Integration stage finished.")   
    }                      

         }
    catch(Exception e) {
     println("Integration stage failed.")
      println("Cleaning integration environment...")
      sh 'kubectl delete -f deploy --namespace=myapp-integration'
          error("Exiting...")                                     
         }
}
   }
 stage('Production') {
      withKubeConfig([credentialsId: 'jenkins-deployer-credentials', serverUrl: 'https://104.155.31.202']) {

       sh 'kubectl create cm nodejs-app --from-file=src/ --namespace=myapp-production -o=yaml --dry-run > deploy/cm.yaml'
sh 'kubectl apply -f deploy/ --namespace=myapp-production'


      //Gathering Node.js app's external IP address
         def ip = ''
         def count = 0
         def countLimit = 10

         //Waiting loop for IP address provisioning
         println("Waiting for IP address")        
         while(ip=='' && count<countLimit) {
          sleep 30
          ip = sh script: 'kubectl get svc --namespace=myapp-production -o jsonpath="{.items[?(@.metadata.name==\'nginx-reverseproxy-service\')].status.loadBalancer.ingress[*].ip}"', returnStdout: true
          ip = ip.trim()
          count++                                                                              
     }

   if(ip==''){
    error("Not able to get the IP address. Aborting...")

   }
   else{
               //Executing tests
    sh "chmod +x tests/production-tests.sh && ./tests/production-tests.sh ${ip}"     
          }                                    
      }
   }
}
```
