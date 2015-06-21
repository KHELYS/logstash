# The purpose of this document
 - This document is a simple guide for installing & configuring Elasticsearch, Logstash and Kibana.
 - This document is revised for the KHELYS study group.

# 본 문서의 목적
- 본 문서는 KHELYS study를 위하여 재작성되었습니다.
- Elasticsearch, Logstash, Kibana의 설치와 설정을 위한 간단한 가이드입니다.

### author : dh.yeo in Naver (printfscanf@naver.com)
### last update : 150621

----

# Java 8 설치
```wget -c -O "jdk-8u45-linux-x64.tar.gz" --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u45-b14/jdk-8u45-linux-x64.tar.gz"```

```tar xvf jdk-8u45-linux-x64.tar.gz```

이후 환경변수, path 설정 등을 통해 새로 설치한 java가 구동되도록 설정합니다.
java -version과 which java를 활용할 수 있습니다.

예)
```
ln -s ~/apps/jdk1.8.0_45 ~/apps/jdk

vim ~/.bashrc
혹은
관리자 계정으로 접속
sudo vim /etc/profile

하단에 다음과 같이 설정 추가
export JAVA_HOME=~/apps/jdk
export PATH=$JAVA_HOME/bin:$PATH
```
-----
# Elasticsearch 설치
```wget https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-1.6.0.zip```

```unzip elasticsearch-1.6.0.zip```

잘 설치되었는지 및 버전 확인
```bin/elasticsearch -v```

### 설정
```ln -s ~/apps/elasticsearch-1.6.0 ~/apps/elasticsearch```

```vim ~/apps/elasticsearch/config/elasticsearch.yml```

```

예)
# 동일한 클러스터로 구성하기 위한 이름입니다.
cluster.name: my-cluster

# 클러스터 내에서 unique해야 하는 node 구분용 이름입니다.
node.name: "my-node01"

# 이하는 단일노드 구성시에는 설정할 필요가 없습니다.
# 클러스터 구성시 디폴트로 멀티캐스트 방식을 이용하는데, unicast를 이용하기 위하여 false로 설정을 합니다.
discovery.zen.ping.multicast.enabled: false
# 클러스터를 구성하는 호스트 목록을 입력합니다.
discovery.zen.ping.unicast.hosts: ["192.168.0.100:9300", "192.168.0.101:9300", "192.168.0.102:9300", "192.168.0.103:9300" ]
```

### head 플러그인 설치
```bin/plugin --install mobz/elasticsearch-head```

### 구동
데몬 모드로 구동합니다.
```bin/elasticsearch -d```

```
Usage: $0 [-vdh] [-p pidfile] [-D prop] [-X prop]
Start elasticsearch.
    -d            daemonize (run in background)
    -p pidfile    write PID to <pidfile>
    -h
    --help        print command line options
    -v            print elasticsearch version, then exit
    -D prop       set JAVA system property
    -X prop       set non-standard JAVA system property
   --prop=val
   --prop val     set elasticsearch property (i.e. -Des.<prop>=<val>)
```

### 구동 및 클러스터 현황 확인
http://localhost:9200/_plugin/head/

-----
# Logstash 설치
```wget http://download.elastic.co/logstash/logstash/logstash-1.5.0.zip```

```unzip logstash-1.5.0.zip```

### start, shutdown용 shell 작성
```ln -s ~/apps/logstash-1.5.0 ~/apps/logstash```

start, shutdown용 shell 작성
```vim ~/apps/logstash/bin/logstash-start.sh```

```
#!/bin/sh
SERVER_HOME="~/apps/logstash"
EXEC_FILE="logstash"
CONFIG_DIR="${SERVER_HOME}/conf.d"
LOG_DIR="~/data/logstash/log"
LOG_FILE="${LOG_DIR}/logstash.log"

mkdir -p $LOG_DIR

$SERVER_HOME/bin/$EXEC_FILE agent -f ${CONFIG_DIR} -l ${LOG_FILE} > ${LOG_DIR}/logstash.stdout 2> ${LOG_DIR}/logstash.err &

PID=$!

if [ $? -eq 0 ]
then
        echo "server start : pid=$PID"
        mkdir -p $SERVER_HOME/bin/pid/
        echo "$PID" > $SERVER_HOME/bin/pid/logstash.pid
        exit 0
else
        echo "server start faild"
        exit 1
fi
```

```vim ~/apps/logstash/bin/logstash-shutdown.sh```

```
#!/bin/sh
SERVER_HOME="~/apps/logstash"
NAME="logstash"
PIDFILE=$SERVER_HOME/bin/pid/logstash.pid
PID=`cat $PIDFILE`

status() {
        if [ -f "$PIDFILE" ] ; then
                if kill -0 $PID > /dev/null 2> /dev/null ; then
                        return 0 # process by this pid is running
                else
                        return 2 # program is dead but pid file exists
                fi
        else
                return 3 # program is not running
        fi
}


if status ; then
        PID=`cat $SERVER_HOME/bin/pid/logstash.pid`
        echo "Killing $NAME(pid $PID) with SIGTERM"
        kill -TERM $PID

        for i in 1 2 3 4 5 ; do
                echo "Waiting $NAME (pid $PID) to die..."
                status || break
                sleep 1
        done

        if status ; then
                echo "$NAME(pid $PID) stop failed; still running."
        else
                echo "$name stopped."
        fi
fi
```

```chmod +x ~/apps/logstash/bin/logstash-s*.sh```


### 설정
```mkdir ~/apps/logstash/conf.d```
```vim ~/apps/logstash/conf.d/logstash.conf```
input, filter, output 등 작성

### 설정 파일 예시
예1) kafka에서 특정 topic을 읽어서 elasticsearch로 output
```
input {
    kafka {
        zk_connect => "192.168.0.200:2181"
        auto_offset_reset => "largest"
        group_id => "logstash"
        topic_id => "my-topic"
        codec => plain {
            charset => "UTF-8"
        }
    }
}

filter {
    grok {
        match => [ "message", "%{TIMESTAMP_ISO8601:timestamp}(.*?\t)%{DATA:thread}(\t)%{IP:ip}(\t)%{GREEDYDATA:msg}" ]
    }
}

output {
    elasticsearch {
        host => ["192.168.0.100:9300", "192.168.0.101:9300", "192.168.0.102:9300", "192.168.0.103:9300"]
        cluster => "my-cluster"
        protocol => "node"
    }

    # stdout { codec => rubydebug }
}
```
예2) 로컬 파일에서 읽어들이고, timestamp를 해당 로그에 기록된 데이터 기준으로 변경하며, elasticsearch로 output
```
input {
        file {
                path => "/data/file/*.log"
		# 파일의 어느 부분부터 읽을 것인지지를 지정
                start_position => beginning
		# 파일의 어느 부분까지 읽었는지를 기록하는 곳을 지정
                sincedb_path => "/dev/null"
                type => "my_type"
		# 여러 줄로 된 로그라도 하나의 로그로 인식하도록 설정
                codec => multiline {
			# 특정한 패턴을 지정하여
                        pattern => "(\t)at (%{GREEDYDATA:errormsg})"
			# 해당 패턴과 일치하거나(negate => false), 일치하지 않으면(negate => true)
                        negate => true
			# 해당 패턴 일치/불일치 발생의 이전(previous) 라인이나 이후(what) 라인과 한 묶음으로 인식
                        what => previous
                        charset => "UTF-8"
                }
        }
}

filter {
        grok {
                match => [ "message", "%{TIMESTAMP_ISO8601:REG_TM}(\|)%{DATA:CLIENT_IP}(\|)%{DATA:SERVER_IP}" ]
        }

        date {
                match => [ "REG_TM", "YYYY-MM-dd HH:mm:ss.S" ]
                target => "@timestamp"
        }

}

output {
    elasticsearch {
        host => ["192.168.0.100:9300", "192.168.0.101:9300", "192.168.0.102:9300", "192.168.0.103:9300"]
        cluster => "my-cluster"
        protocol => "node"
    }

        #stdout { codec => rubydebug }
}
```

grok filter syntax의 경우 정규표현식을 기반으로 하여 그 위에 다양한 패턴이 지정이 되어 있는데, 이 곳에서 확인할 수 있습니다.
https://github.com/elastic/logstash/blob/v1.4.0/patterns/grok-patterns

또한 작성한 grok filter 패턴의 테스트를 이 곳에서 해볼 수 있습니다.
http://grokdebug.herokuapp.com/

이 이외의 예시는 아래 문서를 참고부탁드립니다.

-----
# Kibana 설치
```wget https://download.elastic.co/kibana/kibana/kibana-4.1.0-linux-x64.tar.gz```

```tar xvf kibana-4.1.0-linux-x64.tar.gz```

```ln -s ~/apps/kibana-4.1.0-linux-x64 ~/apps/kibana```

### 설정
```vim ~/apps/kibana/config/kibana.yml```

```
# elasticsearch의 위치를 지정합니다.
elasticsearch_url: "http://localhost:9200"
```
마스터 노드도 데이터 노드도 아닌 elasticsearch의 노드에 kibana를 함께 설치하고, localhost로 가리키는 경우 좀 더 스마트하게 로드 밸런싱 처리를 할 수 있다고 이야기하고 있습니다.

참고 : https://www.elastic.co/guide/en/kibana/current/production.html#load-balancing


### start, shutdown용 shell 파일 작성
```vim bin/kibana-start.sh```

```
#!/bin/sh
SERVER_HOME="~/apps/kibana"
EXEC_FILE="kibana"
CONFIG_DIR="${SERVER_HOME}/conf.d"
LOG_DIR="~/data/kibana/log"
LOG_FILE="${LOG_DIR}/kibana.log"

mkdir -p $LOG_DIR

$SERVER_HOME/bin/$EXEC_FILE > $LOG_FILE 2> $LOG_FILE.err &

PID=$!

if [ $? -eq 0 ]
then
        echo "server start : pid=$PID"
        mkdir -p $SERVER_HOME/bin/pid/
        echo "$PID" > $SERVER_HOME/bin/pid/kibana.pid
        exit 0
else
        echo "server start faild"
        exit 1
fi
```

```vim bin/kibana-shutdown.sh```

```
#!/bin/sh
NAME="kibana"
SERVER_HOME="~/apps/$NAME"
PIDFILE=$SERVER_HOME/bin/pid/$NAME.pid
PID=`cat $PIDFILE`

status() {
        if [ -f "$PIDFILE" ] ; then
                if kill -0 $PID > /dev/null 2> /dev/null ; then
                        return 0 # process by this pid is running
                else
                        return 2 # program is dead but pid file exists
                fi
        else
                return 3 # program is not running
        fi
}


if status ; then
        PID=`cat $SERVER_HOME/bin/pid/$NAME.pid`
        echo "Killing $NAME(pid $PID) with SIGTERM"
        kill -TERM $PID

        for i in 1 2 3 4 5 ; do
                echo "Waiting $NAME (pid $PID) to die..."
                status || break
                sleep 1
        done

        if status ; then
                echo "$NAME(pid $PID) stop failed; still running."
        else
                echo "$name stopped."
        fi
fi
```

``` chmod +x kibana-*.sh```


### 구동

``` bin/kibana-start.sh ```

이후 웹브라우저에서 5601번 포트로 접속하면 됩니다.
``` http://localhost:5601 ```

Settings - Indices에서 Time-field name을 선택하여 @ timestamp 등을 선택하고 Create하여 인덱스를 생성하고,
Discover, Visualize, Dashboard의 각 메뉴에서 여러 기준으로 로그의 분석 내용을 시각화할 수 있습니다.

-----


이 이하의 내용은 예전에 작성된 내용으로, 참고삼아 남겨 둡니다.

-----
# logstash의 로그 수집에 대하여
여러 방법으로 로그 수집이 가능하며, 몇몇 방법의 설치/설정에 대해 알아봅니다.

 **1. logstash-forwarder가 logstash server에 client로 붙어서 로그를 push (로그 파일이 기록되는 곳마다 logstash-forwarder 설치)**
 
 **2. 로그를 발생시키는 서비스에서 직접 logstash server에 로그를 push (logback에 appender를 붙여서 이용)**
 
 **3. logstash server가 자신의 로컬 파일에서 로그를 수집하며, (로그 파일이 기록되는 곳마다 logstash server를 설치하여) elasticsearch server로 push**
 
 **4. kafka에서 topic 구독**

-----
# CASE 01. logstash-forwarder 설치

https://github.com/elastic/logstash-forwarder

logstash-forwarder는 logstash server에 client로 붙어 stdin/file 등을 이용하여 로그를 push하는 역할을 합니다.
logback을 이용하여 로그를 보내거나, logstash server에서 자체적으로 해당 로컬의 파일에서 로그를 읽어오는 경우에는 logstash-forwarder가 필요치 않습니다.

### Elasticsearch public GPG key를 import
```sudo rpm --import http://packages.elasticsearch.org/GPG-KEY-elasticsearch```

### logstash-forwarder 설치를 위한 repository
```sudo vim /etc/yum.repos.d/logstash-forwarder.repo```
```
[logstash-forwarder]
name=logstash-forwarder repository
baseurl=http://packages.elasticsearch.org/logstashforwarder/centos
gpgcheck=1
gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
enabled=1
```

### logstash-forwarder 설치
```sudo yum -y install logstash-forwarder```

### logstash-forwarder 설정
```sudo vim /etc/logstash-forwarder.conf```

network 부분에서 다음 세 줄의 주석을 해제하여 내용을 작성합니다.

```
    "servers": [ "logstash_server_private_IP:5000" ],
    "timeout": 15
```
logstash_server_private_IP:5000 부분에는 logstash server의 ip와 logstash server의 input 설정에서 지정한 포트를 입력합니다.

"files" 부분을 다음과 같이 작성합니다.
```
  "files": [
    {
        "paths": [
        "/var/log/messages",
        "/var/log/secure"
       ],
      "fields": { "type": "syslog" }
    }
  ]
```

따라서, 주석을 제외하면 다음과 같은 내용이 됩니다.
```
{
  "network": {
    "servers": [ "logstash_server_private_IP:5000" ],
    "timeout": 15
  },
  "files": [
    {
        "paths": [
        "/var/log/messages",
        "/var/log/secure"
       ],
      "fields": { "type": "syslog" }
    }
  ]
}
```

### logstash-forwarder 시작
```sudo /etc/init.d/logstash-forwarder start```

-----

# CASE02. logback에 appender을 붙여 사용

logstash-logback-encoder 프로젝트를 이용합니다.
https://github.com/logstash/logstash-logback-encoder


## Maven / Gradle dependency 추가

Maven의 경우
```
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>4.3</version>
</dependency>
```

Gradle의 경우
```
compile("net.logstash.logback:logstash-logback-encoder:4.3")
```

이 이외에도, 다음과 같은 라이브러리 의존성이 추가되어 있어야 합니다.
```
jackson-databind / jackson-core / jackson-annotations
logback-core
logback-classic (required for logging LoggingEvents)
logback-access (required for logging AccessEvents)
slf4j-api
```

## logback.xml 설정
```
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <!-- remoteHost and port are optional (default values shown) -->
        <remoteHost>10.113.182.124</remoteHost>
        <port>5000</port>

        <!-- encoder is required -->
        <encoder class="net.logstash.logback.encoder.LogstashEncoder" />

        <keepAliveDuration>5 minutes</keepAliveDuration>
    </appender>

    <root level="INFO">
        <appender-ref ref="LOGSTASH"/>
    </root>
```

## logstash의 input 설정 조정
위에서 나왔던 것과 같이, TCP appender를 붙인 경우에는 logstash의 input을 tcp {}로 받아야 합니다.
```
input {
    tcp {
        port => 5000
        codec => json_lines
    }
}
```

-----

# CASE03. logstash가 직접 로컬의 파일로부터 로그 수집

logstash의 설정을 다음과 같이 지정합니다.
```
input {
  file {
    path => "/var/log/messages"
    type => "syslog"
  }

  file {
    path => "/var/log/apache/access.log"
    type => "apache"
  }
}
```

그러나, logstash는 stream event에 반응하며, 기존 보유 파일들을 sincedb라는 곳에 기억하고 있기 때문에, 기존 파일을 불러들여 인덱싱하는 것을 필요로 하는 경우 다음과 같이 설정해야 합니다. start_position은 파일을 읽기 시작하는 위치 (beginning/end), sincedb_path는 기존 파일의 변경 유무를 체크하기 위한 정보를 저장하는 경로입니다.

```
input {
  file {
    path => "/var/log/messages"
    type => "syslog"
    start_position => beginning
    sincedb_path => "/dev/null"    
  }
}
```

-----

# CASE04. kafka에서 topic 구독하기

logstash의 input 설정을 다음과 같이 설정합니다.
```
input {
    kafka {
        zk_connect => "localhost:2181"
         group_id => "logstash"
        topic_id => "test"
    }
}
```

-----

# Reference

Elastic
https://www.elastic.co/products

Logstash-logback-encoder
https://github.com/logstash/logstash-logback-encoder

Logstash Learn|Docs
https://www.elastic.co/guide/en/logstash/current/index.html

Kibana User Guide
https://www.elastic.co/guide/en/kibana/current/index.html

How To Install Elasticsearch, Logstash, and Kibana 4 on CentOS 7
https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-4-on-centos-7

Grok Patterns
https://github.com/elastic/logstash/blob/v1.4.0/patterns/grok-patterns

Grok debugger
http://grokdebug.herokuapp.com/
