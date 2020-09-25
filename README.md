# elastic-stack - !! under construction !!
elastic-stack

---

# .
elastic 7.9.1 의 alert, injest manager 를 사용하기 위해 로컬에서 테스트해보았다.

- [Quick start: Get logs and metrics into the Elastic Stackedit](https://www.elastic.co/guide/en/ingest-management/current/ingest-management-getting-started.html)


# elastic

## download &
```console
tar xvfz elasticsearch-7.9.1-darwin-x86_64.tar.gz
cd elasticsearch-7.9.1
```

## [Configuring security in Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/configuring-security.html)

## [Configure Transport Layer Security (TLS/SSL) for internode-communication](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/configuring-tls.html)
```console
# Optional: Create a certificate authority for your Elasticsearch cluster.
# 많은 걸 물어봤는데 일단 다 엔터
bin/elasticsearch-certutil ca

# Generate a certificate and private key for each node in your cluster.
# 이것도 다 엔터
# elastic-stack-ca.p12
bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
```

http client 용 ca 생성
- `Do you wish to change any of these options? [y/N]` 물어볼때 `y` 로 입력 후
- CN=localhost 로 입력
- [elasticsearch-certutil http](elasticsearch-certutil-http.md) 참고
```
bin/elasticsearch-certutil http
```

생성된 ssl http 인증서 사용
```
unzip elasticsearch-ssl-http.zip
cp elasticsearch/http.p12 config/

bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
```

## config/elasticsearch.yml

```yaml
# 추가
xpack.security.enabled: true
xpack.security.authc.api_key.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-stack-ca.p12
xpack.security.transport.ssl.truststore.path: elastic-stack-ca.p12

# This turns on SSL for the HTTP (Rest) interface
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
xpack.security.http.ssl.truststore.path: http.p12
xpack.security.http.ssl.client_authentication: optional
```

## add the password to your Elasticsearch keystore
```console
# 암호가 있다면 keystore에 저장 (테스트용은 암호셋팅이 없어서 필요는 없었을 듯)
bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

## elasticsearch 실행
```console
ES_PATH_CONF=$PWD/config bin/elasticsearch
```

## [Set built-in user passwords](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/built-in-users.html#set-built-in-user-passwords)

```console
# elastic으로 통일했다

bin/elasticsearch-setup-passwords interactive
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,kibana_system,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N] y


Enter password for [elastic]: elastic
Reenter password for [elastic]: elastic
Enter password for [apm_system]: elastic
Reenter password for [apm_system]: elastic
Enter password for [kibana_system]: elastic
Reenter password for [kibana_system]: elastic
Enter password for [logstash_system]: elastic
Reenter password for [logstash_system]: elastic
Enter password for [beats_system]: elastic
Reenter password for [beats_system]: elastic
Enter password for [remote_monitoring_user]: elastic
Reenter password for [remote_monitoring_user]: elastic
Changed password for user [apm_system]
Changed password for user [kibana_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]
```

# kibana

- [Configuring security in Kibana](https://www.elastic.co/guide/en/kibana/7.9/configuring-tls.html)

## download &
```
tar xvfz kibana-7.9.2-darwin-x86_64.tar.gz
cd kibana-7.9.2-darwin-x86_64

# http pem 인증서를 복사 - elasticsearch 와 http tls 통신을 위해
cp elasticsearch-7.9.1/kibana/elasticsearch-ca.pem kibana-7.9.2-darwin-x86_64/config
```

## kibana 서버 ssl 셋팅
```
mkcert -install
mkcert -cert-file config/kibana-server-crt.pem -key-file config/kibana-server-key.pem localhost 127.0.0.1 ::1
bin/kibana-keystore add server.ssl.keyPassphrase
```


## config/kibana.yml

```yaml
elasticsearch.username: "kibana_system"
elasticsearch.password: "elastic"
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.ssl.truststore.path: elastic-stack-ca.p12
elasticsearch.ssl.truststore.password: ""
elasticsearch.ssl.certificateAuthorities: [ "config/elasticsearch-ca.pem" ]

server.ssl.enabled: true
server.ssl.certificate: "config/kibana-server-crt.pem"
server.ssl.key: "config/kibana-server-key.pem"

xpack.ingestManager.enabled: true
xpack.security.enabled: true
xpack.encryptedSavedObjects.encryptionKey: a123456789012345678901234567890b
xpack.security.encryptionKey: "01234567890123456789012345678912"
xpack.reporting.encryptionKey: "01234567890123456789012345678912"
```

- TODO kibana https localhost 서빙이 되어야 한다.
  - injest management 실행하니 es 에 아래의 메시지가 출력되면서 metric이 저장되지 않는 현상
  
```console
[2020-09-25T10:51:10,690][WARN ][o.e.x.s.t.n.SecurityNetty4HttpServerTransport] [mdui-MacBookPro.local] received plaintext http traffic on an https channel, closing connection Netty4HttpChannel{localAddress=/127.0.0.1:9200, remoteAddress=/127.0.0.1:59409}
```

## kibana 실행

```
bin/kibana
```

# injest management

페이지를 열고 그대로 따라해보자

- [Get logs and metrics into the Elastic Stack](https://www.elastic.co/guide/en/ingest-management/current/ingest-management-getting-started.html#ingest-management-getting-started)

최종적으로 이 화면이 나와야 함

![installed](https://www.elastic.co/guide/en/ingest-management/current/images/kibana-ingest-manager-integrations-list-installed.png)


## Fleet Mode, standalone mode 중 선택

donload &
```console
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-7.9.1-darwin-x86_64.tar.gz
tar xzvf elastic-agent-7.9.1-darwin-x86_64.tar.gz
cd elastic-agent-7.9.1
```

Fleet 으로 진행해보자. 페이지를 열고 따라하기

- [Ingest Management - Agent Fleet Mode](https://www.elastic.co/guide/en/ingest-management/current/ingest-management-getting-started.html#agent-fleet-mode)

이 화면을 봐야 함

![add agent](https://www.elastic.co/guide/en/ingest-management/current/images/kibana-ingest-manager-fleet-enrol.png)

## 최종 화면에 보이는 elastic-agent 커맨드를 수행

```console
./elastic-agent enroll https://localhost:5601 ak9ENXdIUUJWWUZqWXVvUlpWTl86akJNSG10UHpRZVNGUnduNlN5QTV6Zw==
./elastic-agent run
```

## In Ingest Manager, click Continue to go to the Fleet tab, where you should see the newly enrolled agent.

![](https://www.elastic.co/guide/en/ingest-management/current/images/kibana-ingest-manager-fleet-agents.png)



## [Manual mode](https://www.elastic.co/guide/en/ingest-management/current/ingest-management-getting-started.html#agent-standalone-mode)

![](https://www.elastic.co/guide/en/ingest-management/current/images/kibana-ingest-manager-configurations-default-yaml.png)

설정을 복사해와 elastic-agent.yml에 저장할때 ES_USERNAME, ES_PASSWORD를 설정한 암호로 바꾸자. (elastic/elastic)
```yaml
outputs:
  default:
    type: elasticsearch
    hosts:
      - 'HOST:PORT'
    username: ES_USERNAME
    password: ES_PASSWORD
```

elastic-agent 실행
```console
./elastic-agent run
```

## trouble shootings

## elatstic-agent 설치 위치
es 서버에 설치해야하나? 아니면 kibana 서버에 설치해야하는지? 
아니면 모니터링 대상 서버에 하면 되나?

### http(s) 통신
ES 로그
```
[2020-09-25T12:57:00,579][WARN ][o.e.x.s.t.n.SecurityNetty4HttpServerTransport] [mdui-MacBookPro.local] received plaintext http traffic on an https channel, closing connection Netty4HttpChannel{localAddress=/127.0.0.1:9200, remoteAddress=/127.0.0.1:61533}
```

tls 통신만을 허용하는 상황에서 elastic-agent가 http 평문으로 통신을 시도하는 모양이다.

9200 포트를 찾아서 https://localhost:9200 으로 변경을 해주었다.
```
grep 9200 *yml
action_store.yml:      - https://localhost:9200
elastic-agent.reference.yml:    hosts: [https://localhost:9200]
elastic-agent.yml:    hosts: [https://localhost:9200]
```

다시 agent 실행
```
./elastic-agent run
```

이번에는 이런 ES에러가..

```
[2020-09-25T20:55:11,749][WARN ][o.e.h.AbstractHttpServerTransport] [mdui-MacBookPro.local] caught exception while handling client http traffic, closing connection Netty4HttpChannel{localAddress=/127.0.0.1:9200, remoteAddress=/127.0.0.1:50698}
io.netty.handler.codec.DecoderException: javax.net.ssl.SSLHandshakeException: Received fatal alert: bad_certificate
...
```

아직 원인 찾지 못함. elastic-agent yml 설정에 ssl 관련 설정 추가하는게 있어야 할 것같은데 reference 에서 찾지 못한 상태

베타라 그런가 문서 설명이 부실하다.

https://github.com/elastic/kibana/issues/73483
```
outputs:
    default:
      api_key: 7_uoA.....
      hosts:
      - https://elasticsearch:9200
      type: elasticsearch
      ssl.certificate_authorities: ["/home/jamie/Projects/GitRepo/apm-integration-testing/scripts/tls/ca/ca.crt"]
```

https://github.com/elastic/kibana/issues/75913
이글 보면 최근까지 yml output에 추가할 cert 필드들에 대한 논의 진행중

# alert & action

- [Alerting and Actions](https://www.elastic.co/guide/en/kibana/7.9/alerting-getting-started.html)

cf. [Elasticsearch alerting](https://www.elastic.co/guide/en/kibana/7.9/watcher-ui.html)

- [Defining alerts](https://www.elastic.co/guide/en/kibana/7.9/defining-alerts.html)

## [Slack Action Type](https://www.elastic.co/guide/en/kibana/7.9/slack-action-type.html)

kibana 서버에서 slack 에 접근할 수 있도록 작업이 되어 있어야 한다

```xml
# kibnana.yml
xpack.actions.whitelistedHosts=[""]

my-slack:
   name: preconfigured-slack-action-type
   actionTypeId: .slack
   config:
     webhookUrl: 'https://hooks.slack.com/services/abcd/efgh/ijklmnopqrstuvwxyz'
```

# 참고
- https://jjeong.tistory.com/1503
