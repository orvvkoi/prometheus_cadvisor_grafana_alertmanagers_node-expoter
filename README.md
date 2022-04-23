## **구성**

**Server A:** Grafana, Prometheus, Alertmanager, cadvisor

**Server B, C, D**:  node-exporter Basic Auth and TLS


## Server B, C, D(원격 서버)

**디렉토리 생성**

```
mkdir -p node-exporter/config
```


**Node Exporter TLS 인증서 생성**

```
# 경로: node-exporter/config

openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 \
  -keyout ./server1.key \
  -out ./server1.crt \
  -subj "/CN=localhost" \
  -addext "subjectAltName = DNS:localhost"

chmod 644 server1.key 
chmod 644 server1.crt 
```


**Node Exporter TlS 설정 및 기본 인증 추가**

```
# 경로: node-exporter/config

# password 생성
htpasswd -nBC 10 "" | tr -d ':\n'

# node-config.yml 파일 생성
vi node-config.yml

# node-config.yml 내용 입력
tls_server_config:
  cert_file: server1.key 
  key_file: server1.key
basic_auth_users:
  prometheus: htpasswd로 생성된 값
```


**node-exporter 설치, 설정**

```
# 경로: node-exporter

curl https://raw.githubusercontent.com/orvvkoi/prometheus_cadvisor_grafana_alertmanagers_node-expoter/main/node-exporter.yml
docker-compose -f node-exporter.yml up -d

# Node Exporter Test
curl localhost:9100/metrics
curl --cacert server1.crt https://localhost:9100/metrics
```



## Server A(관리 서버)

**설치**

```
git clone https://github.com/orvvkoi/prometheus_cadvisor_grafana_alertmanagers_node-expoter.git
cd prometheus_cadvisor_grafana_alertmanagers_node-expoter


# https://grafana.com/docs/grafana/latest/installation/docker/#migrate-to-v51-or-later
sudo chown -R 472:472 grafana/
```


**원격 서버에서 생성한 crt 파일 복사**

```
#경로: prometheus/keys

prometheus/keys/server1.crt
prometheus/keys/server2.crt
prometheus/keys/server3.crt
...
...
```


**prometheus.yml 파일 수정**

```
# 경로: prometheus/prometheus.yml
#
# 파일내 주석 참고
#
# 수정 대상
# job_name: job-server1, job-server2 , job-server3

# example job-server1
- job_name: 'job-server1'
  scrape_interval: 8s
  basic_auth:
    username: "prometheus" # node-exporter node-config.yml의 basic_auth_users id
    password: "my paassword" # node-exporter node-config.yml의 basic_auth_users password의 원본값
  tls_config:
    insecure_skip_verify: true
    ca_file: ./keys/server1.crt # node-exporter node-config.yml의 cert_file 파일
  scheme: https
  static_configs:
    - targets: ['111.111.111.111:9100'] # 대상 서버 IP:PORT
      labels:
        group: "server"
        nodename: "server1-instance"
```


**alertmanager.yml 파일 수정**

```
# 경로: alertmanager/alertmanager.yml
# 파일내 주석 참고
# 상세 설정: https://prometheus.io/docs/alerting/latest/configuration/
#
# 수정대상: email, slack, discord
#
# discord는 alertmanager에서 webhook 작동 안함.
# benjojo/alertmanager-discord로 연동.
#
# 방법1: grafana 내장 alertmanager 사용하여 discord 연동. (자체 rule 생성 필요)
# 방법2: alertmanager와 alertmanager-discord 연동
```


**docker container 생성**

```
docker-compose up -d
```


**grafana 실행**

```
http://Server A IP:3100

Dashboard import: https://grafana.com/grafana/dashboards/11074
```

![dashboards](https://grafana.com/api/dashboards/11074/images/8427/image)


