# h-taxi

## deploy

1. deploy 테스트를 위한 k8s pod 생성

```
# h-taxi-grap.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: h-taxi-grap
spec:
  selector:
    matchLabels:
      run: h-taxi-grap
  replicas: 1
  template:
    metadata:
      labels:
        run: h-taxi-grap
    spec:
      containers:
        - name: h-taxi-grap
          image: devgony/h-taxi-grap:stable
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: h-taxi-grap
  labels:
    run: h-taxi-grap
spec:
  ports:
    - port: 80
  selector:
    run: h-taxi-grap
```

## Gateway
1. gateway 및 virtualService 생성
```
kubectl apply -f - << EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: h-taxi-grap
spec:
  hosts:
    - h-taxi-grap
  http:
  - route:
    - destination:
        host: h-taxi-grap
        subset: v1
      weight: 100
EOF
```
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: h-taxi-grap
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "h-taxi-grap.co.kr"
```
- 서비스 호출 및 VirtualService가 정상적으로 서비스 되고 있음을 확인
```
http http://h-taxi-grap.co.kr/actuator/echo
```

## Circuit Breaker

1. Circuit Breaker 설치
- DestinationRule 생성
```
kubectl apply -f - << EOF
  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  metadata:
    name: h-taxi-grap
  spec:
    host: h-taxi-grap
    trafficPolicy:
      outlierDetection:
        consecutive5xxErrors: 1
        interval: 1s
        baseEjectionTime: 3m
        maxEjectionPercent: 100
EOF
```

2. Circuit Breaker 테스트 환경설정
- Replica를 3개로 늘인다.
```
kubectl scale deploy h-taxi-grap --replicas=3
```
- 새 터미널에서 Http Client 컨테이너를 설치하고, 접속한다.
```
kubectl create deploy siege --image=ghcr.io/acmexii/siege-nginx:latest
kubectl exec -it pod/[SIEGE POD객체] -- /bin/bash
```
3. Circuit Breaker 동작 확인
- 서비스 호출 및 컨테이너가 정상적으로 서비스 되고 있음을 확인
```
http http://h-taxi-grap:8080/actuator/echo
```
- 새로운 터미널에서 마지막에 출력된 Delivery 컨테이너로 접속하여 명시적으로 5xx 오류를 발생 시킨다.
```
# 새로운 터미널 Open
# 3개 중 하나의 컨테이너에 접속
kubectl exec -it pod/[DELIVERY POD객체] -c h-taxi-grap -- /bin/sh
#
# httpie 설치 및 서비스 명시적 다운
apk update
apk add httpie
http PUT http://localhost:8080/actuator/down
```
- Siege로 접속한 이전 터미널에서 Delivery 서비스로 접속해 3회 이상 호출해 본다.
```
http GET http://h-taxi-grap:8080/actuator/health
```
- 최종, 아래 URL을 통해 3개 중 2개의 컨테이너만 서비스 됨을 확인한다.
```
http http://h-taxi-grap:8080/actuator/echo
```
- Pool Ejection 타임(3’) 경과후엔 컨테이너 3개가 모두 동작됨이 확인된다.
```
http http://h-taxi-grap:8080/actuator/echo
```

## Config Map
1. PVC 생성
```
kubectl apply -f - << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fs
  labels:
    app: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Mi
EOF
```
2. Secret 객체 생성
```
kubectl apply -f - << EOF
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  password: YWRtaW4=  
EOF
```

3. 해당 Secret을 h-taxi-grap Deployment에 설정
```
          env:
            - name: superuser.userId
              value: userId
            - name: _DATASOURCE_ADDRESS
              value: mysql
            - name: _DATASOURCE_TABLESPACE
              value: orderdb
            - name: _DATASOURCE_USERNAME
              value: root
            - name: _DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
```
4. MySQL 설치
```
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    name: lbl-k8s-mysql
spec:
  containers:
  - name: mysql
    image: mysql:latest
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-pass
          key: password
    ports:
    - name: mysql
      containerPort: 3306
      protocol: TCP
    volumeMounts:
    - name: k8s-mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: k8s-mysql-storage
    persistentVolumeClaim:
      claimName: "fs"
EOF

kubectl expose pod mysql --port=3306
```
5. Pod 에 접속하여 h-taxi-db 데이터베이스 공간을 만들어주고 데이터베이스가 잘 동작하는지 확인
```
$ kubectl exec mysql -it -- bash

# echo $MYSQL_ROOT_PASSWORD
admin

# mysql --user=root --password=$MYSQL_ROOT_PASSWORD

mysql> create database orderdb;
    -> ;
Query OK, 1 row affected (0.01 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| h-taxi-db            |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)

mysql> exit
```
