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

