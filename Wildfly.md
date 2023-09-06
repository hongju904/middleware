# Wildfly 설치
- https://www.wildfly.org/

압축풀기: ``tar xzvf [파일명]``

---

**1. standalone mode**
- 서버 인스턴스마다 관리 기능 제공

``vi wildfly/standalone/configuration/standalone.xml``
```bash
<interfaces>
        <interface name="management">
            <inet-address value="${jboss.bind.address.management:127.0.0.1}"/>
        </interface>
        <interface name="public">
            <inet-address value="${jboss.bind.address:127.0.0.1}"/>
        </interface>
    </interfaces>
    <socket-binding-group name="standard-sockets" default-interface="public" port-offset="${jboss.socket.binding.port-offset:0}">
```
-	management, public의 IP 주소 지정 변경 가능
- 바로 아래에 port-offset을 주어 포트번호 변경 (기본: 9990+0)
-	혹은 더 아래에 있는 http port 설정 자체를 바꿀 수 있음 (admin)

``bin/standalone.sh``로 기동 가능 (백그라운드 실행시 뒤에 &)
- 종료시 ./jboss-cli.sh --connect --controller={IP주소}:9990 shutdown
- 디폴트 포트가 9990이지만, offset을 주었다면 그만큼 더하면 된다. (=사용하는 포트로)

---

**2. domain mode**
- 여러 개의 서버 인스턴스를 그룹으로 묶어 관리
- ``bin/add-user.sh`` 로 관리자 계정 생성
- host.xml에서 관리자콘솔 페이지 port 변경 가능

``bin/domain.sh``에서 옵션을 주면 바로 실행 가능
- ./domain.sh -bmanagement 0.0.0.0 -b 0.0.0.0 -Djboss.domain.base.dir="~/wildfly/domain"

포트나 도메인 그룹 등, 수정할 부분이 있다면 ``host-slave.xml``, ``host-master.xml``, ``domain.xml``을 수정   
1) host-slave가 속한 group을 지정하고,
2) 설정한 socket-binding-group ref 에 따라 소켓 설정 추가 