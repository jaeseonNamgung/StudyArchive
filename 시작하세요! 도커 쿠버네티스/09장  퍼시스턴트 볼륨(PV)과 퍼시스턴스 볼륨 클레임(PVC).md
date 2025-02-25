디플로이먼트는 상태가 없는(stateless) 애플리케이션이다. 디플로이먼트의 각 파드는 별도의 데이터를 가지고 있지 않으며(단순히 요청 응답만을 반환한다.) 파드가 삭제되면 파드의 저장된 데이터도 삭제된다. (영속적이지 않는다.) 또한 특정 노드에서만 데이터를 저장하면 파드가 다른 노드로 옮겨갔을 때 해당 데이터를 사용할 수 없게 된다. 이를 해결할 수 있는 방법으로 퍼시스턴스 볼륨(PV)가 있다.

**퍼시스턴스 볼륨은 워커 노드들이 네트워크상에서 스토리지를 마운트해 영속적으로 데이터를 저장할 수 있다.**

파드에 장애가 생겨, 다른 노드로 파드가 옮겨 가도 해당 노드에서 퍼시스턴스 볼륨에 네트워크로 연결해 데이터를 계 사용 가능하다.

# 로컬 볼륨: hostPath, emptyDir

1. `hostPath` : 호스트의 디렉토리를 파드와 공유

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: my-container
    image: busybox
    args: [ "tail", "-f", "/dev/null" ]
    volumeMounts:
    - name: my-hostpath-volume
      mountPath: /etc/data
  volumes:
    - name: my-hostpath-volume
      hostPath:
        path: /tmp
```

- `emptyDir` : 파드의 컨테이너 간에 볼륨을 공유하기 위해서 사용

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: content-creator
    image: alicek106/alpine-wget:latest
    args: ["tail", "-f", "/dev/null"]
    volumeMounts:
    - name: my-emptydir-volume
      mountPath: /data                      # 1. 이 컨테이너가 /data 에 파일을 생성하면
  - name: apache-webserver
    image: httpd:2
    volumeMounts:
    - name: my-emptydir-volume
      mountPath: /usr/local/apache2/htdocs/  # 2. 아파치 웹 서버에서 접근 가능합니다.

  volumes:
    - name: my-emptydir-volume
      emptyDir: {}                             # 포드 내에서 파일을 공유하는 emptyDir
```

# 네트워크 볼륨

네트워크 볼륨의 위치는 특별히 정해진 것이 없으며, 네트워크로 접근만 할 수만 있으면 쿠버네티스 클러스터 내부, 외부 어느 곳에 존재해도 접근이 가능하다.

## NFS를 네트워크 볼륨으로 사용하기

- NFS(Network File System)는 대부분의 운영 체제에서 사용할 수 있는 네트워크 스토리지로, 여러 개의 클라이언트가 동시에 마운트해 사용할 수 있다는 특징이 있다.
- 하나의 서버만으로도 간편하게 사용할 수 있으며, NFS를 마치 로컬 스토리지처럼 사용할 수 있다.

- NFS 서버: 영속적인 데이터가 실제로 저장되는 네트워크 스토리지 서버
- NFS 클라이언트: NFS 서버에 마운트해 스토리지에 파일을 읽고 쓰는 역할

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
spec:
  selector:
    matchLabels:
      role: nfs-server
  template:
    metadata:
      labels:
        role: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: gcr.io/google_containers/volume-nfs:0.8
        ports:
        - name: nfs
          containerPort: 2049
        - name: mountd
          containerPort: 20048
        - name: rpcbind
          containerPort: 111
        securityContext:
          privileged: true
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nfs-service
spec:
  ports:
  - name: nfs
    port: 2049
  - name: mountd
    port: 20048
  - name: rpcbind
    port: 111
  selector:
    role: nfs-server
```

```yaml
# NFS 서버의 볼륨을 파드에서 마운트
apiVersion: v1
kind: Pod
metadata:
  name: nfs-pod
spec:
  containers:
    - name: nfs-mount-container
      image: busybox
      args: [ "tail", "-f", "/dev/null" ]
      volumeMounts:
      - name: nfs-volume
        mountPath: /mnt           # 포드 컨테이너 내부의 /mnt 디렉터리에 마운트합니다.
  volumes:
  - name : nfs-volume
    nfs:                            # NFS 서버의 볼륨을 포드의 컨테이너에 마운트합니다.
      path: /
      server: {NFS_SERVICE_IP}
```

# PV, PVC를 이용한 볼륨 관리

쿠버네티스에서 제공하는 대부분의 볼륨은 파드나 디플로이먼트의 yaml 파일에서 직접 작성해서 사용 가능하다.  그러나 이렇게 yaml 파일에서 직접 작성하면 볼륨과 애플리케이션의 정의가 서로 밀접하게 연관되어 분리할 수 없는 상황이 발생한다.

이러한 불편함을 해결하기 위해  **쿠버네티스에선 퍼시스턴트 볼륨(Persistent Volume, PV)과 퍼시스턴트 볼륨 클레임(Persistent Volume Claim, PVC)이라는 오브젝트를 제공한다.**

이 두개의 오프젝트는 파드가 볼륨의 세부적인 사항을 알지 못해도 볼륨을 사용할 수 있도록 추상화해 주는 역할을 담당한다.

> PV는 특정 namespace에 속하지 않지만 PVC는 특정 네임스페이스에 속해있다.
>

# PV, PVC 사용하기

```bash
$ export VOLUME_ID=$(aws ec2 create-volume --size 5 \
--region ap-northeast-2 \
--availability-zone ap-northeast-2a \
--volume-type gp2 \
--tag-specifications \
'ResourceType=volume, Tags=[{Key=KubernetesCluster,Value=mycluster.k8s.local}]' \
| jq '.VolumeId' -r)

$ echo $VOLUME_ID
vol-02178d79...
```

```yaml
# EBS 볼륨을 통해 쿠버네티스의 퍼시스턴트 볼륨을 생성
# ebs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv
spec:
  capacity:
    storage: 5Gi # 볼륨의 크기는 5G
  accessModes:
    - ReadWriteOnce # 하나의 파드(또는 인스턴스)에 의해서만 마운트될 수 있다.
  awsElasticBlockStorage:
    fsType: ext4
    volumeID: <VOLUME_ID>
```

```yaml
# 퍼시스턴트 볼륨 클레임과 포드를 생성
# ebs-pod-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-ebs-pvc
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce # 속성이 ReadWriteOnce인 퍼시스턴트 볼륨과 연결한다.
  resources:
    requests:
      storage: 5Gi  # 볼륨 크기가 최소 5G인 퍼시스턴트 볼륨과 연결한다.
---
apiVersion: v1
kind: Pod
metadata:
  name: ebs-mount-container
spec:
  containers:
    - name: ebs-mount-container
      mage: busybox
      args: [ "tail", "-f", "/dev/null" ]
      volumeMounts:
      - name: nfs-volume
        mountPath: /mnt
  volumes:
  - name: ebs-volume
    persistentVolumeClaim:
      claimName: my-ebs-pvc # my-ebs-pvc라는 이름의 pvc를 사용
```

만약, pvc의 요구사항과 일치하는 pv가 존재하지 않는다면 파드는 계속해서 `pending` 상태로 남아있게 된다. 쿠버네티스는 계속해서 주기적으로 조건이 일치하는 pv를 체크해 연결하려고 시도하기 때문에 조건에 만족하는 pv를 새롭게 생성한다면 자동으로 pvc와 연결된다.

# **퍼시스턴트 볼륨을 선택하기 위한 조건 명시**

퍼시스턴스 클레임을 사용하면 실제로 볼륨이 어떤 스펙을 가졌는지 알 필요가 없다. 하지만 사용하려는 볼륨이 애플리케이션의 최소 조건을 맞출 필요는 있다.

- AccessModes나 볼륨 크기가 이러한 조건에 해당한다.
- 퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임의 `accessMode` 및 볼륨 크기 속성이 부합해야만 쿠버네티스는 두 리소스를 매칭해 바인드한다.

[accessModes 종류]

- ReadWriteOnce(RWO): 읽기 쓰기가 가능하며 1:1 마운트
- ReadOnlyMany(ROX): 읽기만 가능하며 1:N 마운트
- ReadWriteMany(RWX): 읽기 쓰기가 가능하며 1:N 마운트

> EBS 는 읽기 쓰기가 모두 가능하면 1:1 관계의 마운트만 가능 (하나의 EBS에 하나의 파드만 마운트 가능) → RWO 사용이 바람직함
>
>
> NFS는 1:N 마운트가 가능 → RWX 사용이 바람직함
>

# **퍼시스턴트 볼륨와 라이프사이클과 Reclaim Policy**

pv를 생성한 뒤, `kubectl get pv` 명령어로 목록을 확인해보면 `STATUS`라는 항목을 볼 수 있다. **`STATUS` 항목은 퍼시스턴트 볼륨이 사용 가능한지, 퍼시스턴트 볼륨 클레임과 연결됐는지 등을 의미한다.**

- Avaliable: 사용 가능 상태
- Bound: 연결된 상태. 다른 퍼시스턴트 볼륨 클레임과 연결할 수 없다.
- Released: 해당 퍼시스턴트 볼륨의 사용이 끝났다는 것을 의미. 이 상태에 있는 퍼시스턴트 볼륨은 다시 사용할 수 없다.

퍼시스턴트 볼륨 클레임을 삭제했을 때, 퍼시스턴트 볼륨의 데이터를 어떻게 처리할 것인지 별도로 정의할 수 있다. 퍼시트턴트 볼륨의 사용이 끝났을 때 해당 볼륨을 어떻게 초기화할 것인지 별도로 설정할 수 있는데 이를 `Reclaim Policy` 라고 부른다.  크게 `Retain`, `Delete`, `Recycle` 방식이 있다.

Reclaim Policy의 기본값인 `Retain`으로 설정돼 있다면 퍼시스턴트 볼륨의 라이프사이클은 `Available` → `Bound` → `Released`가 된다.

- `Retain` : 연결된 퍼시스턴트 볼륨 클레임 삭제 후에 `Released` 전환, 실제 데이터는 보존
- `Delete` : 퍼시스턴트 볼륨 사용 종료 이후 자동적으로 삭제됨 (**가능한 경우에 한해선 연결된 외부 스토리지도 함께 삭제된다.** )
- `Recycle` : 퍼시스턴트 볼륨 클레임 삭제시, 퍼시스턴트 볼륨의 모든 데이터를 삭제 후 `Available` 로 전환 (Deprecated 됨)

# **StorageClass와 Dynamic Provisioning**

- 지금까지 퍼시스턴트 볼륨을 사용하려면 미리 외부 스토리지를 준비해야 했다.
- 매번 이렇게 볼륨 스토리지를 직접 수동으로 생성하고, 스토리지에 대한 접근 정보를 yaml 파일에 적는 것은 비효율적이다.

이를 위해 쿠버네티스는 **다이나믹 프로비저닝(Dynamic Provisioning)** 이라는 기능을 제공한다. 이는 **퍼시스턴트 볼륨 클레임이 요구하는 조건과 일치하는 퍼시스턴트 볼륨이 존재하지 않는다면 자동으로 퍼시스턴트 볼륨과 외부 스토리지를 함께 프로비저닝하는 기능이다.** 따라서 이 기능을 사용하면 EBS와 같은 외부 스토리지를 직접 미리 생성해 둘 필요가 없다. 퍼시스턴트 볼륨 클레임을 생성하면 외부 스토리지가 자동으로 생성되기 때문이다.

퍼시스턴트 볼륨을 선택하기 위해 스토리지 클래스를 사용했었는데, 이는 다이나믹 프로비저닝에도 사용할 수 있다. 다이나믹 프로비저닝은 스토리지 클래스의 정보를 참고해 외부 스토리지를 생성하기 때문이다.

단 다이나믹 프로비저닝이 지원 가능한 스토리지 프로비저가 미리 활성화 되어 있어야 한다.

> 퍼시스턴스 볼륨 클레임에 storageClassName 항목을 아예 정의하지 않았을 때, 쿠버네티스에서 기본적으로 사용하도록 설정된 스토리지 클래스가 있다면 해당 스토리지 클래스를 통해 다이나믹 프로비저닝이 수행된다. 따라서 다이나믹 프로비저닝을 아예 사용하지 않을 것이라면 storageClassName: “” 와 같이 명시적으로 공백으로 설정하는 것이 좋다.
>

```yaml
# storageclass-slow.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: st1
  fsType: ext4
  zones: ap-northeast-2a
```

```yaml
# storageclass-fast.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
  zones: ap-northeast-2a
```

- `provisioner`:  EBS 동적 프로비저너를 설정
- `type`: EBS 종류

```yaml
# pvc-fast-sc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-fast-sc
spec:
  storageClassName: fast
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**다이나믹 프로비저닝을 사용할 때 주의해야 할 점은 퍼시스턴트 볼륨의 Reclaim Policy가 자동으로 `Delete`로 설정된다는 것이다.**

따라서 퍼시스턴트 볼륨 클레임을 삭제하면 EBS 볼륨 또한 함께 삭제된다. (스토리지 클래스를 정의하는 yaml 파일에 **`reclaimPolicy` 으로 변경 가능)**
