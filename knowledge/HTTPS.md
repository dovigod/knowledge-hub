---
id: 019e8ce6-1c0a-75ee-acfc-9c5397c5635a
name: HTTPS
aliases:
  - HTTP Secure
  - HTTP over SSL
  - HTTP over TLS
  - HTTPS
  - HyperText Transfer Protocol Secure
  - SSL
  - TLS
updated_at: '2026-06-03T12:58:02.701Z'
summary: 'HTTP over TLS — the standard for encrypted, authenticated web traffic.'
sources:
  - 019e8ce5-afa5-74be-8636-3900cef4dbf2
  - 019e8d8e-31db-70ae-a1a5-4073d8b737a5
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# HTTPS

## Overview
HTTPS wraps HTTP in a TLS-encrypted channel, providing confidentiality, integrity, and server authentication (and optionally client authentication via mTLS).

> [!note] HTTPS = HTTP + [[TLS]]. The HTTP semantics are unchanged; TLS handles the secure channel underneath.

## Notes
- **Required for any credential transmission**, including [[Basic Authentication]], cookies, and bearer tokens.
- Protects the entire HTTP message (headers + body) from network observers — but not from the endpoints themselves or from anything that terminates TLS (load balancers, CDNs).
- Modern baseline: TLS 1.2+ with forward-secret cipher suites; TLS 1.3 preferred.
- The TLS handshake uses [[Asymmetric Encryption]] to authenticate the server (via its certificate) and to negotiate a shared session key; bulk traffic then uses symmetric encryption for performance.
- Server identity is established by an X.509 certificate signed by a trusted CA; the client validates the chain and the hostname before sending any application data.

## Examples

> [!example] Handshake at a glance
> 1. Client sends `ClientHello` (supported TLS versions, cipher suites, SNI hostname).
> 2. Server replies with `ServerHello` + certificate chain.
> 3. Both sides derive a shared session key (TLS 1.3: via ECDHE; earlier: RSA or DHE/ECDHE key exchange).
> 4. Encrypted HTTP traffic flows over the established session.

---

## 한국어

### 개요
HTTPS는 HTTP 통신을 TLS로 암호화한 채널로 감싸 기밀성(confidentiality), 무결성(integrity), 서버 인증(server authentication)을 제공한다. 필요하다면 mTLS로 클라이언트 인증까지 더할 수 있다.

> [!note] HTTPS = HTTP + [[TLS]]. HTTP의 의미 자체는 그대로이고, TLS가 그 아래에서 안전한 채널을 담당한다.

### 노트
- [[Basic Authentication]], 쿠키, bearer 토큰 등 **자격 증명을 전송할 때는 반드시 HTTPS가 필요**하다.
- HTTP 메시지 전체(헤더 + 본문)를 네트워크 관찰자로부터 보호하지만, 엔드포인트 자신이나 TLS를 종단(terminate)하는 구성요소(로드밸런서, CDN)로부터는 보호되지 않는다.
- 현대 기준: TLS 1.2+에 forward-secret 암호 스위트, TLS 1.3 권장.
- TLS 핸드셰이크는 [[Asymmetric Encryption]]을 사용해 서버를 인증(인증서 기반)하고 공유 세션 키를 협상한다. 이후의 대량 트래픽은 성능을 위해 대칭 암호화를 사용한다.
- 서버의 정체성은 신뢰된 CA가 서명한 X.509 인증서로 증명되며, 클라이언트는 애플리케이션 데이터를 보내기 전에 인증서 체인과 호스트명을 검증한다.

### 예시

> [!example] 핸드셰이크 한눈에 보기
> 1. 클라이언트가 `ClientHello`를 보낸다(지원 TLS 버전, 암호 스위트, SNI 호스트명).
> 2. 서버가 `ServerHello`와 인증서 체인으로 응답한다.
> 3. 양쪽이 공유 세션 키를 도출한다(TLS 1.3: ECDHE / 이전 버전: RSA 또는 DHE·ECDHE 키 교환).
> 4. 수립된 세션 위로 암호화된 HTTP 트래픽이 흐른다.

## Sources

- [[raw/conversations/019e8ce5-afa5-74be-8636-3900cef4dbf2|019e8ce5-afa5-74be-8636-3900cef4dbf2]]
- [[raw/conversations/019e8d8e-31db-70ae-a1a5-4073d8b737a5|019e8d8e-31db-70ae-a1a5-4073d8b737a5]]
