## Домашнее задание по теме Kubernetes - запуск приложений ##  

### Задание 1 - Создать Deployment ###  
  
Создал нэймспэйс dep-ns:  
```
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl create ns dep-ns
namespace/dep-ns created
```  
  
Создал манифест kind: Deployment с одной репликой пода.     
При применении манифеста поды падали в ошибку, из-за пересечения контейнеров по прослушиваемым портам.  
Добавил в шаблон контейнера network-multitool env переменную с номером порта для прослушивания:  
```  
          ports:
            - containerPort: 8080
          env: 
            - name: "HTTP_PORT"
              value: "8080"
```  
  
После применения манифеста deployment.yml можно видеть поды, репликасеты и депломент:  
```
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl apply -f deployment.yml -n dep-ns
deployment.apps/dep-app created
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl get pods -o wide -n dep-ns
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
dep-app-76cd889cd4-gtwtr   2/2     Running   0          8s    10.1.69.204   uxtu-note   <none>           <none>
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl get replicasets -o wide -n dep-ns
NAME                 DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES                                 SELECTOR
dep-app-76cd889cd4   1         1         1       16s   nginx,multitool   nginx:latest,wbitt/network-multitool   app=dep-app,pod-template-hash=76cd889cd4
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl get deployments -o wide -n dep-ns
NAME      READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS        IMAGES                                 SELECTOR
dep-app   1/1     1            1           31s   nginx,multitool   nginx:latest,wbitt/network-multitool   app=dep-app
```  
  
Увеличил в реальном времени количество реплик  
```
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl scale deploy dep-app --replicas=2 -n dep-ns
deployment.apps/dep-app scaled
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl get pods -o wide -n dep-ns
NAME                       READY   STATUS    RESTARTS   AGE    IP            NODE        NOMINATED NODE   READINESS GATES
dep-app-76cd889cd4-gtwtr   2/2     Running   0          3m7s   10.1.69.204   uxtu-note   <none>           <none>
dep-app-76cd889cd4-m66jg   2/2     Running   0          7s     10.1.69.208   uxtu-note   <none>           <none>
```  
  
Создал манифест kind: Service (прилагается)  
Применил манифест:  
```  
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl apply -f service.yml -n dep-ns
service/dep-service created
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl get svc -o wide -n dep-ns
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)               AGE   SELECTOR
dep-service   ClusterIP   10.152.183.221   <none>        10080/TCP,10081/TCP   9s    app=dep-app
```  
  
Создал под для проверки доступности сервиса изнутри k8s
```
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl apply -f pod.yml -n dep-ns
pod/multitool-pod created
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl get pods -n dep-ns -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
dep-app-76cd889cd4-gtwtr   2/2     Running   0          23m   10.1.69.204   uxtu-note   <none>           <none>
dep-app-76cd889cd4-m66jg   2/2     Running   0          20m   10.1.69.208   uxtu-note   <none>           <none>
multitool-pod              1/1     Running   0          8s    10.1.69.229   uxtu-note   <none>           <none>
alex@uxtu-note:~/Study/kuber3/kuber3$ kubectl exec -n dep-ns -it multitool-pod -- bash
multitool-pod:/# curl http://dep-service:10080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy, 
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional 
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
multitool-pod:/# curl http://dep-service:10081
WBITT Network MultiTool (with NGINX) - dep-app-76cd889cd4-gtwtr - 10.1.69.204 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
```   
  

### Задание 2 - Обеспечить старт основного контейнера при выполнении условий ###  
  
Создал манифест сервиса dep2-service:  
```
apiVersion: v1
kind: Service
metadata:
  name: dep2-service
spec:
  selector:
    app: dep2-app
  ports:
    - name: nginx-port
      protocol: TCP
      port: 10082
      targetPort: 80
```  
  

Создал манифест Deployment с initContainers:
```
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: dep2-app
  labels: 
    app: dep2-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dep2-app
  template:
    metadata:
      labels:
        app: dep2-app
    spec:
      containers: 
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
      initContainers:
        - name: dep2-init
          image: busybox:latest
          command: ['sh', '-c', 'until nslookup dep2-service.dep-ns.svc.cluster.local; do echo waiting for myservice; sleep 2; done']
          image: busybox
```  
  
busybox в цикле проверяет разрешение имени сервиса dep2-service, и как только имя начинает разрешаться он успешно завершается.  
Проверил работу:  
```
alex@uxtu-note:~/Study/kuber3/kuber3/task2$ kubectl apply -f deployment.yml -n dep-ns
deployment.apps/dep2-app created
alex@uxtu-note:~/Study/kuber3/kuber3/task2$ kubectl get pods -o wide -n dep-ns
NAME                        READY   STATUS     RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
dep-app-76cd889cd4-gtwtr    2/2     Running    0          78m   10.1.69.204   uxtu-note   <none>           <none>
dep-app-76cd889cd4-m66jg    2/2     Running    0          75m   10.1.69.208   uxtu-note   <none>           <none>
dep2-app-8664778d65-2frz6   0/1     Init:0/1   0          5s    10.1.69.224   uxtu-note   <none>           <none>
multitool-pod               1/1     Running    0          54m   10.1.69.229   uxtu-note   <none>           <none>
alex@uxtu-note:~/Study/kuber3/kuber3/task2$ kubectl apply -f service.yml -n dep-ns
service/dep2-service created
alex@uxtu-note:~/Study/kuber3/kuber3/task2$ kubectl get pods -o wide -n dep-ns
NAME                        READY   STATUS            RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
dep-app-76cd889cd4-gtwtr    2/2     Running           0          78m   10.1.69.204   uxtu-note   <none>           <none>
dep-app-76cd889cd4-m66jg    2/2     Running           0          75m   10.1.69.208   uxtu-note   <none>           <none>
dep2-app-8664778d65-2frz6   0/1     PodInitializing   0          39s   10.1.69.224   uxtu-note   <none>           <none>
multitool-pod               1/1     Running           0          55m   10.1.69.229   uxtu-note   <none>           <none>
alex@uxtu-note:~/Study/kuber3/kuber3/task2$ kubectl get pods -o wide -n dep-ns
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
dep-app-76cd889cd4-gtwtr    2/2     Running   0          78m   10.1.69.204   uxtu-note   <none>           <none>
dep-app-76cd889cd4-m66jg    2/2     Running   0          75m   10.1.69.208   uxtu-note   <none>           <none>
dep2-app-8664778d65-2frz6   1/1     Running   0          48s   10.1.69.224   uxtu-note   <none>           <none>
multitool-pod               1/1     Running   0          55m   10.1.69.229   uxtu-note   <none>           <none>
```  
  
  
