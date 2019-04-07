---
layout: post
title: "네이버 블로그에 연결한 개인도메인에 HTTPS 적용하기"
categories: development
tags: [https, naver, blog, custom-domain]
---


네이버 블로그는 개인 도메인 연결을 지원하고 있지만, HTTPS 연결은 지원하고 있지 않다.

## 그러면 네이버 블로그를 사용하면서 개인도메인을 연결하면서 무료로 HTTPS 를 사용할 수 는 없을까?

[CloudFlare][cloud-flare] 를 사용하면 가능하다. CloudFlare 는 요즘 급성장하고 있는 CDN 회사이다. 수익구조가 매우 궁금한데, 무료로 CDN 을 제공하고 있다. CDN 뿐만 아니라 무료로 DNS 및 SSL 인증서도 발급 및 적용해 주고 있어 사이트 통합 관리가 편하다.
### 간단하게 정리하면

1. CloudFlare 에 DNS 와 캐시서버를 셋팅
2. CloudFlare 가 발급한 SSL 인증서를 사용하도록 함
3. 캐싱이 되면 페이지 갱신이 늦어지니까 해당 도메인 하위모든 URL 을 Bypass 설정

## 따라 해보자

1. Naver 블로그 관리 → 기본 설정 → 기본 정보 관리 → 블로그 주소 에서 개인도메인을 설정한다. 

    ![](/assets/posts/how-to-enable-https-on-a-naver-blog-having-custom-domain/9ab5c112-bef2-4559-9b9c-558603c5dc7e.png)

2. CloudFlare 에 가입한다.
    - [https://www.cloudflare.com][cloud-flare]
3. 도메인의 DNS 서버를 CloudFlare 로 지정한다.

    ![](/assets/posts/how-to-enable-https-on-a-naver-blog-having-custom-domain/f3552f63-df57-4f1c-9d27-6770de63178a.png)

4. CloudFlare에 A레코드로 네이버 블로그의 주소를 지정한다.
    - 네이버 블로그의 IP 주소: 125.209.214.79

        ![](/assets/posts/how-to-enable-https-on-a-naver-blog-having-custom-domain/24a09431-33f6-4936-a32c-b272843fe3a6.png)

        ![](/assets/posts/how-to-enable-https-on-a-naver-blog-having-custom-domain/102c6193-0ade-4dd2-a08d-43451e2bdffb.png)

        - A 레코드로 개인도메인을 125.209.214.79 으로 지정하고 캐싱 및 SSL 사용이 되도록 구름 모양이 주황색이 되도록 설정한다.
5. CloudFlare에 HTTPS 설정을 한다.
    - Crypto → SSL을 Full 로 지정한다.

        Full (strict)으로 사용하는게 보안에 좋지만, 사용할 수 없다. Full 로 사용하자.

        ![](/assets/posts/how-to-enable-https-on-a-naver-blog-having-custom-domain/7da86655-b369-4d8f-a2b3-23bad8e3bb8a.png)

    - Crypto → Always Use HTTPS 을 On으로 변경한다.

        HTTP 로 접근하면 HTTPS 로 자동으로 redirect 시키도록 Always Use HTTPS 셋팅을 킨다.

        ![](/assets/posts/how-to-enable-https-on-a-naver-blog-having-custom-domain/149e262d-a467-46a3-a931-085e6a2e50ce.png)

6. CloudFlare에 캐시 설정을 한다.
    - Caching → Page Rules 에 규칙을 Bypass 로 추가한다.

        CloudFlare 가 해당 도메인의 모든 페이지를 캐싱하지 않도록 하자. 그래야 글 변경 시 바로바로 반영된다. 개인도메인의 모든 하위 URL(*) 을 Bypass 로 지정한다.

        ![](/assets/posts/how-to-enable-https-on-a-naver-blog-having-custom-domain/a3631d95-d982-4192-94f2-56a06e5c9460.png)

    - Caching → Browser Cache Expiration을 Respect Existing Headers로 변경한다.

        CloudFlare 가 브라우저에 응답시 캐시 제어를 Origin 서버의 값을 전달 하도록 하자. 위에서 진행했던 Bypass 지정을 하면 일반적으로 페이지 캐싱을 하지 않고 항상 Origin 에 요청하고 그대로 응답한다. 그런데 CloudFlare도 그렇게 동작하는지 확인해보지 않아서 Browser Cache Expiration 을 추가 설정한다.

        ![](/assets/posts/how-to-enable-https-on-a-naver-blog-having-custom-domain/bfb324d9-c0ff-40fb-8aa5-f4cddbfe9366.png)

7. 설정은 끝났다. 적용에 시간이 걸릴 수 있으니 조금 기다렸다가 개인 도메인으로 접속해보자. 그러면 멋진 초록색 SSL 인증 마크를 볼 수 있다.

    ![](/assets/posts/how-to-enable-https-on-a-naver-blog-having-custom-domain/34cb371e-75cf-46a7-8566-4ed7c910182b.png)


[cloud-flare]: https://www.cloudflare.com
