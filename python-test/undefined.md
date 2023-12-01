# ---

API를 개발하고 처음에는 Postman을 이용해 테스트를 진행했다. postman에서 스크립트를 작성한 후, csv 파일을 읽어서 연속 테스트도 진행을 했다. 테스트 진행하다 보니, 예외를 테스트 하고 결과 값을 확인하는 부분에서 힘든 부분이 있었다.

FAST API 공식 문서에서 테스팅 방법을 참고하여 API 테스트를 작성했다.

> [https://fastapi.tiangolo.com/tutorial/testing/#testing-extended-example](https://fastapi.tiangolo.com/tutorial/testing/#testing-extended-example)

## 테스트를 위한 DB 및 기본 준비

기존에 있는 디비 설정을 사용하면 테스틀르 진행하면서 데이터가 꼬이거나 전부 삭제될 수 있기 때문에 테스트용 데이터베이스를 생성해서 사용했다. 진행중인 프로젝트 루트에 test 디렉토리를 만들어 테스트를 작성했다.

### 기본 세팅

테스트는 pytest 모듈을 이용하기에 설치가 필요하다.

```bash
$ pip install pytest
```

테스트 실행은 프로젝트 루트에서 `pytest` 명령어를 실행하면 디렉토리 내 `test_*.py` 또는 `*_test.py` 파일을 모두 실행한다. 따라서 반드시 테스트 파일은 test로 시작하거나 test로 끝나도록 생성해야 한다. IDE에서는 파이썬 작업 디렉토리를 자동으로 잡아주기 때문에, 만약 별도의 설정이 필요하다면 환경변수 `PYTHONPATH`를 먼저 세팅해야 한다

```bash
# pytest 실행
$ export PYTHONPATH=[python_project_path]
$ pytest
```

### 테스트용 세션 객체 만들기

FAST Api에서 DB 를 사용할 때, SqlAlchemy를 사용하기 위해서 `create_engine` 함수 및 `sessionmaker` 함수로 세션 객체를 만들어서 사용했다. 이를 테스트에서 사용하기 위해서 테스트 디비 주소 및 설정으로 생성해준다.

```python
SQLALCHEMY_DATABASE_URL = "mysql+pymysql://kristoff:1234@localhost:3306/test_db"

engine = create_engine(SQLALCHEMY_DATABASE_URL)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```



#### 테스트용 DB session 생성

다음 코드는 테스트에서 사용할 DB의 세션과 테스트 클라이언트를 fixture로 생성해놓은 코드이다. session의 경우 매번 테스트를 진행하기 위해서는 데이터베이스의 모든 테이블을 삭제한 후, 새로 생성한 테이블에서 테스트를 진행하도록 구현했다.&#x20;

```python
@pytest.fixture(scope="module")
def session():
    Base.metadata.drop_all(bind=engine)
    Base.metadata.create_all(bind=engine)

    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
```

#### 테스트 클라이언트 생성

TestClient는 FastAPI에서 제공하는 API를 테스트할 수 있는 클라이언트 객체이다. TestClient를 생성할 때, 위에서 생성한 테스트 session을 사용하기 위해서는 꼭 `app.dependency_overrides`를 이용하여 기존의 session을 생성하는 함수를 테스트용 함수로 오버라이딩 해야 한다.

```python
@pytest.fixture(scope="module")
def client(session):
    def override_get_db():
        try:
            yield session
        finally:
            session.close()

    # app에서 사용하는 DB를 오버라이드하는 부분
    # 이 부분을 해주지 않으면 위에서 생성한 테스트용 세션이 동작하지 않는다.
    app.dependency_overrides[get_db] = override_get_db

    yield TestClient(app)
```

### 설정 파일의 전체 코드

```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.db.base import Base
from app.db.base import get_db
from app.main import app

SQLALCHEMY_DATABASE_URL = "mysql+pymysql://kristoff:1234@localhost:3306/test_db"

engine = create_engine(SQLALCHEMY_DATABASE_URL)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)


@pytest.fixture(scope="module")
def session():
    Base.metadata.drop_all(bind=engine)
    Base.metadata.create_all(bind=engine)

    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()


@pytest.fixture(scope="module")
def client(session):
    def override_get_db():
        try:
            yield session
        finally:
            session.close()

    # app에서 사용하는 DB를 오버라이드하는 부분
    app.dependency_overrides[get_db] = override_get_db

    yield TestClient(app)
```

### 테스트 코드 작성

이후 테스트 코드를 작성할 때, 위에서 생성한 session, client 함수를 이용하여 테스트 코드를 작성한다.

위에서 client 및 session의 fixture scope 설정을 module로 설정했기 때문에 현재 테스트 파일을 실행 시키면 DB의 모든 테이블의 데이터를 삭제한다. 이후 새로 테이블을 생성하고 테스트를 진행한다.

IDE에서 client와 session 함수를 import 할 때, session 함수는 사용되지 않았다고 표시될 수 있다. 하지만 session은 사용되지 않는것이 아니라 test함수의 인자로 전달될 때 같이 전달된다. 위에서 client 함수는 파라미터로 session 함수를 받기 때문이다. 만약 session 함수를 import 하지 않으면 에러가 발생한다.

```python
from app.tests.test_config import client, session

user_id = "test_user_1"
user_pw = "1234567890"
user_name = "test_user_name_1"
update_user_name = "test_update_name"
update_user_pw = "0000000000"


def test_user_create(client):
    """
    유저 생성 테스트
    """
    response = client.post(
        "/api/v1/user",
        json={"user_id": user_id, "user_pw": user_pw, "user_name": user_name},
    )

    assert response.status_code == 201
    assert response.json() == {"user_id": user_id, "user_name": user_name}
```
