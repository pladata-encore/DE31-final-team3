# spark 클러스터 구축

ubuntu 22.04 server 버전의 온프레미스 환경에서 Spark 클러스터를 구성하기 위한 단계별 방법을 기록한다.

## 초기 네트워크 환경 설정

1. hostname 설정

`/etc/hosts` 파일을 편집해서 노드로 사용할 각각의 컴퓨터들의 hostname을 기록해두어야 할 필요가 있다.

이를 통해 spark 클러스터에서 각각의 노드를 ip가 아니라 hostname으로 찾을 수 있다.

예시를 들어서, ip가 `192.168.0.100`인 컴퓨터를 master 노드로 사용하고, ip 주소가 각각 `192.168.0.101`, `192.168.0.102`인 컴퓨터 2대를 worker 노드로 사용하려고 한다고 가정하자.

그렇다면 `/etc/hosts`파일에 다음과 같은 내용을 추가해야한다.

```config
...
192.168.0.100 com0
192.168.0.101 com1
192.168.0.102 com2
```

위에서 추가한 내용을 통해, `com0`, `com1`, `com2`라는 hostname으로 각각의 노드에 접근할 수 있다.

2. ssh 연결

기본적으로 클러스터를 구축하고자 한다면 클러스터를 구성하는 노드들이 다른 노드에 ssh를 통해 접근할 때 비밀번호 또는 로그인을 요구하지 않아야 할 필요가 있다.

다음의 명령어를 통해 ssh 키를 새롭게 생성한다.

이때 passphrase는 없이 생성한다.

```bash
ssh-keygen
```

명령어를 실행한다면 default로 home 디렉토리 내부의 `.ssh/` 디렉토리 아래에 `id_rsa`와 `id_rsa.pub`라는 이름의 private/public ssh 키가 생성된다.

추가적으로 다음의 명령어를 통해 비밀번호 없이 로그인이 가능하도록 설정한다.

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

위 명령어를 통해 `~/.ssh` 디렉토리의 아래에 `authorized_keys` 파일에 `id_rsa.pub` 파일의 내용을 추가해 같은 `id_rsa.pub` 파일을 가지고 있다면 비밀번호없이 ssh로 접근 가능하도록 할 수 있다.

`id_rsa`, `id_rsa.pub`, `authorized_keys` 총 3개의 파일을 worker 노드로 삼을 컴퓨터들에 다음 명령어를 통해 복제해준다.

```bash
cd ~/.ssh
scp id_rsa id_rsa.pub authorized_keys user@com1:/home/user/.ssh/
```

각 노드의 유저이름에 알맞게 `user`라는 이름을 변경해준다.

추가적으로, ssh key 파일들은 적절한 권한이 설정되어야 ssh를 통해 접속할 때 문제가 발생하지 않는다.

각 노드에서 ssh를 통해 접속해서 또는 직접적으로 터미널에서 다음 명령어를 실행해준다.
```bash
chmod 700 /home/user/.ssh
chmod 600 /home/user/.ssh/id_rsa
chmod 644 /home/user/.ssh/id_rsa.pub
chmod 600 /home/user/.ssh/authorized_keys
```

## 스파크 설치

1. [공식 홈페이지](https://dlcdn.apache.org/spark/spark-3.5.1/spark-3.5.1-bin-hadoop3.tgz)에서 wget 명령어로 파일 다운로드

파일을 다운로드받은 다음 압축을 해제하고, 단순한 이름으로 바꿔준 다음 `/opt/`디렉토리 아래에 옮기는 과정을 다음의 명령어로 실행한다.

```bash
wget https://dlcdn.apache.org/spark/spark-3.5.1/spark-3.5.1-bin-hadoop3.tgz
tar xvfz spark-3.5.1-bin-hadoop3.tgz
sudo mv spark-3.5.1-bin-hadoop3 /opt/spark
sudo chown -R 1000:1000 /opt/spark/
```	

추가적으로 발생할 수 있는 소유권 이슈를 막기 위해 uid가 1000인 사용자를 기준으로 chown 명령어를 실행해 소유권을 가져온다.

혹시 사용자의 uid가 다르다면 uid에 맞게 변경해서 chown 명령어를 적용해준다.

2. java 설치

Java 17에서는 모듈 시스템이 더욱 강화되면서 모듈의 경계를 넘는 클래스 접근 제한이 강화되었다.

이로 인해 낮은 버전의 spark에서는 기존 라이브러리들이 보안 관련 이슈가 발생하며 제대로 context를 생성하지 못하는 버그가 존재했고, 해당 버그는 Java 11 버전으로 다운그레이드하는 것으로 해결되었다..

비록 현재 설치된 spark 버전 3.5.1이 Java 17과 호환된다고 해도 잠재적인 버그 발생을 방지하기 위해 Java 11을 계속해서 사용했다.

```bash
sudo apt update && sudo apt upgrade -y && sudo apt install openjdk-11-jdk -y
```

3. 환경변수 추가.

ubuntu 22.04 server 환경에서 환경 변수를 추가하려면 `.bashrc` 파일에 추가하는 방식으로 추가할 수 있다.

```bash
vim ~/.bashrc
```

다음의 환경 변수들을 `.bashrc` 파일에 추가하여 spark를 실행할 환경을 구성해준다.

```bash
###### SPARK #####
export SPARK_HOME=/opt/spark
export PATH=$SPARK_HOME/bin
alias spark_start="/opt/spark/sbin/start-all.sh"
alias spark_stop="/opt/spark/sbin/stop-all.sh"
```

작성한 `.bashrc`를 적용해주기 위해 다음의 명령어를 실행한다.

```bash
source ~/.bashrc
```

4. spark 설정 구성

기본적으로 spark를 홈페이지에서 다운로드하면 해당 압축파일에 template가 존재한다.

이 template에서는 기본적인 설정값이 구성되어 있지만 추가적으로 클러스터를 구성하기 위해서는 네트워크 설정과 함께 연동해서 설정값을 추가로 작성해주어야 한다.

우선적으로 다음의 명령어를 실행해 spark 클러스터의 초기 설정 template를 복사해준다.

```bash
cp /opt/spark/conf/spark-env.sh.template /opt/spark/conf/spark-env.sh
vim /opt/spark/conf/spark-env.sh
```

그 후 다음의 변수들을 `spark-env.sh`라는 이름으로 복사한 설정 파일에 추가해준다.

```shell
# Options for the daemons used in the standalone deploy mode
# - SPARK_MASTER_HOST, to bind the master to a different IP address or hostname
SPARK_MASTER_HOST=MASTER_NODE_HOSTNAME
# - SPARK_MASTER_PORT / SPARK_MASTER_WEBUI_PORT, to use non-default ports for the maste
# - SPARK_MASTER_OPTS, to set config properties only for the master (e.g. "-Dx=y")
# - SPARK_WORKER_CORES, to set the number of cores to use on this machine
SPARK_WORKER_CORES=NUMBER_OF_CORE_TO_USE_ON_WORKER
# - SPARK_WORKER_MEMORY, to set how much total memory workers have to give executors (e.g. 1000m, 2g)
SPARK_WORKER_MEMORY=TOTAL_SIZE_OF_MEMORY_TO_USE_ON_A_WORKER
```

위의 변수들 중에서 `MASTER_NODE_HOSTNAME`, `NUMBER_OF_CORE_TO_USE_ON_WORKER`, `TOTAL_SIZE_OF_MEMORY_TO_USE_ON_A_WORKER`는 각각 적절한 값으로 변경해주면 된다.

예를 들어서, `/etc/hosts` 파일의 내부에 `com0`라는 이름으로 저장된 기기를 master 노드로 사용하고, worker 노드에서 사용할 코어의 갯수와 메모리 크기를 각각 4개, 10G로 적용하고 싶다면 다음과 같이 변수를 적용할 수 있다.

```sh
SPARK_MASTER_HOST=com0
SPARK_WORKER_CORES=4
SPARK_WORKER_MEMORY=10g
```

spark를 클러스터 모드로 동작시키기 위해서는 정확한 이름의 worker 노드를 배정하는 것이 필요하다.

worker 노드를 배정하기 위해서는 `workers`라는 이름의 파일을 생성해 해당 파일 안에 `/etc/hosts`에 존재하는 worker 노드의 hostname을 기입하면 된다.

`spark-env.sh`의 경우와 마찬가지로 `workers` 파일 또한 template이 존재하고, 이를 복사해 수정하는 것도 가능하지만 단순히 hostname의 나열이기에 바로 작성해도 무방하다.

다음의 명령어를 통해 workers 파일을 작성한다.

```bash
vim /opt/spark/conf/workers
```

예를 들어서, `com1`과 `com2`라는 이름의 worker 노드들을 클러스터로 구성하고 싶다면 workers 파일을 다음과 같이 작성한다.

```config
com1
com2
```

이때 주의할 점은 `/etc/hosts`에서 인식하는 hostname이 동일해야 한다는 것이다.

worker 노드에서도 동일한 작업을 해주되, 각각 hostname이 정확하게 매치되도록 작성해주어야 한다.

단순한 방법으로는 각 노드들의 `/etc/hosts` 파일에 `com0`, `com1`, `com2`라는 hostname의 각 ip 주소를 추가하여 hostname을 기록한 다음 `/opt/spark/` 디렉토리 자체를 scp를 통해 복제해서 전송하는 것으로 같은 설정값을 공유하게 하여 클러스터를 구성하는 방법이 있다.

앞서 설명한 설정들이 완료되었다면 다음의 명령어를 통해 spark 클러스터를 구동한다.

```bash
spark_start
```

정상적으로 클러스터가 구동되었다면 각각의 노드에서 `jps`명령어를 통해 master 또는 worker 노드가 실행되고 있는지 확인할 수 있다.

5. SparkSession 작성

ubuntu에 설치되어 있는 기본적인 python 버전은 3.10이다.

```bash
sudo apt update && sudo apt install python3-dev python3-pip python3-venv -y
```

위의 명령어를 통해 python의 가상환경과 pip를 설치해준다.

이 후 가상환경 또는 conda를 구축해 실행한 후 다음 명령어를 실행해준다.

```bash
python3 -m pip install -U pip && python3 -m pip install pyspark pandas
```

위의 명령어를 통해 pip에서 pyspark와 pandas를 설치해준다.

pyspark를 통해 SparkSession을 생성하고 동작하는 예시 코드는 다음과 같다.

```python
import pyspark
from pyspark.sql import SparkSession
import pandas as pd
conf = pyspark.SparkConf() \
            .setAppName("pysparkTest") \
            .setMaster("spark://com0:7077") \
            .set("spark.blockManager.port", "10025") \
            .set("spark.driver.blockManager.port", "10026") \
            .set("spark.driver.port", "10027") \
            .set("spark.cores.max", "2")
spark = SparkSession.builder.config(conf=conf).getOrCreate()
```

위의 코드가 동작하기 위해서는 총 4개의 포트가 각각의 노드들에 열려있어야 한다.

필요한 포트는 7077, 10025, 10026, 10027이다.

추가적으로 master 노드에서 제공하는 웹 UI에 접근하기 위해서는 8080 포트 또한 열려있어야 한다.

다음의 명령어를 통해 ubuntu의 방화벽 관리 어플리케이션인 ufw를 통해 각 포트를 공개할 수 있다.

```bash
sudo ufw allow 7077/tcp && sudo ufw allow 8080/tcp && sudo ufw allow 10025/tcp && sudo ufw allow 10026/tcp && sudo ufw allow 10027/tcp
```

위의 명령어가 정상적으로 실행되었는지 확인하려면 다음의 명령어를 실행할 수 있다.

```bash
sudo ufw status
```

공개된 포트들의 목록 중에 위의 포트들이 포함되어 있으면 된다.