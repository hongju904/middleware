# WEB 설치

1. apache 설치
- http://httpd.apache.org/download.cgi : httpd, apr, apr-util, pcre 다운로드
- https://www.openssl.org/source/ : openssl 

압축풀기: ``tar xzvf [파일명]``

#### ◇ 순서대로, 각 압축이 풀린 폴더에서 진행 (pwd로 경로 확인)
cd ~ 의 경로가 /home/user1 일 경우를 기준으로 아래 작성됨   
apr - apr-util - pcre 순서로 의존성을 가지기 때문에,   
``--prefix``옵션으로 설치경로를 설정하고 ``--with``옵션으로 의존성을 연결한다.

- apr
```bash
./configure --prefix=/home/user1/lib/apr
make && make install
```

- apr-util
```bash
./configure --prefix=/home/user1/lib/apr-util --with-apr=/home/user1/lib/apr
make && make install
```

- pcre
```bash
./configure --prefix=/home/user1/lib/pcre \
--with-apr-util=/home/user1/lib/apr-util \
--with-apr=/home/user1/lib/apr
make && make install
```

- openssl
```bash
./config --prefix=/home/user1/lib/openssl
make && make install
```
openssl의 경우, httpd에서 의존성을 얻어야 하므로 미리 설정 (사용하지 않는다면 생략)

- httpd
```bash
./configure --prefix=/home/user1/httpd \
--enable-modules=all --enable-mods-shared=all --enable-so --enable-ssl \
--with-apr=/home/user1/lib/apr \
--with-apr-util=/home/user1/lib/apr-util \
--with-pcre=/home/user1/lib/pcre/bin/pcre-config \
--with-ssl=/home/user1/lib/openssl
make && make install
```

* tomcat과 연동할 경우

https://tomcat.apache.org/download-connectors.cgi : mod_jk 다운로드 후 압축해제   
``cd native``   
``find ~ -name apxs`` 로 apxs가 어디에 있는지 탐색  
```bash
./configure --with-apxs=/home/user1/httpd/bin/apxs (해당경로)
make && make install
```

마지막으로, ``vi ~/httpd/conf/httpd.conf``
```bash
# 기본 도메인,PORT 지정
Listen 5090

# 주석 해제, 도메인 설정
# 도메인이 없을 경우 서버 IP로 지정 (예시는 localhost)
ServerName 127.0.0.1

# https 사용시 관련 주석 해제
LoadModule ssl_module modules/mod_ssl.so
# ssl.conf 참조 추가 
Include conf/httpd-ssl.conf

# tomcat 연동시 mod_jk 관련 주석 해제
Include conf/mod_jk.conf
```

---

2. tomcat 연동
httpd/modules에 ``mod_jk.so`` 생성된 것 확인

``vi ~/httpd/conf/mod_jk.conf``
```bash
LoadModule jk_module modules/mod_jk.so

JkWorkersFile conf/workers.properties
JkMountFile conf/uriworkermap.properties

JkLogFile logs/mod_jk.log
JkShmFile logs/mod_jk.shm
JkLogLevel info
JkLogStampFormat "[%a %b %d %H:%M:%S %Y]"
```

``vi ~/httpd/conf/workers.properties``
```bash
worker.list=tomcat

#config template
worker.template.type=ajp13
worker.template.lbfactor=1
worker.template.socket_timeout=600
worker.template.socket_keepalive=true
worker.template.recovery_options=7
worker.template.ping_mode=A
worker.template.ping_timeout=10000
worker.template.connection_pool_size=1000
worker.template.connection_pool_minsize=1000
worker.template.connection_pool_timeout=60
worker.template.max_packet_size=65536

#config was1
worker.was1.reference=worker.template
worker.was1.host={WAS IP 적어넣기}
worker.was1.port={WAS AJP port 적어넣기}
worker.was1.route=was11

#config was2
worker.was2.reference=worker.template
worker.was2.host={WAS IP 적어넣기}
worker.was2.port={WAS AJP port 적어넣기}
worker.was2.route=was21

#config loadbalancer
worker.kwkep.type=lb
worker.kwkep.retries=10
worker.kwkep.sticky_session=1
worker.kwkep.balance_workers=was1,was2
```
위 예시는 WAS 2대에 로드밸런싱으로 연결한 예시이므로, WAS 숫자에 따라 코드를 제거하거나 증가시킨다.

``vi ~/httpd/conf/uriworkermap.properties``
```bash
/*= tomcat
# 아래로 제외할 확장자 !/*.html 등으로 작성
```

---

3. https (ssl) 설정

https 설정은 lib/openssl에서 진행

- openssl genrsa -des3 -out server.key 2048 이용하여 서버 개인key를 발급받음
- openssl req -new -key server.key -out server.csr 를 이용하여 인증요청서 생성 (개인키 입력)   
- openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt 로 인증서 생성 (개인키 입력)
- server.crt, server.key 생성됨 (경로 확인)

``vi ~/httpd/conf/httpd-ssl.conf``
```bash
Listen {ssl port 지정}

<VirtualHost *:{위에서 지정한 port}>     
    ServerName {도메인 혹은 서버 IP}
    DocumentRoot {http에서 사용하던 root 사용 or 다른 docs 경로 지정}
    SSLEngine on   
    SSLProtocol all -SSLv2 -SSLv3
    SSLCipherSuite HIGH:!RC4:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!EXP:+MEDIUM
    SSLHonorCipherOrder on
    SSLCertificateFile /home/user1/lib/openssl/server.crt
    SSLCertificateKeyFile /home/user1/lib/openssl/server.key

    JkMountFile conf/uriworkermap.properties

# 아래는 LOG 설정
ErrorLog "|/home/user1/httpd/bin/rotatelogs /home/user1/httpd/logs/ssl_error.%Y%m%d 48600"
TransferLog "|/home/user1/httpd/bin/rotatelogs /home/user1/httpd/logs/ssl_access.%Y%m%d 48600"
</VirtualHost>
```
SSLProtocol와 SSLCipherSuite는 브라우저나 보안 요구사항에 따라 변경할 수 있다.

---

4. Log 커스텀

httpd.conf에서 수정   
날짜별로 정리가 되도록 커스텀한 예시
- access.log
```bash
CustomLog "|/home/user1/httpd/bin/rotatelogs logs/access_log.%Y%m%d 86400" combined
```
- error.log
```bash
ErrorLog "|/home/user1/httpd/bin/rotatelogs logs/error_log.%Y%m%d 86400"
```
---

모든 설정이 끝나면, httpd/bin 에서

```./apachctl -k start``` 혹은   
```./httpd -k start```

종료는 ``stop``, 재기동은 ``restart``로 실행