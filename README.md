# elastic-stack
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

# Optional: Generate additional certificates specifically for encrypting HTTP client communications.
# 필요 없을 듯 하지만 일단 엔터
# elasticsearch-ssl-http.zip
bin/elasticsearch-certutil http
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
```

## config/kibana.yml

```yaml
46,47c46,50
elasticsearch.username: "kibana_system"
elasticsearch.password: "elastic"
elasticsearch.ssl.truststore.path: elastic-stack-ca.p12
elasticsearch.ssl.truststore.password: ""
elasticsearch.hosts: ["http://localhost:9200"]
xpack.ingestManager.fleet.tlsCheckDisabled: true
xpack.encryptedSavedObjects.encryptionKey: a123456789012345678901234567890b
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
./elastic-agent enroll http://localhost:5601 ak9ENXdIUUJWWUZqWXVvUlpWTl86akJNSG10UHpRZVNGUnduNlN5QTV6Zw==
./elastic-agent run
```

## In Ingest Manager, click Continue to go to the Fleet tab, where you should see the newly enrolled agent.

이 화면이 나올거라는데.. nginx 를 안띄워놔서 그런가 빈화면만 보이고 있다.
![](https://www.elastic.co/guide/en/ingest-management/current/images/kibana-ingest-manager-fleet-agents.png)


이게 안보여서 매뉴얼 모드로 띄워보았다.

## [Mananual mode](https://www.elastic.co/guide/en/ingest-management/current/ingest-management-getting-started.html#agent-standalone-mode)

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

... 매뉴얼 모드도 최종 결과대로는 안나옴.. 자야겠

# alert & action


# 참고
- https://jjeong.tistory.com/1503
