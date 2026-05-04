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

