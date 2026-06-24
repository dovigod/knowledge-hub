---
id: 019e8ce6-1c0a-75ee-acfc-9c5397c5635a
name: HTTPS
aliases:
  - HTTP
  - HTTP Secure
  - HTTP over SSL
  - HTTP over TLS
  - HTTPS
  - HyperText Transfer Protocol Secure
  - REST over HTTP
  - SSL
  - SSL/TLS
  - TLS
  - Transport Layer Security
  - http
  - https
updated_at: '2026-06-07T08:55:27.452Z'
summary: 'HTTP over TLS — the standard for encrypted, authenticated web traffic.'
sources:
  - 019e8ce5-afa5-74be-8636-3900cef4dbf2
  - 019e8d8e-31db-70ae-a1a5-4073d8b737a5
  - 019e8daf-09b5-7403-af7c-3149b422f8c2
  - 019e8db1-624f-73be-ab60-735be949b701
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# HTTPS

## Overview
HTTPS wraps HTTP in a TLS-encrypted channel, providing confidentiality, integrity, and server authentication (and optionally client authentication via mTLS).

> [!note] HTTPS = HTTP + [[TLS]]. The HTTP semantics are unchanged; TLS handles the secure channel underneath.

The same "plain protocol + TLS wrapper" pattern appears across the ecosystem: `http://` → `https://`, `ws://` → `wss://`, `redis://` → `rediss://`. The trailing `s` is the conventional signal that TLS is being enforced at the transport layer.

> [!tip] Path vs. identity
> Network hijacks ([[BGP Hijacking]], DNS spoofing, ARP spoofing, DNAT rewrites inside a [[Docker]] host) attack **where** traffic goes. HTTPS attacks the next question: **who** the endpoint actually is. The route can be 100% subverted and the connection can still refuse to complete, because the attacker can't forge a CA-signed certificate for the real hostname.

## Notes
- **Required for any credential transmission**, including [[Basic Authentication]], cookies, and bearer tokens.
- Protects the entire HTTP message (headers + body) from network observers — but not from the endpoints themselves or from anything that terminates TLS (load balancers, CDNs).
- Modern baseline: TLS 1.2+ with forward-secret cipher suites; TLS 1.3 preferred.
- The TLS handshake uses [[Asymmetric Encryption]] to authenticate the server (via its certificate) and to negotiate a shared session key; bulk traffic then uses symmetric encryption for performance.
- Server identity is established by an X.509 certificate signed by a trusted CA; the client validates the chain and the hostname before sending any application data. This is the *last* line of defense when lower layers (DNS, BGP, [[iptables]] DNAT) have been compromised.
- The `s`-suffixed URL scheme convention (`https://`, `wss://`, `rediss://`) is an explicit signal to clients that TLS must be used — not just an optional upgrade. Managed cloud services (e.g. managed Redis) typically require the TLS variant.
- Self-signed or internal CA environments often need client-side tweaks (e.g. `rejectUnauthorized: false` in Node.js clients, `curl -k`, "Proceed anyway" in browsers). Each of these collapses HTTPS back to a path-trust model — disable certificate validation only in trusted networks, never in production code that talks to the public internet.
- For internal service-to-service traffic, **mTLS** (mutual TLS) makes certificate validation bidirectional and is what service meshes like Istio and Linkerd automate over [[gRPC]] — both peers prove identity, so a hijacked route can't impersonate either side.
- HTTPS is one transport choice among several in distributed systems: HTTP/HTTPS for external synchronous APIs, [[gRPC]] (typically over TLS via HTTP/2) for internal high-performance synchronous calls, and [[Message Queue]] systems (Kafka, RabbitMQ, SQS) for asynchronous event delivery. TLS is orthogonal to the communication pattern — each of these has its own secure-transport variant.

> [!warning] Why [[Basic Authentication]] *needs* HTTPS
> Basic Auth credentials are sent as `Base64(user:password)` — Base64 is encoding, not encryption, and anyone capturing the request can decode it instantly. The credential is only safe because the surrounding TLS tunnel hides the header from the network. Strip HTTPS away and Basic Auth is effectively plaintext.

## Examples

> [!example] Handshake at a glance
> 1. Client sends `ClientHello` (supported TLS versions, cipher suites, SNI hostname).
> 2. Server replies with `ServerHello` + certificate chain.
> 3. Both sides derive a shared session key (TLS 1.3: via ECDHE; earlier: RSA or DHE/ECDHE key exchange).
> 4. Encrypted HTTP traffic flows over the established session.

> [!example] TLS-enforcing URL schemes
> - `http://host:80` → plaintext HTTP
> - `https://host:443` → HTTP over TLS
> - `redis://host:6379` → plaintext Redis (RESP over TCP)
> - `rediss://user:password@host:6380/0` → Redis over TLS (typical port 6380)

> [!example] MyEtherWallet 2018 — TLS as the last line of defense
> 2018-04-24: attackers BGP-hijacked the IP range of AWS Route 53, then DNS-spoofed `myetherwallet.com` to a server in Russia. Users typed the real URL but landed on a phishing clone — losing roughly 215 ETH (~$150k).
> The clone could not obtain a CA-signed certificate for `myetherwallet.com`, so browsers raised a TLS warning. **Only users who clicked "Proceed anyway" lost funds.** L3 (BGP) and L7 (DNS) were both subverted; cert validation still held — until the human bypassed it.
> Lesson: HTTPS doesn't prevent route hijacking, it makes the hijack visible and refusable. The defense is only intact if `-k` / `rejectUnauthorized: false` / "Proceed anyway" stay off in real traffic.

> [!example] Cert validation demo (sketch)
> Three certs: a root CA (`MyTrustedRootCA`), a legit server cert for `bank.local` signed by that CA, and an attacker's self-signed cert (`issuer == subject == bank.local`). Two nginx servers behind identical-looking configs, only the mounted cert differs. Client uses `curl --resolve bank.local:443:<ip> --cacert ca.crt https://bank.local` — `--resolve` simulates the DNS/BGP hijack by overriding where the hostname points.
> Results: (1) legit IP + trusted CA → 200 OK. (2) attacker IP + trusted CA → `curl: (60) SSL certificate ... self-signed certificate` — connection refused, hijack neutralized. (3) attacker IP + `-k` → STOLEN, victim ignored the warning. Hostname is forgeable; CA-chained signatures aren't, because the attacker doesn't have the CA private key.

> [!tip] HTTPS vs gRPC vs MQ in real systems
> Large-scale systems usually combine all three: HTTPS for public-facing APIs (browsers, partners, mobile), [[gRPC]] for internal service-to-service synchronous calls (strong typing, HTTP/2 multiplexing, lower overhead), and a [[Message Queue]] for asynchronous event flow and service decoupling. Picking one isn't an either/or choice — they answer different questions (sync vs async, external vs internal, request/response vs event). Internal links typically run mTLS on top of whichever transport is used.

---

## 한국어

### 개요
HTTPS는 HTTP 통신을 TLS로 암호화한 채널로 감싸 기밀성(confidentiality), 무결성(integrity), 서버 인증(server authentication)을 제공한다. 필요하다면 mTLS로 클라이언트 인증까지 더할 수 있다.

> [!note] HTTPS = HTTP + [[TLS]]. HTTP의 의미 자체는 그대로이고, TLS가 그 아래에서 안전한 채널을 담당한다.

"평문 프로토콜 + TLS 래퍼" 패턴은 생태계 전반에 반복된다: `http://` → `https://`, `ws://` → `wss://`, `redis://` → `rediss://`. 끝의 `s`는 전송 계층에서 TLS를 강제한다는 관습적 신호다.

> [!tip] 경로(path) vs 신원(identity)
> 네트워크 하이재킹([[BGP Hijacking]], DNS 스푸핑, ARP 스푸핑, [[Docker]] 호스트 안에서의 DNAT 조작)은 "트래픽이 **어디로** 가는가"를 공격한다. HTTPS는 그다음 질문에 답한다 — "그 끝점이 **누구인가**". 경로가 100% 장악당해도 연결은 여전히 거부될 수 있다. 공격자는 진짜 호스트명에 대해 CA가 서명한 인증서를 위조할 수 없기 때문.

### 노트
- [[Basic Authentication]], 쿠키, bearer 토큰 등 **자격 증명을 전송할 때는 반드시 HTTPS가 필요**하다.
- HTTP 메시지 전체(헤더 + 본문)를 네트워크 관찰자로부터 보호하지만, 엔드포인트 자신이나 TLS를 종단(terminate)하는 구성요소(로드밸런서, CDN)로부터는 보호되지 않는다.
- 현대 기준: TLS 1.2+에 forward-secret 암호 스위트, TLS 1.3 권장.
- TLS 핸드셰이크는 [[Asymmetric Encryption]]을 사용해 서버를 인증(인증서 기반)하고 공유 세션 키를 협상한다. 이후의 대량 트래픽은 성능을 위해 대칭 암호화를 사용한다.
- 서버의 정체성은 신뢰된 CA가 서명한 X.509 인증서로 증명되며, 클라이언트는 애플리케이션 데이터를 보내기 전에 인증서 체인과 호스트명을 검증한다. 하위 계층(DNS, BGP, [[iptables]] DNAT)이 이�� 뚫린 상황에서 *마지막* 방어선 역할을 한다.
- `s`가 붙은 URL 스킴 관습(`https://`, `wss://`, `rediss://`)은 단순한 업그레이드 힌트가 아니라 "TLS를 반드시 사용하라"는 명시적 신호다. 클라우드 매니지드 서비스(예: managed Redis)에서는 사실상 TLS 변형이 필수인 경우가 많다.
- self-signed 인증서나 내부 CA 환경에서는 클라이언트 측 옵션 조정(예: Node.js의 `rejectUnauthorized: false`, `curl -k`, 브라우저의 "그래도 진행")이 필요할 수 있다. 이 옵션 하나하나가 HTTPS를 다시 "경로 신뢰 모델"로 되돌린다 — 인증서 검증을 끄는 것은 신뢰된 네트워크에 한해서만, 공용 인터넷과 통신하는 프로덕션 코드에서는 절대 금물.
- 내부 서비스 간 트래픽에는 **mTLS**(mutual TLS)로 인증서 검증을 양방향으로 만든다. Istio, Linkerd 같은 서비스 메시가 [[gRPC]] 위에서 이를 자동화 — 양쪽 모두 신원을 증명하므로 경로가 하이재킹돼도 어느 쪽도 가장할 수 없다.
- HTTPS는 분산 시스템에서 선택할 수 있는 여러 전송 방식 중 하나다: 외부 동기식 API에는 HTTP/HTTPS, 내부 고성능 동기 호출에는 [[gRPC]](보통 HTTP/2 위의 TLS), 비동기 이벤트 전달에는 [[Message Queue]] 시스템(Kafka, RabbitMQ, SQS). TLS는 통신 패턴과 직교(orthogonal)하며, 각각의 방식 모두 자체적인 보안 전송 변형을 가지고 있다.

> [!warning] [[Basic Authentication]]에 HTTPS가 *필수*인 이유
> Basic Auth의 자격 증명은 `Base64(user:password)` 형태로 전송된다 — Base64는 암호화가 아니라 인코딩이라서, 요청을 가로챈 누구든 즉시 디코드할 수 있다. 이 자격 증명이 안전한 유일한 이유는 바깥을 감싸는 TLS 터널이 헤더를 네트워크로부터 숨겨주기 때문이다. HTTPS를 벗기는 순간 Basic Auth는 사실상 평문이나 마찬가지다.

### 예시

> [!example] 핸드셰이크 한눈에 보기
> 1. 클라이언트가 `ClientHello`를 보낸다(지원 TLS 버전, 암호 스위트, SNI 호스트명).
> 2. 서버가 `ServerHello`와 인증서 체인으로 응답한다.
> 3. 양쪽이 공유 세션 키를 도출한다(TLS 1.3: ECDHE / 이전 버전: RSA 또는 DHE·ECDHE 키 교환).
> 4. 수립된 세션 위로 암호화된 HTTP 트래픽이 흐른다.

> [!example] TLS를 강제하는 URL 스킴들
> - `http://host:80` → 평문 HTTP
> - `https://host:443` → TLS 위의 HTTP
> - `redis://host:6379` → 평문 Redis (RESP over TCP)
> - `rediss://user:password@host:6380/0` → TLS 위의 Redis (보통 포트 6380)

> [!example] MyEtherWallet 2018 — 마지막 방어선으로서의 TLS
> 2018-04-24: 공격자가 AWS Route 53의 IP 대역을 BGP 하이재킹한 뒤, `myetherwallet.com`을 러시아 서버로 DNS 스푸핑했다. 사용자들은 진짜 URL을 입력했지만 가짜 피싱 사이트에 도달 — 약 215 ETH(~$150k) 탈취.
> 가짜 사이트는 `myetherwallet.com`에 대한 CA 서명 인증서를 얻을 수 없었기 때문에 브라우저가 TLS 경고를 띄웠다. **"그래도 진행"을 누른 사람만 돈을 잃었다.** L3(BGP)와 L7(DNS)가 모두 뚫렸지만, 인증서 검증은 끝까지 버텼다 — 사람이 직접 우회하기 전까지는.
> 교훈: HTTPS는 경로 하이재킹을 *막는* 게 아니라, 하이재킹을 *드러내고 거부할 수 있게* 한다. 실제 트래픽에서 `-k` / `rejectUnauthorized: false` / "그래도 진행"이 꺼져 있어야만 방어선이 유효하다.

> [!example] 인증서 검증 데모 (스케치)
> 인증서 3종: 루트 CA(`MyTrustedRootCA`), CA가 서명한 정품 `bank.local` 서버 인증서, 그리고 공격자의 자체서명 인증서(`issuer == subject == bank.local`). 외형이 똑같은 nginx 서버 두 대 — 마운트된 인증서만 다르다. 클라이언트는 `curl --resolve bank.local:443:<ip> --cacert ca.crt https://bank.local` 사용 — `--resolve`가 "호스트명이 가리키는 IP를 바꿔치기"해서 DNS/BGP 하이재킹을 흉내낸다.
> 결과: (1) 정품 IP + 신뢰 CA → 200 OK. (2) 공격자 IP + 신뢰 CA → `curl: (60) SSL certificate ... self-signed certificate` — 연결 거부, 하이재킹 무력화. (3) 공격자 IP + `-k` → STOLEN, 피해자가 경고를 무시한 경우. 호스트명은 위조 가능하지만 CA 체인 서명은 위조 불가능 — 공격자가 CA 개인키를 갖고 있지 않기 때문.

> [!tip] 실제 시스템에서의 HTTPS vs gRPC vs MQ
> 대규모 시스템은 보통 세 가지를 함께 쓴다: 외부 노출용 API(브라우저, 파트너, 모바일)에는 HTTPS, 내부 서비스 간 동기 호출에는 [[gRPC]](강타입, HTTP/2 멀티플렉싱, 낮은 오버헤드), 비동기 이벤트 흐름과 서비스 결합도 감소에는 [[Message Queue]]. 셋 중 하나를 고르는 양자택일 문제가 아니라, 각자 다른 질문(동기 vs 비동기, 외부 vs 내부, 요청/응답 vs 이벤트)에 답하는 도구다. 내부 링크에는 어떤 전송이든 위에 mTLS를 얹는 게 보통.

## Sources

- [[raw/conversations/019e8ce5-afa5-74be-8636-3900cef4dbf2|019e8ce5-afa5-74be-8636-3900cef4dbf2]]
- [[raw/conversations/019e8d8e-31db-70ae-a1a5-4073d8b737a5|019e8d8e-31db-70ae-a1a5-4073d8b737a5]]
- [[raw/conversations/019e8daf-09b5-7403-af7c-3149b422f8c2|019e8daf-09b5-7403-af7c-3149b422f8c2]]
- [[raw/conversations/019e8db1-624f-73be-ab60-735be949b701|019e8db1-624f-73be-ab60-735be949b701]]
- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
