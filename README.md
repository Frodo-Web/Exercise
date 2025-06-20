# Описание принятия решений
Rolling update обеспечит плавное добавление и удаление подов, в сочетании, например, с replicas: 3 это будет происходить по принципу 3 + 1, 3 - 1.
```
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```
cpu: 500m в requests приблизительно должно обеспечить пик потребления при старте приложения и первые тяжелые запросы. Затем VPA исходя из метрик requests уменьшит.
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
