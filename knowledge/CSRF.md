---
id: 019e8cf7-9d4f-77a5-b3e0-8e8957e29038
name: CSRF
aliases:
  - Cross-Site Request Forgery
  - XSRF
  - cross-site-request-forgery
updated_at: '2026-06-03T10:11:38.447Z'
summary: >-
  An attack that tricks an authenticated user's browser into submitting an
  unwanted request to a site where they are logged in, exploiting ambient
  session cookies.
sources:
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# CSRF

## Overview
Cross-Site Request Forgery (CSRF) is a web attack where a malicious site causes a victim's browser to issue an authenticated request to a target site, leveraging the cookies/credentials the browser automatically attaches. Because the request comes from the victim's session, the server cannot distinguish it from a legitimate action.

## Notes
- Defenses: anti-CSRF tokens (synchronizer pattern), `SameSite=Lax|Strict` cookies, double-submit cookies, custom headers + CORS, checking `Origin`/`Referer`.
- OAuth/OIDC use the `state` parameter as their CSRF defense — binding the callback to the original session.
- Modern browsers default cookies to `SameSite=Lax`, which mitigates many CSRF vectors but does not replace explicit tokens for sensitive endpoints.
- Not the same as XSS: XSS lets attacker code run *in* the victim's origin; CSRF runs from a *different* origin and abuses the browser's auto-auth.

---

## 한국어

### 개요
Cross-Site Request Forgery(CSRF)는 악성 사이트가 피해자의 브라우저로 하여금, 브라우저가 자동으로 첨부하는 쿠키/자격증명을 이용해 인증된 요청을 대상 사이트에 보내도록 유도하는 웹 공격입니다. 요청이 피해자의 세션에서 나오기 때문에 서버는 정상 동작과 구분할 수 없습니다.

### 노트
- 방어책: anti-CSRF 토큰(synchronizer 패턴), `SameSite=Lax|Strict` 쿠키, double-submit cookie, custom header + CORS, `Origin`/`Referer` 검사.
- OAuth/OIDC는 `state` 파라미터를 CSRF 방어 수단으로 사용하여 callback을 최초 세션과 묶습니다.
- 최신 브라우저는 쿠키를 기본 `SameSite=Lax`로 설정해 많은 CSRF 벡터를 완화하지만, 민감한 엔드포인트에서는 명시적 토큰을 대체하지 않습니다.
- XSS와 다름: XSS는 공격 코드가 피해자 origin *내부*에서 실행되고, CSRF는 *다른* origin에서 실행되어 브라우저의 자동 인증을 악용합니다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
