---
layout: post
title: "django 2.2 에서 PyMySQL 사용시 ImproperlyConfigured 예외 발생"
categories: development
tags: [django, mariadb, PyMySQL]
---

개인 프로젝트로 [bootstrap-django][bootstrap-django] 를 만들고 있다. `django` 에서 가장 잘 지원해주는 `postgresql`를 사용하면 간편하나 postgresql 보다는 `mariadb` 선호하기 때문에 mariadb 를 사용 하려고 했다. 지금 했던 프로젝트에서는 DB client로 `mysqlclient` 를 사용했는데 이게 c 로 작성된 library 라서 설치도 까다롭고 OS 나 버전업에 신경써야 할 것들이 많았다. 그래서 이 프로젝트에서는 순수 Python 으로 작성된 `PyMySQL` 을 써보려고 했다. 셋팅을 하고 테스트를 하니 아래와 같은 예외가 발생했다.

```
django.core.exceptions.ImproperlyConfigured: mysqlclient 1.3.13 or newer is required; you have 0.9.3.
```

django 2.1 에서는 문제가 없었는데, django 2.2 에서만 발생하는 오류였다. 메세지를 보니 mysqlclient 를 1.3.13 으로 업데이트 하라고 했다. 0.9.3은 PyMySQL 의 버전이다. PyMySQL 은 django 가 official 로 지원하는 library 가 아니다. 그래서 PyMySQL 이 mysqlclient 인것 처럼 보이게 `pymysql.install_as_MySQLdb()` 을 통해서 기본 DB 모듈을 변경한다.

```python
def install_as_MySQLdb():
    """
    After this function is called, any application that imports MySQLdb or
    _mysql will unwittingly actually use pymysql.
    """
    sys.modules["MySQLdb"] = sys.modules["_mysql"] = sys.modules["pymysql"]
```

해당 오류발생 코드를 보니 역시나 버젼을 확인하고 예외를 날리고 있었다. git log를 보니 이번에 새로 추가된 코드가 아니라 기존에도 계속해서 버전을 올리면서 사용되던 코드였다.
```python
version = Database.version_info
if version < (1, 3, 13):
    raise ImproperlyConfigured('mysqlclient 1.3.13 or newer is required; you have %s.' % Database.__version__)
```

PyMySQL이 전에 어떻게 동작했나 코드를 좀 더 보니 아래와 같이 `1.3.12` 로 보이게 해서 예외를 회피하고 있었다. 
```python
# we include a doctored version_info here for MySQLdb compatibility
version_info = (1, 3, 12, "final", 0)
```

그래서 버전만 올리면 될 것 같아서 PyMySQL 의 이슈목록에 가보니 누가 먼저 1.3.12를 업데이트 해달라고 했다([https://github.com/PyMySQL/PyMySQL/issues/790][pymysql-issue-790]). 그런데 답변이 버전만 문제가 아니라 django 2.2 에서 mysqlclient 와 다르게 동작하는 부분 때문에 추가 오류가 발생한다고 했다. mysqlclient 는 bytes를 응답하는데 PyMySQL 은 str을 응답한다. 그래서 기존에는 `force_text`라는 함수를 통해 둘다 자연스럽게 지원할 수 있었지만 
```python
def last_executed_query(self, cursor, sql, params):
    return force_text(getattr(cursor, '_last_executed', None), errors='replace')
```

[https://github.com/django/django/blame/0f0d1cd7722238158011c7717d410fae37513ceb/django/db/backends/mysql/operations.py#L140][django-code-01]
의 코드로 인해 django 2.2 이후부터는 오류가 발생한다고 한다.
```python
def last_executed_query(self, cursor, sql, params):
    query = getattr(cursor, '_executed', None)
    if query is not None:
        query = query.decode(errors='replace')
    return query
```
 그래서 그걸 누가 또 django 에 issue 로 올렸다([https://code.djangoproject.com/ticket/30380][django-issue-30380]). 

이슈의 첫 답변에서 우리는 PyMySQL 을 공식 지원하지 않기 때문에 코드를 수정할 수 없다고 하다가 사람들이 계속 지원해달라고 요청해서 해당 commit 을 revert 하기로 했다. master 에는 merge 가 되었는데 [2.2.1][django-release-2-2-1] 에는 포함이 안되었다. 

2019-06-03에 릴리즈 예정인 [2.2.2][django-release-2-2-2] 노트에도 없었다(2019-05-28에 확인). 며칠 안남았으니 2.2.2에도 포함이 안될 것 같고, 2.2.3에는 될지 잘 모르겠다.

어쨌든 프로젝트는 당장 진행하고 싶어서 mysqlclient 로 우회하고 나중에 다시 PyMySQL 을 시도해 보려고 한다.

[bootstrap-django]: https://github.com/antiline/bootstrap-django
[pymysql-issue-790]: https://github.com/PyMySQL/PyMySQL/issues/790
[django-code-01]: https://github.com/django/django/blame/0f0d1cd7722238158011c7717d410fae37513ceb/django/db/backends/mysql/operations.py#L140
[django-issue-30380]: https://code.djangoproject.com/ticket/30380
[django-release-2-2-1]: https://docs.djangoproject.com/en/2.2/releases/2.2.1/
[django-release-2-2-2]: https://docs.djangoproject.com/en/2.2/releases/2.2.2/