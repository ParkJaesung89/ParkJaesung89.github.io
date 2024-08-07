---
title: EKS 환경에서 Prometheus+Grafana+Thanos 구성(수정중)
date: 2024-03-25 12:45:00 +09:00
categories: [AWS, EKS]
tags: [EKS, Prometheus] # TAG names should always be lowercase
---

# EKS for terraform & prometheus+grafana(feet. helm) stack

## 구성도

## Create EKS for terraform

## prometheus+grafana(feet. helm) stack 구성을 위한 사전 client 필수 구성

1. aws cli 설치

```bash
$curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$unzip awscliv2.zip
$sudo ./aws/install
```

2. aws configure 설정

```bash
$aws configure --profile {profile_name}
AWS Access Key ID : {access key}
AWS Secret Access Key : {secret access key}
```

3. awseks 설치

```bash
$curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
$tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
$sudo mv /tmp/eksctl /usr/local/bin
```

4. kubectl 설치

```bash
$curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl
$chmod +x ./kubectl
$mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH

$kubectl get nodes    # cluster 정보를 잘 받아오는지 체크
```

5. Node Roles 변경
   신규 가입되어있는 worker node의 경우 ROLES확인 시 "<none>" 으로 되어있는 것을 확인할 수 있다.
   변경하지 않아도 현재 프로젝트에서는 문제되지 않겠지만, 여러 노드들이 추가되는 경우에 스케줄링 시 원하는 node에 pod를 할당 할 수 없을 것입니다. 그렇기에 기본적으로 ROLES를 설정 하였습니다.

```bash
kubectl label node {node_name} node-role.kubernetes.io/worker=
# ROLES를 worker로 변경
# 각 worker 노드들마다 동일하게 설정
```

<!--
6. Kubernetes Port range 변경
   기본 쿠버네티스의 port range는 30000 - 32767이지만, 서비스 운영시에 포트가 곂칠 경우가 있습니다.
   그럴경우 port range를 변경할 수 있습니다.

```bash
vi /etc/kubernetes/manifests/kube-apiserver.yaml
#command 부분 아래에 내용 추가
   - --service-node-port-range=9000~32767
# 저장 후 나가면 자동 적용(/etc/kubernetes/manifests 경로는 static pod이기 때문에 kubelet에 의해서 자동 적용)
```
-->

6. helm 설치

   ### Helm이란?

   The package manager for Kubernetes로 kubernetes의 관리를 좀더 쉽게 하기위한 툴이라고 보면 됩니다.
   Helm의 가장 중요한 요소인 Helm chart는 kubernetes cluster의 리소스를 기술하고 이를 하나의 애플리케이션으로 패키징하는 파일 컬렉션이며, 쉽게 말해서 RedHat의 yum과 비슷한 기능을 수행하는 역할이라고 보면 됩니다.

   ### Helm chart의 구성요소

   1. Chart.yaml - 이름, 버전, 종속성 등 애플리케이션 메타데이터를 정의합니다.
   2. values.yaml - 옵션들의 값을 설정합니다.
   3. templates 디렉토리
      templates/ - 템플릿을 보관하며, values.yaml 파일에 성정되어 있는 값과 결합하여 매니페스트를 생성합니다.
      charts/ - 수동으로 관리하는 모든 차트의 종속성을 보관합니다.

   ### Helm 배포 패키지의 구성

   1. charts: Helm 차트의 종속 차트를 포함하는 위치입니다. 현재 사용할 패키지의 경우 grafana, kube-state-metrics, prometheus-node-exporter가 존재합니다.
   2. templates: Helm 차트의 템플릿 파일들을 포함합니다. 템플릿은 kubernetes 리소스의 정의를 작성하는데 사용되며, 이를 통해 애플리케이션의 배포, 서비스, 구성 등을 관리할 수 있습니다.
   3. crds: Custom Resource Definitions(CRDs)파일을 포함할 수 있는 위치입니다. CRD는 kubernetes API에 사용자 정의 리소스와 그에 대한 스키마를 추가하는데 사용됩니다.
   4. Chart.yaml: Helm 차트의 메타 정보를 정의합니다. 메타 정보에는 차트의 이름, 버전, 유형, 유지보수자 정보 등이 포함됩니다. 또한 종속 차트, 애플리케이션의 버전 제약 조건 등을 지정할 수도 있습니다.
   5. values.yaml: Helm 차트의 기본 구성 값을 정의합니다. 애플리케이션의 설정 옵션, 환경 변수, 리소스의 크기 등을 설정 할 수 있습니다. 해당 파일에 정의 된 값은 템플릿 파일 내에서 사용 될 수 있으며, 차트를 배포할 때 사용자지정 값으로 오버라이드 할 수도 있습니다.

   - CRDs, templates 디렉토리의 파일들을 수정할 일은 거의 없으며, 주로 values.yaml 을 구성하고자 할 경우 환경에 맞추어 수정하고 추가로 종속 차트의 세부 설정들도 수정해야 되는 경우에는 charts 디렉토리 내의 종속 차트에서 수정합니다.

```bash
$curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
$chmod 700 get_helm.sh
$./get_helm.sh
```

## helm chart

```bash
$helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$helm repo update

$helm search repo prometheus-community
NAME                                                 CHART VERSION APP VERSION DESCRIPTION
prometheus-community/alertmanager                    1.10.0        v0.27.0     The Alertmanager handles alerts sent by client ...
prometheus-community/alertmanager-snmp-notifier      0.3.0         v1.5.0      The SNMP Notifier handles alerts coming from Pr...
prometheus-community/jiralert                        1.7.0         v1.3.0      A Helm chart for Kubernetes to install jiralert
prometheus-community/kube-prometheus-stack           57.1.1        v0.72.0     kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-state-metrics              5.18.0        2.11.0      Install kube-state-metrics to generate and expo...
prometheus-community/prom-label-proxy                0.7.1         v0.8.1      A proxy that enforces a given label in a given ...
prometheus-community/prometheus                      25.18.0       v2.51.0     Prometheus is a monitoring system and time seri...
prometheus-community/prometheus-adapter              4.9.1         v0.11.2     A Helm chart for k8s prometheus adapter
prometheus-community/prometheus-blackbox-exporter    8.12.0        v0.24.0     Prometheus Blackbox Exporter
prometheus-community/prometheus-cloudwatch-expo...   0.25.3        0.15.5      A Helm chart for prometheus cloudwatch-exporter
prometheus-community/prometheus-conntrack-stats...   0.5.10        v0.4.18     A Helm chart for conntrack-stats-exporter
prometheus-community/prometheus-consul-exporter      1.0.0         0.4.0       A Helm chart for the Prometheus Consul Exporter
prometheus-community/prometheus-couchdb-exporter     1.0.0         1.0         A Helm chart to export the metrics from couchdb...
prometheus-community/prometheus-druid-exporter       1.1.0         v0.11.0     Druid exporter to monitor druid metrics with Pr...
prometheus-community/prometheus-elasticsearch-e...   5.6.0         v1.7.0      Elasticsearch stats exporter for Prometheus
prometheus-community/prometheus-fastly-exporter      0.3.0         v7.6.1      A Helm chart for the Prometheus Fastly Exporter
prometheus-community/prometheus-ipmi-exporter        0.3.0         v1.8.0      This is an IPMI exporter for Prometheus.
prometheus-community/prometheus-json-exporter        0.11.0        v0.6.0      Install prometheus-json-exporter
prometheus-community/prometheus-kafka-exporter       2.9.0         v1.7.0      A Helm chart to export the metrics from Kafka i...
prometheus-community/prometheus-memcached-exporter   0.3.1         v0.14.2     Prometheus exporter for Memcached metrics
prometheus-community/prometheus-modbus-exporter      0.1.0         0.4.0       A Helm chart for prometheus-modbus-exporter
prometheus-community/prometheus-mongodb-exporter     3.5.0         0.40.0      A Prometheus exporter for MongoDB metrics
prometheus-community/prometheus-mysql-exporter       2.5.1         v0.15.1     A Helm chart for prometheus mysql exporter with...
prometheus-community/prometheus-nats-exporter        2.16.0        0.14.0      A Helm chart for prometheus-nats-exporter
prometheus-community/prometheus-nginx-exporter       0.2.1         0.11.0      A Helm chart for the Prometheus NGINX Exporter
prometheus-community/prometheus-node-exporter        4.32.0        1.7.0       A Helm chart for prometheus node-exporter
prometheus-community/prometheus-opencost-exporter    0.1.1         1.108.0     Prometheus OpenCost Exporter
prometheus-community/prometheus-operator             9.3.2         0.38.1      DEPRECATED - This chart will be renamed. See ht...
prometheus-community/prometheus-operator-admiss...   0.11.0        0.72.0      Prometheus Operator Admission Webhook
prometheus-community/prometheus-operator-crds        10.0.0        v0.72.0     A Helm chart that collects custom resource defi...
prometheus-community/prometheus-pgbouncer-exporter   0.1.1         1.18.0      A Helm chart for prometheus pgbouncer-exporter
prometheus-community/prometheus-pingdom-exporter     2.5.0         20190610-1  A Helm chart for Prometheus Pingdom Exporter
prometheus-community/prometheus-pingmesh-exporter    0.4.0         v1.2.1      Prometheus Pingmesh Exporter
prometheus-community/prometheus-postgres-exporter    6.0.0         v0.15.0     A Helm chart for prometheus postgres-exporter
prometheus-community/prometheus-pushgateway          2.8.0         v1.7.0      A Helm chart for prometheus pushgateway
prometheus-community/prometheus-rabbitmq-exporter    1.11.0        v0.29.0     Rabbitmq metrics exporter for prometheus
prometheus-community/prometheus-redis-exporter       6.2.0         v1.58.0     Prometheus exporter for Redis metrics
prometheus-community/prometheus-smartctl-exporter    0.7.1         v0.11.0     A Helm chart for Kubernetes
prometheus-community/prometheus-snmp-exporter        5.1.0         v0.25.0     Prometheus SNMP Exporter
prometheus-community/prometheus-stackdriver-exp...   4.5.0         v0.15.0     Stackdriver exporter for Prometheus
prometheus-community/prometheus-statsd-exporter      0.13.0        v0.26.0     A Helm chart for prometheus stats-exporter
prometheus-community/prometheus-systemd-exporter     0.2.0         0.6.0       A Helm chart for prometheus systemd-exporter
prometheus-community/prometheus-to-sd                0.4.2         0.5.2       Scrape metrics stored in prometheus format and ...
prometheus-community/prometheus-windows-exporter     0.3.1         0.25.1      A Helm chart for prometheus windows-exporter
```

## Namespace 생성

격리된 공간에서 운영 및 배포를 하며, 다른 오브젝트와의 충돌을 방지하기 위하여 namespace를 생성합니다.
prometheus-grafana 프로젝트는 해당 monitoring namespace에 구성 예정입니다.

```bash
$kubectl create ns monitoring
```

## Worker Node의 Label 지정

```bash
$kubectl label node {node_name} nodePool={label_name}
#예) kubectl label node ip-10-0-1-246.ap-northeast-2.compute.internal nodePool=mon_cluster
#각 node에 적용하기
```

## Prometheus-operator git clone

helm pull 명령을 통하여 kube-prometheus-stack 패키지를 받아옵니다.
직접 필요한 구성을 yaml에 작성하여 사용할 수 있으나, 2가지 이유로 전체 소스 clone하여 사용합니다.
첫번째, 유사시 default 기반으로의 롤백
두번째, 전체적인 소스를 파악하기 위함입니다.

```bash
$helm pull prometheus-community/kube-prometheus-stack
```

values.yaml 의 설정을 수정하여 배포하기 위하여 values_new.yaml로 복사 한 후 안에 내용을 수정하겠습니다.

```bash
cp /{helm-charts_folder_path}/helm-charts/charts/kube-prometheus-stack/values.yaml /{helm-charts_folder_path}/helm-charts/charts/kube-prometheus-stack/values_new.yaml
```

### values_new.yaml 수정해야 되거나 고려할 설정 값(멀티클러스터시 필요한 설정들)

1. grafana - enabled
   grafana는 prometheus의 시계열 데티어를 활용하여 대시보드를 구성할 수 있는 모니터링 UI 오픈소스 툴로, Prometheus의 데이터를 모니터링하기 위하여 사용합니다.

```bash
grafana:
  enabled: true
  namespaceOverride: ""
```

<!--
2. grafana - DataSource
   grafana에서 지원하는 여러가지 오픈소스 스토리지들을 지정할 수 있습니다. grafana 실행 후 웹에서 콘솔 설정이 가능하나 grafana 재배포를 해야될 경우 저장된 DataSource 정보가 초기화 되기 때문에 매번 설정하지 않기 위하여 해당 옵션을 설정합니다.

```bash

```
-->

2. grafana - service type
   grafana는 웹에서 사용하기 위한 opensource 이기에 ClusterIP 타입보다는 NodePort나 LoadBalancer 타입으로 사용하는 것이 일반적입니다.

- service?
  파드들을 통해 실행되고 있는 애플리케이션을 네트워크에 노출(expose) 시키는 것을 도와주는 가상의 컴포넌트입니다.
  쿠버네티스에서의 파드는 어떤 서비스를 구동중인 상태로 유지하기 위한 일회성 자원일 뿐이며, 언제든 다른 노드로 옮겨지고 삭제될 수 있기 때문에 파드가 외부와 통신하기 위해서는 Static IP를 부여해줄 필요가 있습니다.
- service type?
  위와같은 이유로 여러 서비스 목적별로 service type이 나뉘어 있습니다.
  ClusterIP (기본 형태)
  NodePort
  LoadBalancer
  ExternalName
  위내용에 대한 자세한 설명 => [링크](https://seongjin.me/kubernetes-service-types/)

```bash
service:
  type: NodePort
  nodePort: 31000
  portName: http-web # 기본값

# OR

service:
  type: LoadBalancer
  portName: http-web
```

3. grafana - dashboard
   grafana 재 배포시 대시보드구성 했던 것들이 사라지기 때문에 서비스 운영시에 만들어두었던 대시보드를 json 파일로 추출하여 helm 서브 차트에 있는 grafana 디렉토리 내부에 저장하여 재배포 후 대시보드를 불러올 수 있습니다.

```bash
vi ./charts/grafana/values.yaml

dashboards:
  pod-dashboard:
    file: dashboards/PodDashboard.json
  mysql-dashboard:
    file: dashboards/MysqlDashboard.json
  elastic-dashboard:
    file: dashboards/ElasticsearchDashboard.json
# 대시보드의 이름은 직접 네이밍할 수 있으며, file 설정에는 실제 json 파일의 경로를 작성하면 됩니다.
```

3. Prometheus - remoteWrite(thanos sidecar 방식이 아닐경우)
   remoteWrite 옵션은 Prometheus에 저장되는 시계열 데이터들을 다른 스토리지에 저장할 수 있는 옵션입니다.
   멀티 클러스터 구성시 여러개의 prometheus가 실행될 경우 HA(고가용성)을 위해 시계열데이터들을 Prometheus가 아닌 외부 스토리지에 저장하게 되는데 가장 많이 쓰이는 것이 Thanos 라는 오픈소스 스토리지 입니다.
   Thanos와의 연계는 다음 포스팅에 설명하겠습니다.

```bash
remoteWrite:
- url: http://thanos-receive.monitoring.SVC:19291/api/v1/receive
```

4. Prometheus - podMonitor, serviceMonitor
   노드의 리소스 정보를 Prometheus로 전송하는 node-exporter외에도 Pod, Service의 상태정보를 수집하기 위해 Prometheus는 PodMonitor, ServiceMonitor와 연결되어야 됩니다.
   모니터링 데이터를 수집하기 위해선 아래 두 옵션을 false로 변경해야 됩니다.

```bash
podMonitorSelectorNilUsesHelmValues: false
serviceMonitorSelectorNilUsesHelmValues: false
# 두 옵션을 true로 할 경우 helm 배포시에 prometheus-operator와 동일한 릴리즈 태그로 레이블이 지정된 PodMonitors, ServiceMonitors만 검색되기 때문에 필터링 되지 않고 모든 네임스페이스를 검색하기 위해 false로 설정합니다.
```

### values.yaml의 기본적인 test용으로 수정 한 값

```bash
grafana:
  enabled: true
prometheus:
  enabled: true
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    replicas: 1
    shards: 1
    retention: 30d
    retentionSize: "20GiB"
    scrapeInterval: 15s
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes:
          - ReadWriteOnce
          resources:
            requests:
              storage: 30Gi
          storageClassName: gp2
```

## helm prometheus 배포

```bash
$helm upgrade -i {release_name} -f {사용할_values.yaml} prometheus-community/kube-prometheus-stack --namespace {namespace_name}

# 예) helm upgrade -i prometheus -f prometheus-values.yaml prometheus-community/kube-prometheus-stack --namespace monitoring

$kubectl get pods --namespace monitoring
NAME                                             READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-alertmanager-0           2/2     Running   0          8m51s
prometheus-grafana-86b6d8f896-q5fw4              3/3     Running   0          8m58s
prometheus-kube-state-metrics-59b5d58f8f-rpqr2   1/1     Running   0          8m57s
prometheus-operator-74789b669d-rmqng             1/1     Running   0          8m57s
prometheus-prometheus-node-exporter-8cwk8        1/1     Running   0          8m58s
prometheus-prometheus-node-exporter-bxjjp        1/1     Running   0          8m58s
prometheus-prometheus-prometheus-0               2/2     Running   0          72s
```

### prometheus & grafana 포탈 접속 테스트

prometheus 포탈 접속

```bash
$kubectl port-forward --address=0.0.0.0 -n monitoring prometheus-prometheus-prometheus-0 8080:9090
```

pod와 node의 갯수 쿼리를 이용해서 매트릭을 잘 받아오는지 확인해보자.
count(kube_pod_info)
count(kube_node_info)

grafana 포탈 접속

```bash
# grafana user와 password 확인
$kubectl get secrets -n monitoring prometheus-grafana -ojsonpath="{.data.admin-user}" | base64 -d
$kubectl get secrets -n monitoring prometheus-grafana -ojsonpath="{.data.admin-password}" | base64 -d

# grafana 포트 연결
$kubectl port-forward --address=0.0.0.0 -n monitoring svc/prometheus-grafana 8080:80
```

grafana 그래프 확인

# HA 고가용성 구성(with Thanos)

Prometheus는 kubernetes에서 가장 인기가 좋은 오픈소스 모니터링 툴이지만, 크나큰 단점이 존재합니다.

1. 확장 및 고가용성
   - Prometheus는 단일 서버로 동작하도록 구현되어 있기에 추가적인 작업(샤딩, federation 등)들이 필요합니다.
2. 오래된 데이터의 보관 문제
   - 메트릭을 로컬 디스크에 보관, 수집하는데, 저장소의 용량이 한계에 도달하면 오래된 데이터가 자동으로 삭제되어 일정 시간이 지난 데이터는 조회가 불가능 합니다.

이러한 단점들을 보완하기 위해서 Thanos라는 오픈소스 툴을 사용합니다.

## Thanos란?

Prometheus의 sidecar를 통해서 데이터를 수집하고 중앙 집중화 된 view를 제공하면서 aws와 s3 같은 object storage를 활용하여 오래된 데이터를 보관함으로써 보관 문제를 해결합니다. 또한 중복된 label을 제거하는 중복 제거 기능이 존재합니다.

### Thanos의 주요 컴포넌트

- Sidecar : prometheus에 sidecar 패턴으로 생성되어

- Query

- Query-frontend:

- Compactor:

- Store gateway:

- Ruler:

## prometheus-1 설정

### thanos helm chart add

```bash
$helm repo add bitnami https://charts.bitnami.com/bitnami
```

### objstore.yml 파일 생성

```bash
type: S3
config:
  bucket: "thanos-s3-logs"
  endpoint: "s3.ap-northeast-2.amazonaws.com"
  region: "ap-northeast-2"
  access_key: {access_key}
  secret_key: {secret_key]
  trace:
    enable: true
```

위파일을 이용해서 secret을 생성합니다.

```bash
$kubectl create secret generic s3-secret --from-file=objstore.yml -n monitoring
$kubectl get secret -n monitoring | grep s3             # 확인
s3-secret                                           Opaque               1      15m
```

### thanos sidecar 배포

thanos는 2가지 방법으로 서비스를 제공할 수 있습니다.

1. sidecar
2. thanos 서버

이 글에서는 가장 많이 사용되는 sidecar 방식으로 진행하습니다.

```bash
prometheus:
  thanosService:
    enabled: true
  prometheusSpec:
    externalLabels:
      cluster: prometheus-1     #prometheus-2
    thanos:
      objectStorageConfig:
        key: objstore.yml       # s3 secret 포함
        name: s3-secret
      version: v0.34.1


$helm upgrade -i prometheus prometheus-community/kube-prometheus-stack -f /{Path}/kube-prometheus-stack/prometheus-values.yaml -f /{Path}/thanos/prometheus-thanos-values.yaml -n monitoring

Release "prometheus" has been upgraded. Happy Helming!
NAME: prometheus
LAST DEPLOYED: Thu Mar 28 08:54:53 2024
NAMESPACE: monitoring
STATUS: deployed
REVISION: 6
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=prometheus"

# 예)
#$helm upgrade -i prometheus prometheus-community/kube-prometheus-stack -f /usr/local/src/kube-prometheus-stack/prometheus-values.yaml -f /usr/local/src/thanos/prometheus-thanos-values.yaml -n monitoring
```

배포 완료 후, 상태를 확인해 보면 아래와 같이 컨테이너가 2개였다가 3개로 증가 한 것을 확인 할 수 있습니다.

```bash
kubectl get pod -n monitoring

```

해당 pod안에 실제로 sidecar가 생성되어있는지 확인해보겠습니다.

```bash
# pod안의 container들의 네임을 확인 할 수 있습니다.
$kubectl get pod prometheus-prometheus-prometheus-0 -n monitoring -o=jsonpath='{.spec.containers[*].name}' | tr ' ' '\n\n'
prometheus
config-reloader
thanos-sidecar

# pod의 설정 중 sidecar 의 정보 입니다.
$kubectl get pod -n monitoring prometheus-prometheus-prometheus-0 -o yaml
~~~
- args:
    - sidecar
    - --prometheus.url=http://127.0.0.1:9090/
    - '--prometheus.http-client={"tls_config": {"insecure_skip_verify":true}}'
    - --grpc-address=:10901
    - --http-address=:10902
    - --log.level=info
    - --log.format=logfmt
    image: quay.io/thanos/thanos:v0.34.1
    imagePullPolicy: IfNotPresent
    name: thanos-sidecar
    ports:
    - containerPort: 10902
      name: http
      protocol: TCP
    - containerPort: 10901
      name: grpc
      protocol: TCP
    resources: {}
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: FallbackToLogsOnError
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-nqrt6
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  hostname: prometheus-prometheus-prometheus-0
~~~

# thanos의 서비스도 생성된 것을 확인 할 수 있습니다.
$kubectl get svc -n monitoring | grep thanos
prometheus-thanos-discovery           ClusterIP   None             <none>        10901/TCP,10902/TCP          43m
```

## prometheus-2 설정

우선 prometheus-1과 동일하게 구성진행하기 전에 cluster 연동 및 다중 클러스터 관리를 위한 방법을 알아봅니다.

```bash
# kubeconfig에 Cluster 추가합니다.
$aws eks update-kubeconfig --region ap-northeast-2 --name {cluster_name} --profile {profile_name}
# 예) aws eks update-kubeconfig --region ap-northeast-2 --name jsp-eks2 --profile jaesung.park

# 추가후에 cluster 등록 확인합니다.
$cat /root/.kube/config | grep cluster:
    cluster: arn:aws:eks:ap-northeast-2:{account}:cluster/jsp-eks
    cluster: arn:aws:eks:ap-northeast-2:{account}:cluster/jsp-eks2

# 현재 연결되있는 cluster 확인
$kubectl config current-context

# content 변경으로 cluster 세션을 변경
$kubectl config use-context arn:aws:eks:ap-northeast-2:{account}:cluster/jsp-eks2
```

prometheus-2 클러스터에도 prometheus-1과 동일하게 pod를 올릴 것입니다.
prometheus-1과 차이점은 cluster 명을 구분할 수 있도록 prometheus-2 로 변경하는 것 외에 없습니다.

```bash
cp /{path}/thanos/prometheus-thanos-values.yaml /{path}/thanos/prometheus-thanos-values2.yaml
vi /{path}/thanos/prometheus-thanos-values2.yaml

prometheus:
  thanosService:
    enabled: true
  prometheusSpec:
    externalLabels:
      cluster: prometheus-2         ## prometheus-1 cluster와의 구분을 위해서 이름 변경
    thanos:
      objectStorageConfig:
        key: objstore.yml
        name: s3-secret
      version: v0.34.1

# prometheus helm 배포
$helm upgrade -i prometheus prometheus-community/kube-prometheus-stack -f /{path}/kube-prometheus-stack/prometheus-values.yaml -f /{path}/thanos/prometheus-thanos-values2.yaml -n monitoring
```

배포가 완료 되었으면, thanos 구성 전에 prometheus-2에서 thanos sidecar의 노출이 필요합니다.
(이부분은 처음 prometheus-values.yaml 설정시에 같이 진행하면 됨. 처음에 하나 나중에하나 차이는 type 변경하나임)

```bash
# prometheus-values.yaml 파일안에 type을 Loadbalancer로 변경하면 됩니다.
#type: ClusterIP
type: LoadBalancer
```

### thanos 배포

sidecar 배포시에 이미 thanos repo는 등록하였기 때문에 repo 등록은 넘어갑니다.
바로 위에서 생성한 thanos-values.yaml 파일로 thanos 배포진행 합니다.

```bash
$ helm upgrade -i thanos bitnami/thanos -n monitoring -f thanos-values.yaml

# helm release 확인(prometheus와 thanos의 배포시의 release 확인)
$ helm list -n monitoring
NAME       NAMESPACE  REVISION UPDATED                                  STATUS    CHART                        APP VERSION
prometheus monitoring 7        2024-03-28 13:15:39.597206318 +0900 KST  deployed  kube-prometheus-stack-57.1.1 v0.72.0
thanos     monitoring 1        2024-03-28 13:31:41.722690715 +0900 KST  deployed  thanos-14.0.2                0.34.1
```

prometheus와 thanos-query 포탈 접근하여 확인하기

```bash
kubectl port-forward service/thanos-query 9091:9090 -n monitoring --address=0.0.0.0 &
kubectl port-forward service/prometheus-prometheus 9090:9090 -n monitoring --address=0.0.0.0 &
```

## 구성시 에러

thanos-query에서 cluster-2가 확인되지 않음. + S3에 metric이 쌓이지 않음
thanos-ruler, thanos-storagegateway, thanos-compactor 세 pod가 pending 상태확인
pod 상태 체크

```bash
$ kubectl describe pods xxxxx -n monitoring

Warning  FailedScheduling  18m (x185 over 15h)  default-scheduler  0/2 nodes are available: 2 node(s) had untolerated taint {node.kubernetes.io/unreachable: }. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.

Warning  FailedScheduling  16m                  default-scheduler  0/4 nodes are available: 1 node(s) didn't find available persistent volumes to bind, 1 node(s) had untolerated taint {node.kubernetes.io/not-ready: }, 2 node(s) had untolerated taint {node.kubernetes.io/unreachable: }. preemption: 0/4 nodes are available: 4 Preemption is not helpful for scheduling.

Warning  FailedScheduling  16m                  default-scheduler  0/4 nodes are available: 2 node(s) didn't find available persistent volumes to bind, 2 node(s) had untolerated taint {node.kubernetes.io/unreachable: }. preemption: 0/4 nodes are available: 4 Preemption is not helpful for scheduling.

Warning  FailedScheduling  62s (x3 over 11m)    default-scheduler  0/2 nodes are available: 2 node(s) didn't find available persistent volumes to bind. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
```
