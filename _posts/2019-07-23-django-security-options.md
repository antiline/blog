---
layout: post
title: "django 기본 보안 설정"
categories: development
tags: [django, security]
---

django 2.2 를 기준으로 설정하면 좋은 보안 옵션들에 대해 정리한다. 
내용을 보기 전에 [Security in Django][security-in-django] 와 [System check framework][system-check-framework] 를 읽어보면 좋다.

### TL;DR
- 전체 설정은 맨 아래에 있다.


## HOST
- [https://docs.djangoproject.com/en/2.2/ref/settings/#std:setting-ALLOWED_HOSTS][django-allowd-host]

```python
ALLOWED_HOSTS = ['exmaple.com']
```

`ALLOWED_HOSTS`를 지정하면 `Host`헤더를 확인한다. 서비스 인프라 구성상 불필요한 경우도 있지만, 가급적 접근 가능 도메인을 지정해서 관리해야 한다.
개발 환경에서는 불편함이 있을 수 있다. 환경별 Setting 파일을 분리해서 개발에서는 아래와 같이 설정하면 된다.

```python
ALLOWED_HOSTS = ['localhost', '127.0.01']
```

혹은 개발용 도메인을 지정해서 사용하면 좋다.

```python
ALLOWED_HOSTS = ['exmaple.dev']
```


## SSL/HTTPS
- [https://docs.djangoproject.com/en/2.2/topics/security/#ssl-https][django-ssl-https]

```python
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

HTTP로 요청이 들어오면 HTTPS로 redirect 한다. 
보통 nginx나 apache 혹은 AWS ELB가 대신 HTTPS 를 처리하기 때문에 관련 헤더를 `SECURE_PROXY_SSL_HEADER`에 설정하면 된다.
단순 redirect를 하는 것이기 때문에 `HTTP to redirect to HTTPS`를 django 보다는 nginx 나 apache 에서 하면 좋다.

### HSTS 
```python
SECURE_HSTS_SECONDS = 31536000  # 365 * 24 * 60 * 60
SECURE_HSTS_PRELOAD = True
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
```

`Strict-Transport-Security "max-age=31536000; includeSubdomains; preload"` 를 응답한다. 
이렇게 설정하면 브라우저가 HTTPS 연결이 아닌 경우에 접근을 차단한다. 사이트 오픈 후 적용한다면 `SECURE_HSTS_SECONDS`의 시간을 60초 정도로 작게 해서 테스트를 하고 서서히 늘려가면 좋다.
개발 환경에서는 인증서 문제가 있을 수 있기 때문에 개발 편의상 프로덕션 환경에만 적용하는것이 좋다.(쉽지 않지만, 개발환경에서도 올바른 인증서를 사용한다면 더욱 좋다.)


## X-Content-Type-Options
- [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options][django-x-content-type-options]

```python
SECURE_CONTENT_TYPE_NOSNIFF = True
```

`X-Content-Type-Options: nosniff` 헤더를 응답한다. MIME type을 브라우저가 추측하지 말고 지정한 형시으로만 파일을 사용하도록 한다.


## X-XSS-Proection
- [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-XSS-Protection][django-x-xss-protection]

```python
SECURE_BROWSER_XSS_FILTER = True
```

`x-xss-protection: 1; mode=block` 헤더를 응답한다. 브라우저에서 XSS 보호기능을 활성화 한다. XSS 공격이 감지되면 화면을 빈 페이지로 그린다.


## X-Frame-Options
- [https://docs.djangoproject.com/en/2.2/ref/clickjacking/#setting-x-frame-options-for-all-responses][django-x-frame-options]

```
`X_FRAME_OPTIONS = 'DENY'
```

`X-Frame-Options: deny` 헤더를 응답한다. 기본값은 `SAME_ORIGIN` 인데 `DENY` 로 하고 필요한 Endpoint만 `@xframe_options_sameorigin` 데코레이터를 통해 지원하는 것이 좋다.


## CSRF
- [https://docs.djangoproject.com/en-US/2.2/ref/csrf/#how-it-works][django-csrf]
- [https://developer.mozilla.org/ko/docs/Web/HTTP/Cookies][mdn-cookies]

SESSION 과 CSRF 토큰 쿠키의 설정이 동일하기 때문에 같이 설명한다.

#### HttpOnly
```python
SESSION_COOKIE_HTTPONLY = True
CSRF_COOKIE_HTTPONLY = True
```
HttpOnly 기본값이 True 이기 때문에 세팅하지 않아도 된다. 설정을 하면 JS 에서 쿠키에 접근하지 못한다.

#### Secure
```python
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```
Secure를 설정하면 HTTPS 프로토콜에서만 브라우저가 서버로 쿠키를 포함해서 요청한다.

#### SameSite
```python
SESSION_COOKIE_SAMESITE = 'Lax'
CSRF_COOKIE_SAMESITE = 'Lax'
```
기본값이 `Lax`이기 때문에 세팅하지 않아도 된다.

`Strict`이 더 좋은 옵션으로 보일 수 있지만 대부분의 경우는 그렇지 않다. 
`Strict`은 `cross-site`의 `top-level navigations`을 [지원][rfc6265-4.1.2.7]하지 않는다. 
다른 사이트에서 링크를 통해 사이트에 들어오는 경우에 쿠키를 보내지 않는다.


## CORS
- django-cors-headers 라이브러리를 사용한다.
- [https://github.com/ottoyiu/django-cors-headers/][django-csp]


```python
CORS_ORIGIN_ALLOW_ALL = False
CORS_ORIGIN_WHITELIST = ['https://sub.example.com']
CORS_URLS_REGEX = r'^/api/.*$'

CORS_ALLOW_CREDENTIALS = True 

CSRF_TRUSTED_ORIGINS = CORS_ORIGIN_WHITELIST
```

읽기 전용의 오픈된 자원이 아니면 `CORS_ORIGIN_ALLOW_ALL` 를 끄고 관리해야 한다. `CORS_ORIGIN_WHITELIST` 으로 도메인을 고정하거나 그 양이 많다면 `CORS_ORIGIN_REGEX_WHITELIST` 으로 관리할 수 있다. `CORS_URLS_REGEX` 으로 필요한 API URL만 추가 지정하는 것도 좋다.

인증이나 기타의 목적으로 쿠키를 보내기 위해서는 `access-control-allow-credentials: true`헤더를 응답해야 한다. `CORS_ALLOW_CREDENTIALS`를 True 로 하면 된다. 
설정을 해도 쿠키가 전송되지 않는다면 쿠키의 `samesite` 옵션 때문일 수 있다. `SESSION_COOKIE_SAMESITE`의 기본값이 `Lax` 이다. `Lax`를 `None`으로 바꾸지 말고, `SESSION_COOKIE_DOMAIN = '.example.com'`처럼 하위 도메인을 포함하도록 지정하면 좋다. 하위 도메인이 아니라면 쿠키로 인증을 하는 방법 대신에  토큰 사용하는 것을 권장한다.

`CSRF_TRUSTED_ORIGINS`는 `CSRF` 에 대한 설정이지만 여기서 설명한다. `Django` 는 `CSRF` 방어를 위해 `Referer`가 `Host` 헤더와 같은지 확인한다. 그리고 추가적으로 `CSRF_TRUSTED_ORIGINS`에 지정된 도메인들을 확인한다. `.example.com` 와 같이 모든 하위 도메인 지정도 가능하다.


## CSP
- django-csp 라이브러리를 사용한다.
- https://django-csp.readthedocs.io/en/latest/

```python
CSP_DEFAULT_SRC = ("'self'")
```

가능한 조합이 매우 많다. `CSP_DEFAULT_SRC = ("'self'")` 으로 설정하고 하나씩 정책을 변경해 가면서 적용하면 좋다.


---------------------------------------------
## settings/production.py
```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'csp.middleware.CSPMiddleware',
]

ALLOWED_HOSTS = ['exmaple.com']

SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

SECURE_HSTS_SECONDS = 31536000  # 365 * 24 * 60 * 60
SECURE_HSTS_PRELOAD = True
SECURE_HSTS_INCLUDE_SUBDOMAINS = True

SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True

CSP_DEFAULT_SRC = ("'self'")

CORS_ORIGIN_ALLOW_ALL = False
CORS_ORIGIN_WHITELIST = ['sub.example.com']
CORS_URLS_REGEX = r'^/api/.*$'

CORS_ALLOW_CREDENTIALS = True 

CSRF_TRUSTED_ORIGINS = CORS_ORIGIN_WHITELIST

SESSION_COOKIE_HTTPONLY = True
CSRF_COOKIE_HTTPONLY = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_SAMESITE = 'Lax'
SESSION_COOKIE_SAMESITE = 'Lax'
```


[security-in-django]: https://docs.djangoproject.com/en/2.2/topics/security/
[system-check-framework]: https://docs.djangoproject.com/en/2.2/ref/checks/#security
[django-allowd-host]: https://docs.djangoproject.com/en/2.2/ref/settings/#std:setting-ALLOWED_HOSTS
[django-ssl-https]: https://docs.djangoproject.com/en/2.2/topics/security/#ssl-https
[django-x-content-type-options]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options
[django-x-xss-protection]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-XSS-Protection
[django-x-frame-options]: https://docs.djangoproject.com/en/2.2/ref/clickjacking/#setting-x-frame-options-for-all-responses
[django-csrf]: https://docs.djangoproject.com/en-US/2.2/ref/csrf/#how-it-works
[django-csp]: https://django-csp.readthedocs.io/en/latest/
[mdn-cookies]: https://developer.mozilla.org/ko/docs/Web/HTTP/Cookies
[caniuse-smaesite]: https://caniuse.com/#feat=same-site-cookie-attribute
[rfc6265-4.1.2.7]: https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-03#section-4.1.2.7