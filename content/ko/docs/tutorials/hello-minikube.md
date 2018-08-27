---
title: Hello Minikube
content_template: templates/tutorial
weight: 5
---

{{% capture overview %}}

이 튜토리얼의 목적은 간단한 Node.js 로 작성된 Hello World 어플리케이션을 쿠버네티스에서 실행되는 어플리케이션으로 변환하는 것 입니다. 이 튜토리얼을 통해 로컬에서 작성된 코드를 Docker 컨테이너 이미지로 변환 한 다음, [Minikube](/docs/getting-started-guides/minikube) 에서 해당 이미지를 실행하는 방법을 보여줍니다. Minikube는 무료로 로컬 기기를 이용해서 쿠버네티스를 실행할 수 있는 간단한 방법을 제공합니다.

{{% /capture %}}

{{% capture objectives %}}

* Node.js 로 hello world 어플리케이션을 실행
* Minikube 에 만들어진 어플리케이션을 배포
* 어플리케이션 로그의 확인
* 어플리케이션 이미지 업데이트


{{% /capture %}}

{{% capture prerequisites %}}

* macOS 의 경우, [Homebrew](https://brew.sh) 를 사용하여 Minikube 를 설치할 수 있습니다.

  {{< note >}}
  **Note:** macOS 10.13 으로 업데이트 후 Homebrew 에서 `brew update` 를 실행 시 다음과 같은 오류가 발생할 경우:

  ```
  Error: /usr/local is not writable. You should change the ownership
  and permissions of /usr/local back to your user account:
  sudo chown -R $(whoami) /usr/local
  ```
  Homebrew 를 다시 설치하여 문제를 해결:
  ```
  /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
  ```
  {{< /note >}}

* 예제 어플리케이션을 실행하기 위해서는 [NodeJS](https://nodejs.org/en/) 가 필요합니다.

* Docker 를 설치하십시오. macOS 경우,
[Docker for Mac](https://docs.docker.com/engine/installation/mac/) 를 권장합니다.


{{% /capture %}}

{{% capture lessoncontent %}}

## Minikube 클러스터 만들기

이 튜토리얼에서는 [Minikube](https://github.com/kubernetes/minikube) 를 사용하여 로컬 클러스터를 만듭니다. 이 튜토리얼에서는 macOS에서 [Docker for Mac](https://docs.docker.com/engine/installation/mac/) 을 사용한다고 가정하였습니다. Docker for Mac 대신 Linux 혹은 VirtualBox 와 같이 다른 플랫폼을 사용하는 경우, Minikube 를 설치하는 방법이 약간 다를 수 있습니다. 일반적인 Minikube 설치 지침은 [Minikube installation guide](/docs/getting-started-guides/minikube/) 를 참조하십시오.

Homebrew를 사용하여 최신 Minikube 버전을 설치:
```shell
brew cask install minikube
```

[Minikube driver installation guide](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md#hyperkit-driver) 에 설명한 것과 같이 HyperKit 드라이버를 설치하십시오.

Homebrew를 사용하여 쿠버네티스 클러스터와 상호 작용을 위한 `kubectl` 명령줄 도구 다운로드:

```shell
brew install kubernetes-cli
```

프록시를 거치지않고 직접 https://cloud.google.com/container-registry/ 같은 사이트에 액세스 할 수 있는지 확인하려면 새 터미널을 열고 다음과 같이 실행하십시오.

```shell
curl --proxy "" https://cloud.google.com/container-registry/
```

Docker 데몬이 시작되었는지 확인하십시오. Docker 가 실행 중인지는 다음과 같은 커맨드를 사용하여 확인:

```shell
docker images
```

프록시가 필요하지 않은 경우, Minikube 클러스터를 시작:

```shell
minikube start --vm-driver=hyperkit
```
프록시가 필요한 경우, 다음 방법을 사용하여 프록시 설정으로 Minikube 클러스터를 시작:

```shell
minikube start --vm-driver=hyperkit --docker-env HTTP_PROXY=http://your-http-proxy-host:your-http-proxy-port  --docker-env HTTPS_PROXY=http(s)://your-https-proxy-host:your-https-proxy-port
```

명시된 `--vm-driver = hyperkit` 플래그는 Docker for Mac 을 사용하고 있음을 의미합니다.
기본 VM 드라이버는 VirtualBox입니다.

이제 Minikube 컨텍스트를 설정하십시오. 컨텍스트는 `kubectl` 과 상호 작용하는 클러스터를 결정합니다. `~/.kube/config` 파일에서 사용 가능한 모든 컨텍스트를 볼 수 있습니다.

```shell
kubectl config use-context minikube
```

`kubectl` 이 클러스터와 통신할 수 있도록 설정되어 있는지 확인:

```shell
kubectl cluster-info
```

브라우저에서 쿠버네티스 대시보드 열기:

```shell
minikube dashboard
```

## Node.js 어플리케이션 만들기


다음 단계는 어플리케이션을 작성해봅니다. 아래 코드를 `hellonode` 폴더에 `server.js` 라는 이름으로 저장:

{{< codenew language="js" file="minikube/server.js" >}}

작성된 어플리케이션 실행:

```shell
node server.js
```

http://localhost:8080/ 에 접속하여 "Hello World!" 라는 메세지를 확인 할 수 있습니다.

Node.js 서버를 중단하려면 **Ctrl-C** 를 입력합니다.

다음 단계는 작성된 어플리케이션을 Docker 컨테이너로 변환해봅니다.

## Docker 컨테이너 이미지 만들기

앞에서 사용하였던 `hellonode` 폴더에  `Dockerfile` 이라는 이름으로 파일을 만듭니다. Dockerfile 은 작성하고자하는 이미지에 대한 구성요소를 미리 설정해놓은 파일입니다. 기존 이미지를 확장하여 Docker 컨테이너 이미지를 작성할 수 있습니다. 이 튜토리얼에서는 기존 Node.js 이미지를 확장하여 사용합니다.


{{< codenew language="conf" file="minikube/Dockerfile" >}}

Docker 레지스트리에 있는 공식 Node.js LTS 이미지를 사용해서, 8080 포트를 열고, `server.js` 파일을 이미지에 복사하고 Node.js 서버를 시작하도록 구성하였습니다.

이 튜토리얼은 Minikube를 사용하기 때문에, Docker 이미지를 레지스트리로 Push 하는 대신, Minikube VM과 같은 Docker 호스트를 사용하여 간단하게 이미지를 작성하고, 자동으로 목록에 저장되도록 변경 할 수 있습니다. 변경하려면 Minikube Docker 데몬을 확인 후, 아래와 같이 실행:

```shell
eval $(minikube docker-env)
```

{{< note >}}
**Note:** 나중에 Minikube 호스트를 사용하지 않을 경우, `eval $ (minikube docker-env -u)` 를 실행하여 변경을 취소 할 수 있습니다.
{{< /note >}}


Minikube Docker 데몬을 사용하여 Docker 이미지를 작성 (마지막의 점에 주의):

```shell
docker build -t hello-node:v1 .
```

이제 Minikube VM은 작성한 이미지를 실행 할 수 있습니다.

## 디플로이먼트 만들기

쿠버네티스 [*파드*](/docs/concepts/workloads/pods/pod/)는 관리 및 네트워크 구성을 목적으로 함께 묶은 하나 이상의 컨테이너 그룹입니다. 이 튜토리얼의 파드에는 단 하나의 컨테이너만 있습니다. 쿠버네티스 [*디플로이먼트*](/docs/concepts/workloads/controllers/deployment/) 는 포드의 상태를 검사하고 종료되는 일이 발생하면 파드 컨테이너를 다시 시작합니다. 파드의 생성 및 확장을 관리에 디플로이먼트를 권장합니다.

`kubectl run` 커맨드를 사용하여 파드를 관리하는 디플로이먼트를 만드십시오. 파드는 `hello-node:v1` Docker 이미지로 만들어진 컨테이너를 실행 합니다. Docker 레지스트리에서 가져오지 않고 로컬이미지를 사용하려면 `--image-pull-policy` 플래그를 `Never` 로 설정 (레지스트리에 Push 하지 않았기 때문에 로컬 이미지를 사용해야 함):

```shell
kubectl run hello-node --image=hello-node:v1 --port=8080 --image-pull-policy=Never
```

디플로이먼트를 확인:


```shell
kubectl get deployments
```

출력:


```shell
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-node   1         1         1            1           3m
```

파드를 확인:


```shell
kubectl get pods
```

출력:


```shell
NAME                         READY     STATUS    RESTARTS   AGE
hello-node-714049816-ztzrb   1/1       Running   0          6m
```

클러스터 이벤트를 확인:

```shell
kubectl get events
```

`kubectl` 의 설정을 확인:

```shell
kubectl config view
```

`kubectl` 커맨드에 대한 더 많은 정보를 원하는 경우,
[kubectl overview](/docs/user-guide/kubectl-overview/) 를 확인하세요.

## 서비스 만들기

기본적으로 파드는 쿠버네티스 클러스터 내의 내부 IP 주소로만 접속가능합니다. 쿠버네티스 가상 네트워크 밖에서 `hello-node` 컨테이너에 접속하기 위해서는 파드를 쿠버네티스 [*서비스*](/docs/concepts/services-networking/service/) 로 노출하여야 합니다.

디플로이먼트에서 `kubectl expose` 커맨드를 통해 파드를 퍼블릭 인터넷에 노출:

```shell
kubectl expose deployment hello-node --type=LoadBalancer
```

방금 생성한 서비스 확인:

```shell
kubectl get services
```

출력:

```shell
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
hello-node   ClusterIP   10.0.0.71    <pending>     8080/TCP   6m
kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP    14d
```

`--type=LoadBalancer` 플래그는 해당 서비스를 클러스터 외에 노출 시키는 것을 의미합니다. 로드 밸런서를 지원하는 클라우드 제공 업체의 경우, 외부 IP 주소가 서비스에 액세스 할 수 있도록 제공됩니다. Minikube 에서 `LoadBalancer` 타입은 `minikube service` 커맨드를 통해 서비스에 접근 가능하도록 합니다.

```shell
minikube service hello-node
```

로컬 IP 주소를 이용하여 App을 제공하고 "Hello World" 메세지를 보여주는 브라우저 윈도우를 자동으로 열어줍니다.

브라우저 또는 curl 을 통해 새 웹서비스에 리퀘스트를 보냈을 때, 로그가 쌓이는 것을 확인:

```shell
kubectl logs <POD-NAME>
```

## App 업데이트

`server.js` 파일을 새로운 메세지를 출력하도록 수정:

```javascript
response.end('Hello World Again!');

```

새로운 버전의 이미지를 작성 (마지막의 점에 주의):

```shell
docker build -t hello-node:v2 .
```

디플로이먼트의 이미지를 업데이트:

```shell
kubectl set image deployment/hello-node hello-node=hello-node:v2
```

App 을 다시 실행하여 새로운 메세지를 확인:

```shell
minikube service hello-node
```

## 애드온 활성화하기

Minikube 에는 로컬 쿠버네티스 환경에서 활성화 / 비활성화 할 수 있거나 이미 열려 있는 내장 애드온 셋이 있습니다.

최초의 현재 지원되는 애드온 목록을 확인:

```shell
minikube addons list
```

출력:

```shell
- storage-provisioner: enabled
- kube-dns: enabled
- registry: disabled
- registry-creds: disabled
- addon-manager: enabled
- dashboard: disabled
- default-storageclass: enabled
- coredns: disabled
- heapster: disabled
- efk: disabled
- ingress: disabled
```

커맨드를 적용하기 위해서는 Minikube 가 실행 중이어야 합니다. 예시로, `heapster` 애드온을 활성화:

```shell
minikube addons enable heapster
```

출력:

```shell
heapster was successfully enabled
```

파드와 생성된 서비스 확인:

```shell
kubectl get po,svc -n kube-system
```

출력:

```shell
NAME                             READY     STATUS    RESTARTS   AGE
pod/heapster-zbwzv                1/1       Running   0          2m
pod/influxdb-grafana-gtht9        2/2       Running   0          2m

NAME                           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
service/heapster               NodePort    10.0.0.52    <none>        80:31655/TCP        2m
service/monitoring-grafana     NodePort    10.0.0.33    <none>        80:30002/TCP        2m
service/monitoring-influxdb    ClusterIP   10.0.0.43    <none>        8083/TCP,8086/TCP   2m
```

브라우저에서 엔드포인트를 열어 heapster 와 상호작용:

```shell
minikube addons open heapster
```

출력:

```shell
Opening kubernetes service kube-system/monitoring-grafana in default browser...
```

## 제거하기

이제 클러스터에서 만들어진 리소스들을 제거:

```shell
kubectl delete service hello-node
kubectl delete deployment hello-node
```

상황에 따라, 생성된 Docker 이미지를 강제로 제거:

```shell
docker rmi hello-node:v1 hello-node:v2 -f
```

상황에 따라, Minikube VM 을 정지:

```shell
minikube stop
eval $(minikube docker-env -u)
```

상황에 따라, Minikube VM 을 삭제:

```shell
minikube delete
```

{{% /capture %}}


{{% capture whatsnext %}}

* [Deployment objects](/docs/concepts/workloads/controllers/deployment/) 에 대해서 더 배워봅니다.
* [Deploying applications](/docs/user-guide/deploying-applications/) 에 대해서 더 배워봅니다.
* [Service objects](/docs/concepts/services-networking/service/) 에 대해서 더 배워봅니다.

{{% /capture %}}
