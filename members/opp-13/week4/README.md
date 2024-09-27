# iptables 총정리

## 1. iptables란?
Netfilter 모듈을 이용한 IP packet filter 설정 도구. 리눅스 커널 단에 있는 Netfilter를 CLI에서 쉽게 설정할 수 있도록 한다.

규칙의 집합인 체인을 테이블에 넣어 패킷을 필터링 한다. 

#자세한 원리는 #3에서 다룰것이다. "규칙의 집합은 체인이고 체인의 집합은 테이블이다." 이 정도로만 이해하고 넘어가도 iptables를 쓰는데는 괜찮다.

참조

IPv4의 경우 iptables를 사용하지만 IPv6는 ip6tables, ARP는 arptables, Ethernet frames의 경우 ebtables를 사용한다.

UFW같은 방화벽의 경우 iptables를 제어하여 패킷을 필터링한다.

Match Extensions 및 Target Extensions은 생략함

https://linux.die.net/man/8/iptables

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
-A, --append chain rule-specification
[chain]의 마지막에 [rule-specification] 추가

-D, --delete chain rule-specification
[chain]의 규칙 중 [rule-specification]과 일치하는 규칙 삭제

-D, --delete chain rulenum
[chain]의 규칙 순서 중 [rulenum]에 해당하는 규칙 삭제
참고로 [rulenum]는 0이 아닌 1부터 시작한다.

-I, --insert chain [rulenum] rule-specification
[chain]의 [rulenum]의 순서에 [rule-specification]를 추가
#예를 들어 [rulenum]가 1이면 제일 상단에 추가됨

-R, --replace chain rulenum rule-specification
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

-N, --new-chain chain
사용자 정의 체인을 생성한다. 이 경우 기존 체인의 이름과 곂치면 안된다.

-X, --delete-chain [chain]
사용자 정의 체인을 삭제한다.
#만약 [chain]이 없을 경우 해당 테이블의 모든 사용자 정의 체인을 삭제한다.
#삭제하려는 체인의 규칙은 모두 비어있어야 한다.
#삭제하려는 체인은 다른 체인에서 참조되면 안된다.

-P, --policy chain target
[chain]의 정책을 지정된 [target]으로 설정한다. 
#이 때 [chain]은 기본 제공된(사용자 정의가 아닌) 체인만 설정 가능하다.
#체인은 [target]이 될 수 없다.

-E, --rename-chain old-chain new-chain
[old-chain]의 이름을 [new-chain]으로 수정한다.
#테이블의 구조에는 영향이 가지 않는다.

-h
사용법을 제공한다.
</code>
</pre>

### 2.4 [PARAMETERS]
[rule-specification]을 구성하는데 사용한다.

<pre>
<code>
-p, --protocol [!] protocol
규칙 또는 확인할 packet의 프로토콜이다.
tcp, udp, icmp 또는 all 중 하나일 수 있으며 /etc/protocols에 있는 이름 또한 가능하다. 
#!를 사용하여 의미를 반전시킬 수 있다.
#숫자로 [protocol]을 지정할 수 있다. 이때 0은 전체 프로토콜을 의미한다.
#기본값은 0이다.

-s, --source [!] address[/mask]
출발지를 의미한다. [address]는 네트워크 이름, 호스트 이름(권장 x), ip주소(이 경우 mask 값도 포함 가능)을 넣을 수 있다.
#!를 사용하여 의미를 반전시킬 수 있다.
#--src도 가능하다.

-d, --destination [!] address[/mask]
목적지를 의미한다 문법은 -s와 같다.
#--dst도 된다.

-j, --jump target
규칙의 [target]을 정의한다. 즉 packet이 규칙에 맞을 경우 할일을 정의한다.
[target]은 사용자 정의 체인(이 규칙이 있는 체인 제외), 패킷의 운명을 즉시 결정하는 특수 내장 타겟 중 하나, 또는 확장일 수 있다.
#규칙에서 이 옵션과 -g 옵션이 사용되지 않는 경우 패킷에는 영향이 가지 않지만 규칙의 카운터는 증가한다.

-g, --goto chain
-j옵션과 비슷하지만 RETURN 시 현재 체인에서 처리하지 않고 -j를 호출한 체인에서 계속된다.

-i, --in-interface [!] name
packet이 수신된 인터페이스의 이름이다(INPUT, FORWARD, PREROUTING 체인에만 한정). 
#!를 사용하여 의미를 반전시킬 수 있다.
#인터페이스 이름이 +로 끝나면 이 이름으로 시작하는 모든 인터페이스를 의미한다. (eth+ -> eth0, eth1 ...)
#만약 이 옵션이 생략되면 모든 인터페이스의 이름으로 설정된다.

-o, --out-interface [!] name
packet이 전송될 인터페이스의 이름이다(FORWARD, OUTPUT, POSTROUTING 체인에만 한정). 
#-i와 사용법은 같다.

[!] -f, --fragment
단편화된 packet의 2번째 조각 이후의 packet만 참조한다. 
#!인자는 단편화되지 않은 packet 그리고 단편화된 packet의 첫번째 조각의 패킷에 대해 처리합니다.
#단편화된 패킷의 2번째 조각 부터는 포트(출발지/목적지) 및 ICMP 패킷인지 알 수 없는 경우가 있습니다.

-c, --set-counters PKTS BYTES
INSERT, APPEND, REPLACE 명령 중 규칙의 packet과 byte counters를 초기화 할 수 있게 합니다. 
</code>
</pre>


<br /><br />

참조

https://linux.die.net/man/8/iptables


<br /><br />

## 3. Iptables의 작동 원리

기본적으로 iptables의 체인은(기본 체인) Netfilter Hook에 의해 트리거 된다.

### 3.1 Netfilter Hook

#### Netfilter Hook의 종류
<pre>
<code>
NF_IP_PRE_ROUTING
ip_rcv() 함수와 ipv6_rcv() 함수에 존재한다.
라우팅 탐색 전에 적용된다.
PREROUTING 체인을 트리거 한다. 

NF_IP_LOCAL_IN
ip_local_deliver() 함수와 ip6_input() 함수에 존재한다. 
라우팅 후에 적용된다.
INPUT 체인을 트리거 한다. 

NF_IP_FORWARD
ip_forward() 함수와 ip6_forward() 함수에 존재한다.
포워딩 되는 모든 패킷은 라우팅 후 실행된다.
FORWARD 체인을 트리거 한다. 

NF_IP_LOCAL_OUT
__ip_local_out() 함수와 __ip6_local_out() 함수에 존재한다. 
로컬에서 생성된 모든 패킷에 대해 실행되는 첫번째 Hook이다.
OUTPUT 체인을 트리거 한다. 

NF_IP_POST_ROUTING
ip_output() 함수와 ip6_finish_output2() 함수에 존재한다.
포워딩 되거나 로컬에서 생성 후 전송되는 송신 패킷에 대하여 실행된다.
POSTROUTING 체인을 트리거 한다. 
</code>
</pre>

#### Low Level에서의 Netfilter Hook이 실행되는 방식
이 부분에서는 Netfilter Hook이 되는 과정과 hook으로 트리거된 체인이 어느 테이블을 참조하는지를 볼 것이다.

모든 Netfilter Hook이 되는 과정은 사실 커널을 다루지 않는 한 필요 없는지라 NF_IP_PRE_ROUTING을 예시로 설명을 진행하겠다.

만약 이런 부분을 전부 보고 싶다면 O'reily의 "리눅스 네트워크의 이해"를 정독할 것을 추천한다.

이전에 NF_IP_PRE_ROUTING은 ip_rcv()함수에 존재한다고 서술했다.
리눅스의 IPV4 프로토콜은 부팅 시에 초기화되고 ip_init 함수를 실행한다. 그 결과 packet_type 스트럭쳐 안의 ip_rcv 함수는 프로토콜의 헨들러 함수로 등록된다. '상위 프로토콜'에서 ETH_P_IP값으로 받아진 모든 이더넷 프레임은 ip_rcv 함수로 처리된다. 

참고로 아래는 유효한 이더넷 타입일 시 Protocol 별 Function handler이다. (h_proto > 1536일 경우)
|Protocol|Ethernet type|Function handler|
|---|--|---|
|ETH_P_IP|0x0800|ip_rcv, ic_bootp_recv|
|ETH_P_X25|0x0805|X25_lap_receive_frame|
|ETH_P_ARP|0x0806|arp_rcv|
|ETH_P_BPQ|0x08FF|bpq_rcv|
|ETH_P_DNA_RT|0x6003|dn_route_rcv|
|ETH_P_RARP|0x8035|ic_rarp_recv|
|ETH_P_8021Q|0x8100|vlan_skb_rcv|
|ETH_P_IPX|0x8137|ipx_rcv|
|ETH_P_IPV6|0x86DD|ipv6_rcv|
|ETH_P_PPP_DISC|0x8863|pppoe_disc_rcv|
|ETH_P_PPP_SES|0x8864|pppoe_rcv|

#프로토콜을 더 확인하고 싶다면 아래의 링크에서 확인할 수 있다.
https://sites.uclouvain.be/SystInfo/usr/include/linux/if_ether.h.html

#그리고 ETH_P_IP의 경우 ic_bootp_recv가 있는데 이 부분은 부팅 때 동적 IP설정을 위해서만 사용되고 설정을 받아오면 제거된다.
https://github.com/torvalds/linux/blob/master/net/ipv4/ipconfig.c#L989

아무튼 다시 ip_rcv 함수로 돌아가서 이 함수의 코드를 살펴보면 다음과 같다.

<pre>
<code>
/*
 * IP receive entry point
 */
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt,
	   struct net_device *orig_dev)
{
	struct net *net = dev_net(dev);

	skb = ip_rcv_core(skb, net);
	if (skb == NULL)
		return NET_RX_DROP;

	return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
		       net, NULL, skb, dev, NULL,
		       ip_rcv_finish);
}
</code>
</pre>
https://github.com/torvalds/linux/blob/master/net/ipv4/ip_input.c#L557-L572

보다시피 NF_HOOK 함수가 실행되고 그 후에  ip_rcv_finish가 실행된다. 이때 NF_HOOK함수는 NF_IP_PRE_ROUTING를 인자를 가지고 있고 따라서 PREROUTING 체인이 실행된다.

아래 이미지는 커널에서 호출되는 함수 및 netfilter 흐름도이다. 이 이미지는 2006년 기준 코드이며 현재 코드와 완전히 맞지 않다는 점을 확실히 해 두었으면 좋겠다.
#예를 들어 ip_finish_output 뒤에 NF_HOOK으로 NF_IP_POST_ROUTING이 되어 있는데 현재는 전혀 그렇지 않다는 것을 확인할 수 있다.

과거 코드

https://github.com/torvalds/linux/blob/b31e5b1bb53b99dfd5e890aa07e943aff114ae1c/net/ipv4/ip_output.c#L218

현재 코드

https://github.com/torvalds/linux/blob/master/net/ipv4/ip_output.c#L317

<img src="https://img-blog.csdn.net/20131011091321515" height="10%"/>

### 3.2 체인

[위에서 테이블 별 거치는 체인을 언급한적이 있다.](https://github.com/ByeongHunKim/Cloudclub-istio-skyline/tree/feature/opp-13/week4/members/opp-13/week4#%ED%85%8C%EC%9D%B4%EB%B8%94-%EB%B3%84-%EA%B1%B0%EC%B9%98%EB%8A%94-%EC%B2%B4%EC%9D%B8)

따라서 위의 Netfilter Hook으로 체인이 트리거 되면 설정된 각 체인을 거치며 알맞은 규칙을 탐색 후 맞는 부분이 있으면 해당 규칙을 적용하게 된다.

예를 들어 PREROUTING 체인이 트리거 되면 PREROUTING 체인의 테이블인 raw, mangle, nat(DNAT) 테이블을 차례로 거치며 규칙을 위에서 아래로 탐색한다. 탐색 중 규칙에 부합하는 경우가 있을 때 그 규칙에 맞는 처리를 하게 된다.


#### 예시

아래는 nat테이블의 체인별 규칙이다.
<pre>
<code>
$ sudo iptables -t nat -L -v -n;
# Warning: iptables-legacy tables present, use iptables-legacy to see them
Chain PREROUTING (policy ACCEPT 301K packets, 42M bytes)
 pkts bytes target     prot opt in     out     source               destination
   11   624 DOCKER     0    --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 31981 packets, 4403K bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER     0    --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 31981 packets, 4403K bytes)
 pkts bytes target     prot opt in     out     source               destination
 3148  201K MASQUERADE  0    --  *      !br-616bab02b220  192.168.49.0/24      0.0.0.0/0    #제일 먼저 탐색됨
    0     0 MASQUERADE  0    --  *      !docker0  172.17.0.0/16        0.0.0.0/0    #2번째
    0     0 MASQUERADE  6    --  *      *       192.168.49.2         192.168.49.2         tcp dpt:22    #3번째
    0     0 MASQUERADE  6    --  *      *       192.168.49.2         192.168.49.2         tcp dpt:2376  #4....
    0     0 MASQUERADE  6    --  *      *       192.168.49.2         192.168.49.2         tcp dpt:5000
    0     0 MASQUERADE  6    --  *      *       192.168.49.2         192.168.49.2         tcp dpt:8443
    0     0 MASQUERADE  6    --  *      *       192.168.49.2         192.168.49.2         tcp dpt:32443
</code>
</pre>

<img src="https://velog.velcdn.com/images/koo8624/post/6354d79a-b2f5-4eaf-ad54-f0db6abaf962/image.png" width="80%"/>
