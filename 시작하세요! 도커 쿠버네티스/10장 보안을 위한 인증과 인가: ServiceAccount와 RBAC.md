쿠버네티스는 보안 측면에서도 다향한 기능을 제공하고 있는, 그중에서 가장 자주 사용되는 것은 **RBAC(Role Based Access Control)를 기반으로 하는 서비스 어카운트(Service Account)라는 기능이다.**

서비스 어카운트는 사용자 또는 애플리케이션 하나에 해당하며, RBAC라는 기능을 통해 특정 명령을 실행할 수 있는 권한을 서비스 어카운트에 부여한다.  권한을 부여받은 서비스 어카운트는 해당 권한에 해당하는 기능만 사용할 수 있다.  (리눅스에서 root 유저와 일반 유저를 나누는 기능과 유사)

# 쿠버네티스의 권한 인증 과정

- 쿠버네티스는 설치할 때 설치 도구가 자동으로 kubectl이 관리자 권한을 갖도록 설정해둔다.
    - `~/.kube/config` 파일에서 확인 가능
- `kubectl` 을 사용할 때는 기본적으로 `~/.kube/config` 라는 파일에 저장된 설정을 읽어 쿠버네티스 클러스터를 제어
    - `client-certificate-data` 와 `client-key-data` 에 설정된 데이터는 base64로 인코딩된 인증서이다.  (이 키 쌍은 쿠버네티스에서 최고 권한(cluster-admin)을 갖는다

> ~/.kube/config 파일에서 확인 가능하다.
>

# 서비스 어카운트와 롤(Role), 클러스터 롤(Cluster Role)

- 서비스 어카운트는 체계적으로 권한을 관리하기 위한 쿠버네티스 오브젝트이다.
- 서비스  어카운트는 네임스페이스에 속하는 오브젝트이다.
- `--as`  옵션을 사용하면 임시로 특정 서비스 어카운트를 사용할 수 있다.
    - 서비스 어카운트를 사용 후 적절한 권한을 부여해야만 쿠버네티스의 기능을 제대로 사용할 수 있다.

```bash
kubectl get services --as system:serviceaccount:default:alicek106
```

[롤(Role)]

- 롤은 부여할 권한이 무엇인지를 나타내는 쿠버네티스 오브젝트이다.
- 롤은 네임스페이스에 속하는 오브젝트이다.

[클러스터 롤(Cluster Role)]

- 클러스터 롤은 말 그대로 클러스터 단위의 권한을 정의할 때 사용한다.
- 클러스터 롤은 네임스페이스에 속하지 않은 전역적인 쿠버네티스 오브젝트이다.

```yaml
# service-reader-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: service-reader
rules:
- apiGroups: [""]         # 대상이 될 오브젝트의 API 그룹
  resources: ["services"] # 대상이 될 오브젝트의 이름
  verbs: ["get", "list"]  # 어떠한 동작을 허용할 것인지 명시
```

- apiGroups:
    - 어떠한 API 그룹에 속하는 오브젝트에 대해 권한을 지정할지 설정
    - “” 로 설정되어 있으면, 이는 파드, 서비스 등이 포함된 코어 API 그룹을 의미한다.
- resources:
    - 어떠한 쿠버네티스 오브젝트에 대해 권한을 정의할 것인지 지정
- verbs:
    - 이 롤을 부여받은 대상이 resources에 지정된 오브젝트들에 대해 어떤 동작을 수행할 수 있는지 정의

롤은 특정 기능에 대한 권한만을 정의하는 오브젝트이기 때문에 롤을 생성하는 것만으로는 서비스 어카운트나 사용자에게 권한을 부여되지 않는다. 롤을 특정 대상에게 부여하려면 **롤 바인딩(RoleBinding)** 이라는 오브젝트를 통해 특정 대상과 롤을 연결해야 한다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: service-reader-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount # 권한을 부여할 대상이 ServiceAccount 이다.
  name: alicek106 # alicek106이라는 이름의 서비스 어카운트에 권한을 부여
  namespace: default
roleRef:
  kind: Role # Role에 정의된 권한을 부여
  name: service-reader # service-reader라는 이름의 Role을 대상(subjects)에 연결
  apiGroup: rbac.authorization.k8s.io
```

롤 바인딩에서는 어떠한 대상을 어떠한 롤에 연결할 것인지 정의한다.

> 롤 바인딩과 롤, 서비스 어카운트는 모두 1:1 관계가 아니다. 하나의 롤은 여러 개의 롤 바인딩에 의해 참조될 수 있고, 하나의 서비스 어카운트는 여러 개의 롤 바인딩에 의해 권한을 부여 받을 수 있다.
>

롤이나 클러스터 롤에서 사용되는 verbs 항목에는 get, list, watch, create, update, patch, delete 등에서 선택해 사용할 수 있지만, 와이드 카드를 의미하는 `*`를 사용할 수 있다. 단 특정 리소스에 한정된 기능을 사용할 때는 서브 리소스(sub resource)를 명시해야 할 수도 있다.

예를 들어 `kubectl exec` 명령어로 파드 내부에 들어가기 위한 권한을 생성하려면 파드의 하위 리소스인 `pod/exec`를 `resources` 항목에 정의해야 한다.

```yaml
...
- apiGroups: [""]
	resources: ["pod/exec"]
	verbs: ["create"]
...
```

롤을 사용할 때와 마찬가지로 클러스터 롤을 특정 대상에서 연결하려면 클러스터 롤 바인딩이라고 쿠버네티스 오브젝트를 사용해야 한다

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  namespace: default
  name: nodes-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nodes-reader-clusterrolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: alicek106
  namespace: default
roleRef:
  kind: ClusterRole
  name: nodes-reader
  apiGroup: rbac.authorization.k8s.io
```

# 여러 개의 클러스터 롤을 조합해서 사용하기

자주 사용되는 클러스터 롤이 있다면 다른 클러스터 롤에 포함시켜 재사용할 수 있는데, 이를 **클러스터 롤 애그리게이션(aggregation)** 이라고 한다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: parent-clusterrole
  labels:
    rbac.authorization.k8s.io/aggregate-to-child-clusterrole: "true"
rules: []
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: child-clusterrole
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-child-clusterrole: "true"
rules: [] # 어떠한 권한도 정의하지 않았습니다.
...
```

- 클러스터 롤에 포함시키고자 하는 다른 클러스터 롤을 `matchLabels의 라벨 셀렉터`로 선택하면 하위 클러스터 롤에 포함돼 있는 권한을 그대로 부여받을 수 있다.

# 쿠버네티스 API 서버에 접근

쿠버네티스의 API 서버는 HTTP 요청을 통해 쿠버네티스의 기능을 사용할 수 있도록 `REST API`를 제공한다.

쿠버네티스의 `REST API`에 접근하기 위한 엔드포인트는 자동으로 개방되기 때문에 별도의 설정을 하지 않아도 API 서버에 접근할 수 있다.

- 쿠버네티스 API 서버에 접근하려면 별도의 인증 정보를 HTTP 페이로드에 포함시켜 REST API 요청을 전송해야 한다.
- 쿠버네티스는 서비스 어카운트를 위한 인증 정보를 시크릿에 저장한다.
- 서비스 어카운트를 생성하면 이에 대응하는 시크릿이 자동으로 생성되며, 해당 시크릿은 서비스 어카운트를 증명하기 위한 수단으로 사용된다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: api-url-access
rules:
- nonResourceURLs: ["/metrics", "/logs"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: api-url-access-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: api-url-access
subjects:
- kind: ServiceAccount
  name: alicek106
  namespace: default
```

> API 서버의 몇몇 경로들은 기본적으로 서비스 어카운트가 접근할 수 없도록 제한돼 있다.
예: `/logs`, `/metrics` (권한이 없다는 오류 발생)
클러스터 롤을 사용하면 이러한 URL에도 접근할 수 있도록 권한을 부여할 수 있다.
>

# 클러스터 내부에서 kubernetes 서비스를 통해 API 서버에 접근

사용자가 쿠버네티스의 기능을 사용하려면 kuberctl이나 REST API 등의 방법을 통해 API 서버에 접근할 수 있다.

쿠버네티스는 클러스터 내부에서 API 서버에 접근할 수 있는 `kubernetes`라는 서비스 리소스를 미리 생성해 놓는다.

```java
$ kubectl get svc
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP  PORT(S)  AGE
kubernetes  ClusterIP   10.96.0.1    <none>       443/TCP  119d
```

쿠버네티스 내부에서 kubernetes라는 이름의 서비스에 접근한다고 해서 특별한 권한이 따로 주어지는 것은 아니다. 서비스 어카운트에 부여되는 시크릿의 토큰을 HTTP 요청에 담아 kubernetes 서비스에 전달해야만 인증과 인가를 진행할 수 있다.

쿠버네티스는 파드를 생성할 때 **자동으로 서비스 어카운트의 시크릿을 파드 내부에 마운트**한다.

파드를 생성하기 위한 YAML 스펙에 아무런 설정을 하지 않으면 자동으로 default 서비스 어카운트의 시크릿을 파드 내부에 마운트한다.

시크릿의 데이터는 기본적으로 파드의 `/var/run/secret/kubernetes.io/serviceaccount` 경로에 마운트 된다.

# 서비스 어카운트에 이미지 레지스트리 접근을 위한 설정

서비스 어카운트를 이용하면 비공개 레지스트리 접근을 위한 시크릿을 서비스 어카운트 자체에 설정할 수 있다.

즉, 디프로이먼트 파드의 YAML 파일마다 `docker-registry` 타입의 시크릿 이름을 정의하지 않아도 된다.

```yaml
# sa-reg-auth.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reg-auth-alicek106
  namespace: default
imagePullSecrets:
- name: registry-auth
```

파드를 생성하는 YAML 파일에서 `serviceAccountName` 항목에 reg-auth-alicek106 서비스 어카운트를 지정해 생성하면 자동으로 `imagaePullSecrets` 항목이 파드 스펙에 추가된다.

# kubeconfig 파일에 서비스 어카운트 인증 정보 설정

`kubectl` 명령어를 사용해 쿠버네티스 클러스터를 제어할 때는 `~~kubeconfig~~`라고 하는 특수한 설정 파일을 통해 인증을 진행한다.

쿠버네티스를 설치하면 `kubeconfig` 파일에는 기본적으로 클러스터 관리자 권한을 가지는 인증서 정보가 저장되며, 아무런 제한 없이 쿠버네티스를 사용할 수 있다.  하지만 여려 명의 개발자가 `kubectl` 명령어를 사용해야 한다면 서비스 어카운트를 이용해 적절한 권한 설정이 필요하다.

권한이 제한된 서비스 어카운트를 통해 `kubectl` 명령어를 사용하도록 `kubeconfig`에서 설정할 수 있다.

즉, 서비스 어카운트와 연결된 시크릿의 token 데이터를 `kubeconfig`에 명시함으로써 `kubectl` 명령어의 권한을 제한할 수 있다.

`kubeconfig` 파일은 일반적으로 `~/.kube/config` 경로에 있다.

`kubectl` 명령어로 쿠버네티스의 기능을 사용하면 `kubectl`은 기본적으로 `kubeconfig`의 설정 정보에서 API 서버의 주소의 주소와 사용자 인증 정보를 로드한다.

- `cluster`: `kubectl`이 사용할 쿠버네티스 API 서버의 접속 정보의 목록
- `users`: 쿠버네티스의 API 서버에 접속하기 위한 사용자 인증 정보 목록
- `context`: `cluster` 항목과 `users` 항목에 정의된 값을 조합해 최종적으로 사용할 쿠버네티스 클러스터의 정보를 설정

# 유저(User)와 그룹(Group)의 개념

쿠버네티스에선 서비스 어카운트 외에도 유저와 그룹이라는 개념이 있다.
- 유저: 실제 사용자
- 그룹: 여러 유저들을 모아 놓은 집합

쿠버네티스에는 유저나 그룹이라는 오브젝트가 없기 때문에 `kubectl get user` 또는 `kubectl get group`과 같은 명령어를 사용할 수 없다.

서비스 어카운트를 생성하면 system:serviceaccount:<네임스페이스 이름>:<서비스 어카운트 이름> 이라는 유저 이름으로 서비스 어카운트를 지칭할 수 있다.
따라서 서비스 어카운트에 권한을 부여하는 롤 바인팅을 생성할 때 아래와 같이 YAML 파일을 작성해 생성해도 서비스 어카운트 롤이 정상적으로 부여된다.

```yaml
....
subjects:
- kind: User
  name: system:serviceaccount:default:alicek106
  namespace: defatul
roleRef:
kind: Role
....
```

그룹은 이러한 유저를 모아 놓은 집합이다. 대표적인 예시는 system:serviceaccount로 시작하는 그룹이다. 이 그룹은 모든 네임스페이스에 속하는 모든 서비스 어카운트가 속해 있는 그룹이다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: service-reader-rolebinding
subjects:
- kind: Group
  name: system:serviceaccounts
roleRef:
  kind: ClusterRole  # 클러스터 롤 바인딩에서 연결할 권한은 클러스터 롤이여야 합니다.
  name: service-reader
  apiGroup: rbac.authorization.k8s.io
```

## 다양한 인증 방법에서의 User와 Group

쿠버네티스의 인증방법에는 서비스 어카운트의 시크릿 토큰만 존재하는 것은 아니다. 쿠버네티스에서 자체적으로 지원하는 인증 방법인 `x509 인증서` 이다. 그뿐만 아니라 인증 서버를 사용하면 깃허브 계정, 구글 계정, LDAP 데이터 등을 쿠버네티스 사용자 인증에 사용할 수 있다.
