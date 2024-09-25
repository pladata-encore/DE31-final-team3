# airflow 클러스터 구축

[참조](https://developnote-blog.tistory.com/176)

8월 5일부터 `airflow` 클러스터 설치를 진행하는 과정에서 본래 `celery executor` 모드로 클러스터를 구축하려고 했다.

그러나 애초에 k8s 클러스터에 올리는 시점에서 그냥 `kubernetes executor`로 실행시키기로 했다.

그 과정에서 위의 참조 링크를 따라 설치를 진행했다.

git의 ssh-key를 추가하고 이를 클러스터에 secret으로 적용하는 방법과 간단한 16진수 crypto를 만들어 flask의 secret으로 클러스터에 적용하는 방법 또한 위의 참조 링크를 따라하면 된다.

본래는 별개의 `postgresql`과 `redis`를 설치하고 `airflow`를 설치하려고 하였으나 지속적으로 버그가 발생하였기 때문에 `postgresql`은 그냥 `helm` 차트에서 설치시에 제공해주는 구성을 사용하고, `redis`는 사용하지 않기로 했다.

## helm chart를 통한 airflow 설치

### helm repo add

만일 helm repo의 목록으로 `apache-airflow`가 추가되어 있지 않다면 우선적으로 추가해주어야 한다.

다음의 명령어를 통해 추가할 수 있다.

```bash
helm repo add apache-airflow https://airflow.apache.org
helm repo list
```

### helm pull

repo가 추가되었다면, chart를 pull해올 수 있다.

tgz 형태의 압축 파일로 받아서 unzip 하기 위한 명령어는 다음과 같다.

```bash
helm pull apache-airflow/airflow
tar xvfz airflow-1.15.0.tgz # version may differ
cd airflow
```

만약 tgz 말고 그냥 디렉토리째로 pull하려면 --untar 키워드를 추가로 줄 수 있다.

```bash
helm pull apache-airflow/airflow --untar
cd airflow
```

## pv, pvc 설정

### airflow-pv.yaml

`airflow`의 log와 `postgresql`에 직접 묶어줄 volume을 설정해줘야 한다.

다음의 yaml 파일을 작성해 클러스터에 적용해 준다.

`airflow-pv.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-airflow-com2
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: airflow
  hostPath:
    path: /mnt/data/airflow
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - com2
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: airflow-logs-pvc
  namespace: airflow
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: airflow
```

기본적으로 위의 yaml은 클러스터의 노드중 `com2`라는 노드 내부에 존재하는 `/mnt/data/airflow`라는 디렉토리에 할당되게 구성했다.

`airflow`의 경우 각 파드에서 생성하는 유저의 uid가 다르기 때문에 그냥 마운트되는 디렉토리의 권한 자체를 전부 허용하는 방식으로 해결했다.

`com2`라는 이름의 노드에서 다음의 명령어로 적용한다.

```bash
sudo mkdir -p /mnt/data/airflow
sudo chmod 777 -R /mnt/data/airflow
```

### postgresql-pv.yaml

---

`airflow`에서 metadata 저장소로 사용하는 `postgresql`의 경우 `apache-airflow` `helm` 차트를 통해서 설치한다면 기본적으로 함께 설치된다.

기존에는 별개의 `postgresql`을 설치해 해결하는 방향으로 구축하려 했으나 `bitnami` 레포를 통해 설치한 `postgresql-HA`에서 원인을 알기 어려운 여러 버그들이 발생하는 상황에서 `airflow` 스케줄러와 웹서버가 설치된 `postgresql`을 찾지 못하는 문제가 발생하면서 `airflow` 차트에서 제공하는 `postgresql`을 사용하는 쪽으로 방향을 전환했다.

airflow 차트에서 제공되는 `postgresql`을 사용하는 경우에도 동일하게 pv와 pvc를 배정해주어야 한다.

다음의 yaml 파일을 작성해 클러스터에 적용한다.

`postgresql-pv.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgresql-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: postgresql
  hostPath:
    path: /mnt/data/postgresql
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - com1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
  namespace: airflow
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: postgresql
```

`postgresql`의 경우에도 `airflow`의 경우와 동일한 방식의 명령어를 적용한다.

```bash
sudo mkdir -p /mnd/data/postgresql
sudo chmod 777 -R /mnt/data/postgresql
```

---

## airflow values.yaml override

`helm`을 통해 `airflow`를 설치하기 위해 설정값을 덮어씌울 `values-override.yaml`을 작성해준다.

앞서 설정한 pvc들을 바라보도록 `airflow`의 logger와 `postgresql`의 persistence 설정을 잡아준다.

`values-override.yaml`
```yaml
executor: "KubernetesExecutor"
webserverSecretKey: webserver-secret-key
webserverSecretKeySecretName: airflow-webserver-secret
data:
  metadataConnection:
    user: user
    pass: password
    db: airflow
dags:
  gitSync:
    enabled: true
    repo: git@github.com:S0rrow/FPT5.git
    branch: dev
    rev: HEAD
    depth: 1
    maxFailures: 3
    subPath: dags
    sshKeySecret: "airflow-ssh-git-secret"
    wait: 60
scheduler:
  livenessProbe:
    initialDelaySeconds: 10
    timeoutSeconds: 120
    failureThreshold: 20
    periodSeconds: 60
  replicas: 2
webserver:
  service:
    type: NodePort # LoadBalancer
    ports:
       - name: airflow-ui
         port: "{{ .Values.ports.airflowUI }}"
         targetPort: "{{ .Values.ports.airflowUI }}"
         nodePort: 31151
logs:
  persistence:
    enabled: true
    existingClaim: airflow-logs-pvc
    size: 10Gi

postgresql:
  enabled: true
  image:
    tag: 11.22.0
  auth:
    enablePostgresUser: true
    postgresPassword: postgres
    username: user
    password: password
    database: "airflow"
  primary:
    service:
      type: NodePort
      nodePorts:
        postgresql: 30032
    persistence:
      existingClaim: postgresql-pvc
      size: 20Gi
redis:
  enabled: false
env:
  #  - name: "AIRFLOW__LOGGING_REMOTE_LOGGING"
  #  value: "True"
  #- name: "AIRFLOW__LOGGING__REMOTE_LOG_CONN_ID"
  #  value: "S3Conn"
  #- name: "AIRFLOW__LOGGING__REMOTE_BASE_LOG_FOLDER"
  #  value: "s3://{bucket_name}/airflow/logs"
  #- name: "AIRFLOW__LOGGING__ENCRYPT_S3_LOGS"
  #  value: "False"
  - name: "AIRFLOW__CORE__DEFAULT_TIMEZONE"
    value: "Asia/Seoul"
```

기존에 참조했던 링크의 경우에는 AWS의 S3를 사용한다는 가정 하에 버켓의 이름과 원격 연결을 구성하였으나, 현재 내가 구성하는 환경의 경우 온프레미스 환경에서 로컬 PV를 연결해주었기 때문에 해당 변수들을 주석처리했다.

위의 파일을 앞서 `helm`을 통해 pull했던 airflow 디렉토리 아래에 위치시키고 해당 위치에서 다음의 명령어를 실행한다.

```bash
kubectl create ns airflow # ns는 namespace의 축약
helm install airflow -n airflow -f values-override.yaml ./ # 현재 디렉토리의 Charts를 사용해 설치
```

설치가 완료되었다면 `airflow-webserver`라는 이름의 웹 서버 역할을 담당하는 NodePort 서비스가 생성된다.

해당 서비스의 내용을 편집하여 LoadBalancer로 수정해 포트포워딩 없이도 외부에서 접속 가능하도록 수정한다.

```bash
kubectl edit svc airflow-webserver -n airflow
```

정상적으로 설치와 pod 초기화가 완료되었고 모든 pvc가 연결되었다면 웹 서버에 할당된 외부 ip 주소로 8080 포트로 접속 가능할 것이다.

혹시나 서버나 스케줄러가 Crash되거나 초기화에 실패하는 경우 git 레포지토리와의 연동 또는 주소가 정확한지, SSH 키가 제대로 생성되고 적용되었는지 여부를 확인해볼 필요가 있다.
