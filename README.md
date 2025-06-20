# Описание принятия решений
Rolling update обеспечит плавное добавление и удаление подов, в сочетании, например, с replicas: 3 это будет происходить по принципу 3 + 1, 3 - 1.
```
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```
cpu: 500m в requests должно дать гарантию что в момент первой инициализации будет хватать ресурсов для старта, обработки первых тяжелых запросов. Затем VPA исходя из метрик requests уменьшит, пода будет запущена с меньшими реквестами. <br>
При необходимости, если условия того требуют можно также выставить лимиты
```
          resources:
            requests:
              memory: "128Mi"
              cpu: "500m"
```
Задержки перед readiness пробой, нужно в связи долгой инициализацией приложения ( до  10 секунд)
```
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
```
Если на нодах выставлена зональная метка, как это происходит в облаках, можем на неё положится, чтобы распределить поды. maxSkew позволит соблюдать баланс с разницей не более чем в 1 поду. Например зона A имеет 3 поды, зона Б имеет 2 поды, зона А не сможет запланировать 4-ю в данном случае.
```
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: "topology.kubernetes.io/zone"
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app
```
VPA в автоматическом режиме
```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       "Deployment"
    name:       "my-app"
  updatePolicy:
    updateMode: "Auto"
```
HPA позволяет нам всегда иметь как минимум 3 поды, для распределения по зонам и лучшей отказоустойчивости, и максимум 10, думаю что должно хватить в периоды высоких нагрузок. <br>
HPA полагается на реквесты, не лимиты, и будет добавлять поду, когда утилизация всех текущих подов будет 50%, например, requests 100m, у нас 2 поды с утилизацией по 75m -> HPA добавляет поду.
```
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```
