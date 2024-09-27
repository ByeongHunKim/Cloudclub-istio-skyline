# iptables 총정리

## 1. iptables란?
Netfilter 모듈을 이용한 IP packet filter 설정 도구. 리눅스 커널 단에 있는 Netfilter를 CLI에서 쉽게 설정할 수 있도록 한다.

규칙의 집합인 체인을 테이블에 넣어 패킷을 필터링 한다. 

#자세한 원리는 #3에서 다룰것이다. "규칙의 집합은 체인이고 체인의 집합은 테이블이다." 이 정도로만 이해하고 넘어가도 iptables를 쓰는데는 괜찮다.

참조

IPv4의 경우 iptables를 사용하지만 IPv6는 ip6tables, ARP는 arptables, Ethernet frames의 경우 ebtables를 사용한다.

UFW같은 방화벽의 경우 iptables를 제어하여 패킷을 필터링한다.

## 2. iptables의 명령어 구조

<pre>
<code>
iptables -t [table] [Commands] [chain] [rule-specification] [options]

Commands 별 명령어는 다음과 같다.
iptables [-t table] -[AD] chain rule-specification [options]
iptables [-t table] -I chain [rulenum] rule-specification [options]
iptables [-t table] -R chain rulenum rule-specification [options]
iptables [-t table] -D chain rulenum [options]
iptables [-t table] -[LFZ] [chain] [options]
iptables [-t table] -N chain
iptables [-t table] -X [chain]
iptables [-t table] -P chain target [options]
iptables [-t table] -E old-chain-name new-chain-name
</code>
</pre>

### 2.1 [Targets]
규칙이 성립할 시 취할 행동 또는 이동할 사용자 정의 체인

기본 TARGETS
<pre>
<code>
ACCEPT
패킷을 통과

DROP
패킷을 버림

QUEUE
패킷을 userspace로 넘김
#queue handler에 따라 패킷이 userspace 프로세스로 넘어가는지 다름.
#2.4.x 및 2.6.x 커널(2.6.13까지)의 경우 ip_queue 큐 핸들러를 포함, 2.6.14 이상 커널은 nfnetlink_queue 큐 핸들러를 추가로 포함. QUEUE 타겟을 가진 패킷은 이 경우 '0'번 큐로 전송됩니다.
#NFQUEUE 타겟을 참조하는 것을 권장

#주로 application에 packet을 대기 (queue)시키는 데 사용 (검증 필요)

RETURN
현재 체인을 호출한 규칙으로 복귀 후 그 다음 규칙으로 재개

[Chain]
이동할 [Chain]을 지정 
#RETURN으로 원래 체인으로 복귀할 수 있다.
</code>
</pre>


### 2.2 [Tables]





<pre>
<code>
-t, --table [table]
[table]을 선택한다.
</code>
</pre>


#### 테이블 종류
<pre>
<code>
filter
-t 옵션으로 따로 [table]을 설정하지 않을 시 사용되는 기본 테이블이다. 주로 방화벽과 같은 패킷 제어에 사용한다.

nat
새로운 연결이 들어올 때 사용된다. 주로 IP를 변환하는데 사용된다.

mangle
주로 Packet Header를 수정하기 위해 사용된다.

raw
Connection tracking을 예외로 하기 위해 NONTRACK target과 같이 쓰는 테이블이다.

security
내부의 internal SELinux 보안 context marks를 패킷에 설정하는데 사용된다.

#man 페이지에는 없지만 시스템에는 있음
#참고로 커널 설정과 모듈 존재 유무에 따라 테이블이 있을 수 있고 없을 수 있음
https://linux.die.net/man/8/iptables
</code>
</pre>

#### 테이블 별 거치는 체인
<img src="https://lh6.googleusercontent.com/pLsVK7r6rcyyyJcwBgUXoV0PYjJCVQhJnOE7ZvuwAxThi4NqEjZrrc4d0Ikbl1kby8qRv6UejX46ie1rFuOjs58rfGJKNOde-FHqee7XFwV385bNczED0oNyFEwrcfkxFguuxEsh" width="100%"/>

#### filter
<pre>
<code>
INPUT
local socket으로 향하는 packet을 필터링

FORWARD
라우팅 되고있는 packet을 필터링

OUTPUT
local에서 생성된 packet을 필터링
</code>
</pre>

#### nat
<pre>
<code>
PREROUTING 
곧 들어오는 packet을 변경

OUTPUT 
local에서 생성된 packet을 라우팅 되기 전 변경

POSTROUTING 
나가려는 packet을 변경

#아래는 커널 2.4.34 버전 이후 지원하는 기능
INPUT
</code>
</pre>

#### mangle
<pre>
<code>
PREROUTING 
라우팅 되기 전 packet을 변경

OUTPUT 
local에서 생성된 packet을 라우팅 되기 전 변경

#아래는 커널 2.4.18 버전 이후 지원하는 기능

INPUT
들어오는 packet을 변경

FORWARD
라우팅 되고 있는 packet을 변경

POSTROUTING 
나가려는 packet을 변경
</code>
</pre>

#### raw
<pre>
<code>
PREROUTING 
모든 네트워크 인터페이스에서 오는 모든 packet

OUTPUT
local에서 생성된 packet 대상
</code>
</pre>

혹시 위 이미지가 안 보이면 아래 링크의 "Relationships Between Chains and Tables" 부분을 참조할 것

https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture

<br /><br />


자세한 내용은 아래를 참조

https://www.kangtaeho.com/66

https://velog.io/@koo8624/Linux-A-Deep-dive-into-iptables-and-Netfilter

### 2.3 [Commands]

<pre>
<code>
-A, --append [chain] [rule-specification]
[chain]의 마지막에 [rule-specification] 추가

-D, --delete [chain] [rule-specification]
[chain]의 규칙 중 [rule-specification]과 일치하는 규칙 삭제

-D, --delete [chain] [rulenum]
[chain]의 규칙 순서 중 [rulenum]에 해당하는 규칙 삭제
참고로 [rulenum]는 0이 아닌 1부터 시작한다.

-I, --insert [chain] [rulenum] [rule-specification]
[chain]의 [rulenum]의 순서에 [rule-specification]를 추가
#예를 들어 [rulenum]가 1이면 제일 상단에 추가됨

-R, --replace [chain] [rulenum] [rule-specification]
[chain]의 [rulenum]의 순서에 있는 규칙을 [rule-specification]로 변경
#만약 출발지/목적지 ip가 여러개이면 해당 명령어는 실패한다.

-L, --list [chain]
[chain]의 모든 규칙들을 조회
#만약 [chain]을 안 넣으면 모든 체인의 이름이 표시된다.
#[-t table]을 명시하지 않으면 기본 설정인 Filter 테이블의 규칙이 표시된다.
# 따라서 다른 테이블을 조회하려면 iptables -t,--table [table] -n -L을 사용, 일반적으로 -n옵션을 같이 쓴다.
#-Z 옵션과 같이 쓰는 것도 가능하다. iptables -t [table] -L -v -Z [체인]으로 사용한다. -Z 옵션 위치와는 관계 없이  현재 규칙을 확인 후 카운터를 초기화 한다.  

-F, --flush [chain]
[chain]의 모든 규칙을 삭제
#만약 체인이 지정되지 않으면 테이블의 모든 체인의 규칙을 삭제

-Z, --zero [chain]
[chain]의 모든 packet과 byte 카운터를 초기화(해당 규칙을 지난 packet 개수 및 byte)

-N, --new-chain [chain]
사용자 정의 체인을 생성한다. 이 경우 기존 체인의 이름과 곂치면 안된다.

-X, --delete-chain [chain]
사용자 정의 체인을 삭제한다.
#만약 [chain]이 없을 경우 해당 테이블의 모든 사용자 정의 체인을 삭제한다.
#삭제하려는 체인의 규칙은 모두 비어있어야 한다.
#삭제하려는 체인은 다른 체인에서 참조되면 안된다.

-P, --policy [chain] [target]
[chain]의 정책을 지정된 [target]으로 설정한다. 
#이 때 [chain]은 기본 제공된(사용자 정의가 아닌) 체인만 설정 가능하다.
#체인은 [target]이 될 수 없다.

-E, --rename-chain [old-chain] [new-chain]
[old-chain]의 이름을 [new-chain]으로 수정한다.
#테이블의 구조에는 영향이 가지 않는다.

-h
사용법을 제공한다.
</code>
</pre>





참조

https://linux.die.net/man/8/iptables
