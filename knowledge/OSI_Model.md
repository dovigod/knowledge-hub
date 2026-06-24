---
id: 019ea11d-90bf-70bb-9018-fa68c0741e5b
name: OSI Model
aliases:
  - 7계층
  - OSI
  - OSI 7 Layer
  - OSI 7-layer model
  - OSI 7계층
  - OSI Reference Model
  - OSI model
updated_at: '2026-06-07T08:53:31.172Z'
summary: >-
  Seven-layer conceptual model that describes how network protocols communicate,
  from physical signaling (L1) to application data (L7).
sources:
  - 019e8d52-83e5-72d1-a0b2-281e58439695
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# OSI Model

## Overview

The OSI (Open Systems Interconnection) Model is a conceptual framework standardized by ISO that divides network communication into seven distinct layers, each with specific responsibilities. While real-world stacks (like TCP/IP) don't map cleanly to all seven, the model remains the lingua franca for discussing where a protocol or security concern operates.

> [!example] The seven layers
> L1 Physical → L2 Data Link → L3 Network → L4 Transport → L5 Session → L6 Presentation → L7 Application

## Notes

- **L1 Physical**: cables, radio, raw bits
- **L2 Data Link**: Ethernet, MAC addresses, switches, bridges (e.g. [[Docker]]'s `docker0`), `veth` pairs (virtual NICs carrying Ethernet frames)
- **L3 Network**: [[IP]], routing, ICMP, firewalls, iptables NAT (DNAT/MASQUERADE rewriting IP addresses)
- **L4 Transport**: [[TCP]], [[UDP]], port numbers — stateful filtering, `bind()`/LISTEN, port mapping's "port" half (`-p 8080:80`)
- **L5 Session**: session establishment (rarely a distinct layer in practice; absorbed into L4 in TCP/IP)
- **L6 Presentation**: encryption, serialization — TLS lives here (often folded into L7)
- **L7 Application**: HTTP, [[SSH]], DNS, gRPC, nginx, curl — content-aware policies
- **Security implications**:
  - L3/L4 filtering = traditional firewalls
  - L7 filtering = WAF, API gateway, [[Cilium]] policies
- Related: [[TCP/IP Model]], [[Cilium]], [[Firewall]], [[Docker]], [[Container Networking]]

> [!tip] Mnemonic
> "Please Do Not Throw Sausage Pizza Away" — Physical, Data Link, Network, Transport, Session, Presentation, Application.

## Examples

> [!example] Mapping [[Docker]] container networking to OSI
> Tracing a `docker run -p 8080:80 nginx` request from a client to the container, layer by layer:
> - **L7 Application**: nginx, HTTP request/response, `curl`, status `200 OK`
> - **L6 Presentation**: plain HTTP → almost nothing; with HTTPS this is where TLS encryption sits
> - **L5 Session**: collapsed into L4 in the TCP/IP world
> - **L4 Transport**: ports `80` / `8080`, `bind()`, TCP LISTEN, the "port" half of port mapping (segments, port numbers)
> - **L3 Network**: container IP `192.168.215.2`, gateway, iptables NAT rules (`DNAT` for inbound, `MASQUERADE` for outbound) — packets, IP addresses
> - **L2 Data Link**: `veth` pair (virtual NIC carrying Ethernet frames), `docker0` bridge (virtual switch), MAC addresses — frames
> - **L1 Physical**: the `veth` virtual "cable" itself — raw bits

> [!note] Key device classifications
> - **`veth` pair** = virtual NIC. Spans L1 (the "cable") and L2 (carries MAC-addressed frames). The `link/ether` line in `ip addr` is the L2 evidence.
> - **`docker0`** = L2 device (bridge/switch), but also holds an IP and acts as the container's L3 default gateway.
> - **iptables NAT** straddles L3 (rewrites IP) and L4 (rewrites port). `-p 8080:80` is pure L4 mapping.

> [!example] Encapsulation journey
> Sending: wrap top-down — L7 HTTP → L4 TCP segment + port → L3 IP packet → L2 Ethernet frame + MAC → L1 bits.
> Mid-path: NAT rewrites the L3 destination IP and L4 destination port; `docker0` forwards based on L2 MAC.
> Receiving: unwrap bottom-up, layer by layer.

> [!warning] What does *not* fit on the OSI map
> - **PID namespace**: OS-level process isolation, not networking.
> - **cgroups**: resource quotas (CPU/memory), not networking.
> - **NET namespace**: not a single layer — a *container* that clones the L2–L4 stack wholesale. This is why two containers can both bind port `80` (L4) without conflict: each has its own port table.
> The split is two-pronged: networking parts live inside OSI; isolation primitives live outside it.

> [!tip] Path-hijacking attacks mapped to OSI
> "Whoever owns the path owns the traffic." Each attack lives at a specific layer:
> - **ARP spoofing** — L2 (forge gateway MAC)
> - **Rogue DHCP** — L2/L3
> - **iptables DNAT hijack** — L3+L4 (rewrite destination IP/port for in-host traffic)
> - **DNS spoofing/hijacking** — L7
> - **BGP hijacking** — L3 (route announcements; nation-state scale; cf. [[MyEtherWallet 2018]] for a real incident)
> - **Evil Twin AP** — L1/L2
> Defense splits into: (a) prevent path capture (least privilege, NetworkPolicy, RPKI, DNSSEC) and (b) make capture useless (TLS / mTLS at L6 — the certificate check is the final line, as MEW victims who ignored the browser warning learned the hard way).

---

## 한국어

### 개요

OSI(Open Systems Interconnection) 모델은 ISO에서 표준화한 개념적 프레임워크로, 네트워크 통신을 7개의 명확한 계층으로 나누며 각 계층은 특정 책임을 갖는다. 실제 스택(TCP/IP 같은)이 7개 계층에 완벽히 매핑되지는 않지만, 프로토콜이나 보안 관심사가 어디에서 동작하는지 논의할 때의 공용어로 남아 있다.

> [!example] 7개 계층
> L1 물리 → L2 데이터 링크 → L3 네트워크 → L4 전송 → L5 세션 → L6 표현 → L7 응용

### 노트

- **L1 물리**: 케이블, 무선, 원시 비트
- **L2 데이터 링크**: Ethernet, MAC 주소, 스위치, 브리지([[Docker]]의 `docker0` 등), `veth` pair(이더넷 프레임을 나르는 가상 NIC)
- **L3 네트워크**: [[IP]], 라우팅, ICMP, 방화벽, iptables NAT(DNAT/MASQUERADE — IP 주소 갈아끼우기)
- **L4 전송**: [[TCP]], [[UDP]], 포트 번호 — 상태 기반 필터링, `bind()`/LISTEN, 포트 매핑의 '포트' 부분(`-p 8080:80`)
- **L5 세션**: 세션 수립 (실제로는 별개 계층이 드묾; TCP/IP에선 L4에 흡수)
- **L6 표현**: 암호화, 직렬화 — TLS가 여기 (보통 L7에 포함됨)
- **L7 응용**: HTTP, [[SSH]], DNS, gRPC, nginx, curl — 컨텐츠 인식 정책
- **보안 시사점**:
  - L3/L4 필터링 = 전통 방화벽
  - L7 필터링 = WAF, API 게이트웨이, [[Cilium]] 정책
- 관련: [[TCP/IP Model]], [[Cilium]], [[Firewall]], [[Docker]], [[Container Networking]]

> [!tip] 암기법
> "Please Do Not Throw Sausage Pizza Away" — Physical, Data Link, Network, Transport, Session, Presentation, Application.

### 예시

> [!example] [[Docker]] 컨테이너 네트워킹을 OSI에 매핑
> `docker run -p 8080:80 nginx` 요청이 클라이언트→컨테이너로 흐르는 과정을 계층별로:
> - **L7 응용**: nginx, HTTP 요청/응답, `curl`, `200 OK`
> - **L6 표현**: 평문 HTTP면 거의 없음. HTTPS면 TLS 암호화가 여기
> - **L5 세션**: TCP/IP 세계에선 L4에 흡수
> - **L4 전송**: 포트 `80` / `8080`, `bind()`, TCP LISTEN, 포트 매핑의 '포트' 쪽(세그먼트, 포트번호)
> - **L3 네트워크**: 컨테이너 IP `192.168.215.2`, 게이트웨이, iptables NAT 규칙(들어올 때 `DNAT`, 나갈 때 `MASQUERADE`) — 패킷, IP 주소
> - **L2 데이터링크**: `veth` pair(이더넷 프레임을 나르는 가상 NIC), `docker0` 브리지(가상 스위치), MAC 주소 — 프레임
> - **L1 물리**: `veth`의 가상 '랜선' 자체 — 원시 비트

> [!note] 핵심 장비 분류
> - **`veth` pair** = 가상 NIC. L1(랜선 비유)과 L2(MAC 박힌 프레임을 다룸)에 양다리. `ip addr`의 `link/ether` 줄이 L2 증거.
> - **`docker0`** = L2 장비(브리지/스위치)지만 IP도 가져서 컨테이너의 L3 기본 게이트웨이 역할도 함.
> - **iptables NAT** = L3(IP 재작성) + L4(포트 재작성) 양다리. `-p 8080:80`은 순수 L4 매핑.

> [!example] 패킷 캡슐화 여정
> 보낼 때: 위→아래로 봉투 싸기 — L7 HTTP → L4 TCP 세그먼트+포트 → L3 IP 패킷 → L2 Ethernet 프레임+MAC → L1 비트.
> 경로 중간: NAT이 L3 dst IP와 L4 dst port를 갈아끼움. `docker0`는 L2 MAC을 보고 전달.
> 받을 때: 아래→위로 봉투 벗기기.

> [!warning] OSI에 안 들어가는 것
> - **PID namespace**: OS 프로세스 격리, 네트워킹 아님.
> - **cgroups**: 자원 제한(CPU/메모리), 네트워킹 아님.
> - **NET namespace**: 단일 계층이 아니라 L2~L4 스택을 통째 복제하는 *그릇*. 그래서 같은 80번 포트(L4)도 충돌 없이 쓸 수 있다 — 각자의 포트 테이블이 따로니까.
> 두 갈래: 네트워크 부품(OSI ��)과 격리 부품(OSI 밖).

> [!tip] 경로 장악 공격의 OSI 매핑
> '경로를 쥔 자가 트래픽을 쥔다.' 각 공격이 사는 계층:
> - **ARP 스푸핑** — L2 (게이트웨이 MAC 위조)
> - **Rogue DHCP** — L2/L3
> - **iptables DNAT 하이재킹** — L3+L4 (호스트 내부 트래픽의 dst IP/port 갈아끼움)
> - **DNS 스푸핑/하이재킹** — L7
> - **BGP 하이재킹** — L3 (경로 광고; 국가급 규모; 실제 사례는 [[MyEtherWallet 2018]])
> - **Evil Twin AP** — L1/L2
> 방어는 두 갈래: (a) 경로 장악 자체 막기(최소권한, NetworkPolicy, RPKI, DNSSEC), (b) 장악당해도 무의미하게(L6의 TLS/mTLS — 인증서 검증이 마지막 방어선. MEW에서 브라우저 경고를 무시한 사람만 털렸던 그 지점).

## Sources

- [[raw/conversations/019e8d52-83e5-72d1-a0b2-281e58439695|019e8d52-83e5-72d1-a0b2-281e58439695]]
- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
