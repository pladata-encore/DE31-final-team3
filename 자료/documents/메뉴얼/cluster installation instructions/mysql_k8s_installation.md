# mysql 설치

on-premise k8s 클러스터 상에 `mysql`을 설치하는 방법을 기록한다.

우선 다음의 명령어를 통해 `mysql`과 관련된 클러스터 구성 오브젝트를 담아두기 위한 namespace를 생성한다.

```bash
kubectl create namespace mysql
```

## volume binding

데이터를 지속적으로 담아둘 pv와 pvc를 구성한다.

만약 따로 volume이 묶여있지 않다면 pod가 죽는 즉시 데이터도 사라지기 때문에 필요하다.

다음의 yaml을 작성한다.

`mysql-pv.yaml`
```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-mysql-com3
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: mysql
  hostPath:
    path: /mnt/data/mysql
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - com3
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: mysql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: mysql
```

단, 클러스터에 `com3`라는 이름으로 묶인 노드에 해당 디렉토리가 존재해야 한다.

때문에 yaml 파일을 작성하고 적용하기 전에 다음 명령어를 실행한다.

```bash
sudo mkdir -p /mnt/data/mysql
sudo chmod 777 -R /mnt/data/mysql
```

그 후 다음의 명령어를 통해 작성한 yaml을 클러스터에 적용해 pv, pvc를 생성한다.

```bash
kubectl apply -f mysql-pv.yaml
```

## secrets

공식 kubernetes 홈페이지에서 가이드하는 `mysql` pod deployment 생성을 위한 yaml에도 root 계정의 비밀번호를 그대로 사용하기 보다는 secret을 사용하는 것을 추천한다.

따라서 `mysql`의 root 계정의 비밀번호를 담기 위한 k8s secret을 다음의 yaml을 기반으로 생성한다.

`mysql-secret.yaml`
```bash
apiVersion: v1
kind: Secret
metadata:
  name: mysql-root
  namespace: mysql
type: Opaque
data:
  password: BASE64_ENCODED_STRING
```

위의 yaml 파일에서 `BASE64_ENCODED_STRING` 부분은 실제로 비밀번호로 활용하고자 하는 문자열을 base64로 인코딩한 것이다.

예시를 들자면 다음의 명령어를 통해 `testpw`라는 문자열을 base64로 인코딩할 수 있다.

```bash
echo -n testpw | base64
```

원하는 문자열을 base64로 인코딩한 이후 해당 문자열로 `BASE64_ENCODED_STRING`을 대체한 이후 파일을 저장하고 해당 yaml을 다음 명령어로 k8s 클러스터에 적용한다.

```bash
kubectl apply -f mysql-secret.yaml
```

정상적으로 적용되었는지는 다음 명령어를 통해 확인할 수 있다.

```bash
kubectl describe secret mysql-root -n mysql
```

## deployments

`mysql`을 실질적으로 설치하기 위해 앞선 pvc와 secret을 적용한 yaml 파일의 내용은 다음과 같다.

`mysql-deployment.yaml`
```bash
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: mysql
spec:
  type: LoadBalancer
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-root
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

위 yaml에는 LoadBalancer와 Deployment를 함께 구성했다.

만약 service를 외부에서 사용하지 않는다면 ClusterIP 또는 NodePort로 변경해도 무방하다.

다음의 명령어를 통해 해당 yaml을 클러스터에 적용한다.

```bash
kubectl apply -f mysql-deployement.yaml
```

정상적으로 적용되었다면 다음의 명령어를 통해 실행되고 있는 pod를 확인할 수 있다.

```bash
kubectl get pods -n mysql
```
