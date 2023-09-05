# WAS 설치

1. tomcat 설치
- https://tomcat.apache.org/download-80.cgi : tomcat
- https://www.oracle.com/java/technologies/downloads/#java11 : jdk 다운로드

압축풀기: ``tar xzvf [파일명]``

### ◇ jdk 환경변수 설정
``vi ~/.bash_profile``
```bash
export JAVA_HOME=/sw/home/btp4/jdk11
export PATH=$PATH:$JAVA_HOME/bin
```
``source .bash_profile``

---

2. instance 생성
tomcat 경로에 domains 디렉토리 생성하여 instance 개별 폴더 관리   
톰캣의 ``conf`` 디렉토리 복사, ``work`` 및 ``logs`` 디렉토리 신규 생성

### 1) port 설정
``vi conf/server.xml``
```bash
# 수정
<Server port="${tomcat.port.shutdown}" shutdown="SHUTDOWN">

# 수정
<Connector port="${tomcat.port.http}" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="${tomcat.port.https}"
               maxParameterCount="1000"
               />

# 주석 해제 후 변경
<Connector protocol="AJP/1.3"
               address="0.0.0.0"
               port="${tomcat.port.ajp}"
               redirectPort="${tomcat.port.https}"
               maxParameterCount="1000"
               secretRequired="false"
               />

# 수정
<Host name="localhost"  appBase="${tomcat.app.home}"
            unpackWARs="true" autoDeploy="true">
```

### 2) 스크립트 설정

``vi env.sh``
```
export DATE=`date +%Y%m%d`

# check :: 1. USER_N / 2. SVCNAME / 3. TOMCAT_HOME
export USER_N=user1
export SVCNAME=instance1
export TOMCAT_HOME=/sw/home/${USER_N}/tomcat
export SERVER_HOME=$TOMCAT_HOME/domains/${SVCNAME}
export LOG_DIR=/sw/home/${USER_N}/LOGS
export LOG_HOME=$LOG_DIR/${SVCNAME}
# export APP_HOME=/sw/home/${USER_N}/webroot
export APP_HOME=$TOMCAT_HOME/webapps

# check :: PORT_OFFSET
export PORT_OFFSET=11

let SHUTDOWN_PORT=8005+${PORT_OFFSET}
let HTTP_PORT=8080+${PORT_OFFSET}
let HTTPS_PORT=8443+${PORT_OFFSET}
let AJP_PORT=8009+${PORT_OFFSET}

export SHUTDOWN_PORT
export HTTP_PORT
export HTTPS_PORT
export AJP_PORT

# CATALINA setting
export CATALINA_HOME=${TOMCAT_HOME}
export CATALINA_BASE=${SERVER_HOME}
export CATALINA_OUT=${LOG_HOME}/${SVCNAME}_${DATE}.log

# JVM Options : Server
export JAVA_OPTS="-server ${JAVA_OPTS}"
# JVM Options : Memory
export JAVA_OPTS="${JAVA_OPTS} -Xms1024m -Xmx1024m -Xss256k"

# jdk
export JAVA_OPTS="${JAVA_OPTS} -Dspring.profiles.active=prod"
export JAVA_OPTS="${JAVA_OPTS} -Xlog:gc*:${LOG_HOME}/gc_${DATE}.log"
export JAVA_OPTS="${JAVA_OPTS} -XX:+UseParallelOldGC"

export JAVA_OPTS="${JAVA_OPTS} -XX:+ExplicitGCInvokesConcurrent"
export JAVA_OPTS="${JAVA_OPTS} -XX:+HeapDumpOnOutOfMemoryError"
export JAVA_OPTS="${JAVA_OPTS} -XX:HeapDumpPath=${LOG_HOME}/heapDump_${DATE}.hprof"

export JAVA_OPTS="${JAVA_OPTS} -Djava.net.preferIPv4Stack=true"
export JAVA_OPTS="${JAVA_OPTS} -Dsun.rmi.dgc.client.gcInterval=3600000"
export JAVA_OPTS="${JAVA_OPTS} -Dsun.rmi.dgc.server.gcInterval=3600000"
export JAVA_OPTS="${JAVA_OPTS} -Djava.awt.headless=true"

# tomcat server.xml setting
export JAVA_OPTS="${JAVA_OPTS} -Dfile.encoding=UTF-8"
export JAVA_OPTS="${JAVA_OPTS} -DjvmRoute=${SVCNAME}"
export JAVA_OPTS="${JAVA_OPTS} -Dtomcat.app.home=${APP_HOME}"
export JAVA_OPTS="${JAVA_OPTS} -Dtomcat.port.shutdown=${SHUTDOWN_PORT}"
export JAVA_OPTS="${JAVA_OPTS} -Dtomcat.port.http=${HTTP_PORT}"
export JAVA_OPTS="${JAVA_OPTS} -Dtomcat.port.https=${HTTPS_PORT}"
export JAVA_OPTS="${JAVA_OPTS} -Dtomcat.port.ajp=${AJP_PORT}"
export JAVA_OPTS="${JAVA_OPTS} -Dtomcat.log.home=${LOG_HOME}"

export JAVA_OPTS="${JAVA_OPTS} -Djava.security.egd=file:/dev/./urandom"
export JAVA_OPTS="${JAVA_OPTS} -Djsse.enableSNIExtension=false"

echo "============================================================="
echo "   Apache Tomcat 8.x Info                                    "
echo "-------------------------------------------------------------"
echo "instance name : ${SVCNAME}"
echo "APP_HOME : ${APP_HOME}"
echo "LOG_HOME : ${LOG_HOME}"
echo "SHUTDOWN_PORT : ${SHUTDOWN_PORT}"
echo "HTTP_PORT : ${HTTP_PORT}"
echo "HTTPS_PORT : ${HTTPS_PORT}"
echo "AJP_PORT : ${AJP_PORT}"
echo "============================================================="
```
USER_N, SVCNAME, TOMCAT_HOME 에 오류가 없는지 linux 환경과 경로를 확인 후 수정   
PORT_OFFSET을 수정함으로써 port 번호를 일괄적으로 변경한다.

``vi startup.sh``
```bash
#!/bin/bash
. ./env.sh

cd ${CATALINA_HOME}/bin
./startup.sh

echo ""
echo "tail -f ${CATALINA_OUT}"
echo ""
```

``vi shutdown.sh``
```bash
#!/bin/bash
. ./env.sh

PID=`ps -ef | grep java | grep ${SVCNAME} | awk '{print $2}'`
if [ x${PID} = "x" ]; then
 echo "Tomcat SERVER ${SVCNAME} Cannot started..."
 exit;
fi

cd ${CATALINA_HOME}/bin
./shutdown.sh
```
이후 ``chmod 755 startup.sh shutdown.sh env.sh``   
``./startup.sh``로 기동 가능하다.