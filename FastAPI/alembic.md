# alembic 



## alembic이란?

`alembic`은 DB migration을 위해 사용되는 도구로,   

이를 사용하면 DB 스키마의 변경사항(테이블 추가, 칼럼 수정, 인덱스 생성 등)들을 코드로 정의할 수 있다.   

또한 스키마 변경사항을 버전으로 관리할 수 있기 때문에 특정 시점으로 쉽게 롤백하거나 이동할 수 있다.   

그 외에도 DB 스키마와 코드 상 모델 정의 간의 차이를 감지할 수 있고 필요 시 마이그레이션 스크립트도 생성할 수 있다. 

따라서 기능 개발 및 배포 과정에서 DB 스키마 불일치를 방지할 수 있다.  



## alembic 사용법

### 1. 패키지 설치하기

아래의 스크립트로 패키지를 설치할 수 있다.

```bash
>> pip install alembic
```

### 2. 초기화하기

아래의 스크립트로 dependency를 추가할 수 있다.

```bash
>> alembic init [폴더명] #일반적으로 폴더명은 alembic을 사용
```

이 명령어는 `alembic.ini` 설정 파일과 마이그레이션 스크립트가 저장될 디렉토리를 생성한다.  

생성된 디렉토리 및 파일은 다음과 같다.  

```
├── alembic
|    ├── versions
|    ├── README
|    ├── env.py
|    ├── script.py.mako
├── alembic.ini
```

이렇게 자동으로 생성된 파일 중에서 세팅해야 하는 파일들이 있다. 

### 3. 세팅하기

초기화 후에는 자신의 프로젝트에 맞게 `alembic.ini`파일과 `alembic/env.py` 파일을 세팅해주어야 한다.  

#### `alembic.ini`

`alembic.ini` 파일은 alembic에 대한 기본 설정을 담는 파일이다.   

이 파일에 작성한 내용들은 나중에 alembic이 실행될 때 생성자에게 전달되어 Config객체가 되며, 이를 `alembic/env.py` 파일에서 사용할 수 있다.  

초기 생성파일에서 주석을 제외하고 나머지만 살펴보자.  

```ini
[alembic]
script_location = alembic
prepend_sys_path = .
version_path_separator = os
sqlalchemy.url = driver://user:pass@localhost/dbname
```

- **`script_location`**: 마이그레이션 스크립트(`alembic/env.py`)가 저장된 경로를 지정, 이 부분은 필수적임

- `prepend_sys_path`: 시스템 경로에 추가할 디렉토리를 작성, 필수적이지 않으며 초기 파일과 같이 현재 디렉토리라면 삭제해도 무관

- `version_path_separator`: 버전 경로의 구분자를 세팅, os별 기본 구분자(윈도우 세미콜론, 유닉스계열 콜론)를 사용한다면 삭제해도 무관, 내부적으로 기본 구분자는 `os.pathsep` 을 통해 확인

- **`sqlalchemy.url`**: DB 연결 url을 세팅, 이 부분은 필수적이며 본인의 DB에 맞게 바꿔야 함

이 부분을 제외하고는 모두 로깅에 대한 설정으로 필수적이진 않지만, 디버깅 및 모니터링을 용이하게 하기 위해서는 세팅하는 것이 좋다.  

```ini
[loggers]
keys = root,sqlalchemy,alembic

# ~~중략~~

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic
```

`[loggers]`에서 key 에 `root`, `sqlalchemy`, `alembic` 세 가지를 두었는데,  그러면 이 세 개의 로거를 사용하겠다는 의미이다.   

그리고 `[logger_{key}]`에 정의한 설정이 각 로거의 상세설정이 된다.  

즉, `root` 라는 이름의 로거는 `[logger_root]`에서 상세설정을, `sqlalchemy`라는 이름의 로거는 `[loger_sqlalchemy]`에서 상세설정을, `alembic`라는 이름의 로거는 `[logger_alembic]`에서 상세설정을 한다.  

각 로거의 상세설정을 살펴보면 

- `level`: 로거가 처리할 로그 레벨을 지정하는 것으로, 명시된 레벨보다 높은 레벨의 로그만 남김, 로그 레벨은 `NOTSET < DEBUG < INFO < WARNING < ERROR < CRITICAL`순서로 상세한 내용은 [공식문서](https://docs.python.org/3/library/logging.html#logging-levels)에서 확인할 수 있음  

- `handlers`: `handler`설정에서의 key를 세팅하는 곳으로, key별 상세설정한 방식으로 로그를 남김  

- `qualname`: 실제 소스에서 해당 로거를 부를 이름을 세팅하는 곳으로, 이름이 없는 경우 루트로거이며, 로거를 계층구조로 세팅할 수도 있음(예시에서 `sqlalchemy.engine`과 같이 하위로거를 세팅할 수 있음)

```ini
[handlers]
keys = console

# ~~중략~~

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic
```

`[handlers]`에서 key에 `console` 한 가지를 두었는데, 핸들러를 하나만 사용하겠다는 의미이다.  

또한 `[handler_{key}]`에 정의한 설정이 각 핸들러의 상세설정이 된다.  

핸들러의 상세설정을 살펴보면  

- `class`: 로깅에서 제공하는 핸들러 클래스 중 사용할 핸들러를 지정하는 곳이며, 사용가능한 핸들러는 [공식문서](https://docs.python.org/3/library/logging.handlers.html)에서 확인할 수 있음
- `args`: 핸들러별 인자를 지정하는 곳이며 핸들러별로 모두 인자가 다르므로 위의 공식문서에서 각 핸들러별 인자값을 확인할 수 있음  
- `level`: 핸들러가 처리할 로그 레벨을 지정
- `formater`: `formatter`설정에서의 key를 세팅하는 곳으로 key별 상세설정한 형식으로 로그를 남김

```ini
[formatters]
keys = generic

# ~~중략~~

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

`[formatters]`에서 key에 `generic` 한 가지를 두었는데, 핸들러를 하나만 사용하겠다는 의미이다.  

또한 `[formatter_{key}]`에 정의한 설정이 각 핸들러의 상세설정이 된다.  

포메터의 상세설정을 살펴보면  

- `format`: 로그 메세지의 형식을 지정하는 곳으로, 사용할 수 있는 로그 레코드의 속성(`levelname`, `name`, `message` 등)은 [공식문서](https://docs.python.org/3/library/logging.html#logrecord-attributes)에서 확인할 수 있음  
- `datefmt`: 로그 레코드 속성 중 `asctime`의 형식을 지정하는 곳으로, 로그 메세지 형식 상 `asctime`이 사용되지 않았다면 지정할 필요가 없음  

간단히 아래와 같이 `alembic.ini` 파일을 세팅했다면

```ini
[alembic]
script_location = alembic
sqlalchemy.url = driver://user:pass@localhost/dbname

[loggers]
keys = sqlalchemy

[handlers]
keys = console

[formatters]
keys = generic

[logger_sqlalchemy]
level = WARN
handlers = console
qualname = sqlalchemy.engine

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(name)s | %(asctime)s - [%(levelname)s] %(message)s (%(filename)s:%(lineno)d)
datefmt = %Y-%m-%d %H:%M:%S
```

실제 소스에서 아래와 같이 부르고 사용할 수 있으며

```python
import logging
from logging.config import fileConfig
from alembic import context

# alembic.ini를 config로 사용
config = context.config
fileConfig(config.config_file_name)

# 로거 가져오기
engine_logger = logging.getLogger('sqlalchemy.engine')

# 로그 메시지 생성
engine_logger.info("This is an info message.")
engine_logger.warning("This is a warning message.")
engine_logger.error("This is an error message.")
```

이렇게 생성된 로그는 아래와 같이 출력됩니다.  

```bash
sqlalchemy.engine | 2024-05-23 12:34:57 - [WARNING] This is a warning message. (example.py:16)
sqlalchemy.engine | 2024-05-23 12:34:58 - [ERROR] This is an error message. (example.py:17)
```

~~(물론, `alembic.ini`을 사용하지 않고 자신이 하나하나 다 세팅을 해줘도 된다.~~

~~도전정신과 시간적 여유와 할 수 있다는 자신감이 있다면...)~~

#### `alembic/env.py`

`alembic/env.py`는 `alembic`이 마이그레이션할 DB연결과 SQLAlchemy 메타데이터 객체를 설정하는 파일이다.  

초기 생성파일에서 주석을 제외하고 나머지만 살펴보자.  

```python
from logging.config import fileConfig
from sqlalchemy import engine_from_config
from sqlalchemy import pool
from alembic import context

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)
```

필요한 모듈을 import하고 앞서 세팅한 `alembic.ini`파일을 config로 사용하도록 세팅하는 부분이다.  

```python
target_metadata = None
```

이 부분은 필수적으로 세팅해야하는 부분인데, SQLAlchemy 모델을 정의한 파일의 Base를 import하고 그 값으로 변경해주어야 한다.  

예시로 보자면 아래와 같이 세팅해주어야한다.  

```python
from app.mymodel import Base

target_metadata = Base.metadata
```

`alembic/env.py`의 나머지 부분은 Offline/Online 모드에 대한 기본 세팅이다.  

Offline 모드는 DB연결 없이 마이그레이션 파일을 생성하는 모드이고, Online 모드는 DB에 직접 연결하여 차이점을 비교하고 마이그레이션을 수행하는 모드이다.  

```python
def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection, target_metadata=target_metadata
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

이 부분은 요구사항이 있을 경우 추가 구현을 하여도 되지만 그렇지 않다면 별도의 세팅 없이 사용할 수 있다.  

### 4. 사용하기

#### a. 마이그레이션 스크립트 생성하기

마이그레이션이 필요할 때(SQLAlchemy 모델을 변경하는 등), 자동 생성기능을 사용하여 다음 명령어로 변경사항을 반영하는 version 파일을 생성할 수 있다.  

```bash
>> alembic revision --autogenerate -m "Initial migration"
```

(`-m` 이후의 부분은 메세지를 작성하는 부분이다.)  

물론 자동생성 기능을 사용하지 않고

```bash
>> alembic revision -m "Initial migration"
```

이와 같이 명령을 실행하여도 version파일을 생성하지만, 다음과 같이 내용이 비어있게 된다.  

```python
"""add new table

Revision ID: ae1027a6acf
Revises: 
Create Date: 2023-05-23 12:00:00.000000

"""
from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision = 'ae1027a6acf'
down_revision = None
branch_labels = None
depends_on = None


def upgrade():
    # ### commands auto generated by Alembic - please adjust! ###
    pass
    # ### end Alembic commands ###


def downgrade():
    # ### commands auto generated by Alembic - please adjust! ###
    pass
    # ### end Alembic commands ###

```

따라서 `upgrade()`와 `downgrade()` 에 대한 내용을 직접 작성해주어야한다.  

이렇게 `revision` 명령어를 통해 생성된 version파일은 초기화 시 생성되었던 디렉토리인 `alembic/versions`안에 생성된다.  

#### b. 마이그레이션 적용하기

다음 명령어를 사용하여 생성된 마이그레이션 스크립트를 실행하여 DB에 적용할 수 있다.  

```bash
>> alembic upgrade [option]
```

 마이그레이션 업그레이드 옵션은 다음과 같은 것들이 있다.  

- `<revision_id>`:  각 마이그레이션 파일에 있는 고유한 ID를 그대로 작성(예시: `ae1027a6acf`)
- `head`: 헤드 버전(최신 버전)으로 업그레이드(예시: `head`)
- `{+n}`: 현재 버전을 기준으로 n단계 앞으로 업그레이드(예시: `+1`)

#### c. 마이그레이션 되돌리기

필요한 경우 특정 마이그레이션 버전으로 롤백할 수 있으며 명령어는 다음과 같다.  

```bash
>> alembic downgrade [option]
```

마이그레이션 다운그레이드 옵션은 다음과 같은 것들이 있다.  

- `<revision_id>`:  각 마이그레이션 파일에 있는 고유한 ID를 그대로 작성(예시: `ae1027a6acf`)
- `head{-n}`: 헤드 버전(최신 버전)을 기준으로 n단계 뒤로 롤백(예시: ` head-2`)
- `{-n}`: 현재 버전을 기준으로 n단계 뒤로 롤백(예시: `-3`)
- `base`: 모든 마이그레이션을 롤백

