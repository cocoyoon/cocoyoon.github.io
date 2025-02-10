---
title: 플랫폼 개발 회고
date: '2024-02-24'
tags: ['web-service']
draft: false
summary: 플랫폼 개발 사이클에 대한 회고
---

# Frontend

### SEO

- JSON-LD 구조화 데이터 적극 활용
- Metadata 태그
  - `layout.ts`
    - `<html lang="..">`
      - 웹페이지의 주 언어를 명시하는 html 속성
      - 해당 언어권 사용자에게 검색 결과를 더 잘 노출
      - 브라우저 번역 지원: 다른 언어이면 번역 제안
  - [open-graph](https://ogp.me)
  - twitter-tag
- sitemap
- manifest

# Backend

## Cloudflare

```
- Current
Client ← (HTTPS) → Cloudflare ← (HTTP) → Your Server

- Full SSL
Client ← (HTTPS) → Cloudflare ← (HTTPS) → Your Server
```

## Telegram Bot

Telegram 은 Telegram 서버와 통신할 수 있는 bot 을 지원한다. 워낙 큰 서비스이기 때문에 다양한 언어로 구현되어 있고 _callback_, _message_ 등의 다양한 액션을 가능하게 한다. 어떤 버튼을 눌렀을 때 어떠한 액션을 하게 하는 등의 자유도가 있다. 디코디드에서는 이 bot 을 이용해서 관리자 대시보드를 대체하는 bot 서비스를 구축하였다.

### Application

Telegram bot 을 돌리기 위한 사전 설정값들이다.

- `token` : 텔레그램 봇 토큰
- `request` : 요청에 대한 설정값

```python
self.application = (
	Application.builder()
		.token(self.bot_token)
		.request(request)
		.concurrent_updates(True)
		.get_updates_request(request)
		.build()
)
```

### `poll` vs `webhook`

기본적으로 다음 두가지 방식을 지원한다. `poll` 은 지속적으로 특정 interval 만큼 이벤트가 있는지 텔레그램 서버로부터 확인한다. 가장 간단하게 구현할 수 있는 방법이지만 _interval_ 동안 계속 polling 을 해줘야하므로 어느정도의 리소스를 사용하게 된다. 반면 `webhook` 방식은 텔레그램 서버로부터 트리거를 받으면 어떤 액션을 취하는 방식이다. 당연히 이벤트가 있을때마다 트리거가 되기때문에 훨씬 효율적인 방법이지만 사전에 웹훅을 등록하는등의 사전작업이 필요하다. 현재 `v0.*.*` 버전에서는 빠르게 구현하기 위해 polling 방식을 채택하였지만 릴리즈 이후에는 `webhook` 방식으로 변경할 예정이다.

**Poll**

```python
poll_interval = 3.0 # 3 seconds
def start:
	await self.application
	asyncio.create_task(poll(poll_interval=poll_interval))
```

- `fastapi` lifespan 을 이용해서 `TelegramService` 의 주기를 관리하였다

### Handler

Telegram 은 다음 세 가지 핸들러를 지원한다.

- `CommandHandler`: `/*` 이런식으로 채팅창에 명령어를 입력했을 때 처리를 정의
  - `/help` : 봇의 사용 방법 가이드를 정의할 수 있다
  - `/metrics` : 현재 디코디드 플랫폼의 메트릭을 정리
- `MessageHandler`: 채팅방에 메세지가 입력됐을 때 처리를 정의
- `CallbackQueryHandler`: 특정 버튼을 눌렀을 때 처리를 정의
  - `InlineKeyboardButton` 에 `title` 과 `callback_data` 를 넣어주어 버튼의 title 과 어떤 `callback` 을 실행할지를 결정하게 된다
  - 마지막으로 `InlineKeyboardMarkup` 에 버튼들을 담아주어 `reply_markup` 을 생성해준다

### Callback Handle

- 디코디드는 `TGServiceKind` 에 따라 각기 다른 callback 을 처리하게 해주었다. 서로 구분하기 위해 다음과 같은 프로토콜을 따른다
- `{TGServiceKind}-{request_id}-{TGCallbackActionKind}-[additional_data]`
  - `TGServiceKind`: 현재 처리할 서비스를 의미. `TGServiceKind::Request*` 의 경우 다음엔 무조건 `request_id` 가 오게된다
  - `request_id` : 현재 요청 `id` 를 의미
  - `callback_action` : 현재 어떤 액션을 수행할 버튼인지 의미
  - `additional_data` : 추가적으로 전달할 데이터를 의미

```python
class TGServiceKind(Enum):
	REQUEST_IMAGE = "ri"
	REQUEST_PROVIDE = "rp"

class TGCallbackActionKind(Enum):
	APPROVE = "approve" # 승인과 관련된 콜백
	REJECT = "reject" # 거절과 관련된 콜백
	COMPLETE = "complete"
```

- Telegram Server 의 `callback` 키 길이 제한이 있어 각 키별로 상태를 관리할 수 있게 `SessionManager` 를 구현하였다. 구현은 간단하다. `{TGServiceKind}_{identifier}` 를 키로 하는 _session_ dictionary 를 두어 timestamp 를 찍어 관리한다.

## FastAPI

### Package Manager

#### uv

- 테스트 코드 돌릴 시, 기본 패키지도 dev 의존성에 추가해줘야한다

```bash
uv pip install -e ".[dev]"
```

#### Poetry

- 파이썬 패키지 관리 라이브러리(e.g Rust 의 Cargo 와 비슷)
- 의존성 관리 예시

```text
- >=2.10.1: 2.10.1 이상의 모든 버전 (예: 2.10.1, 2.11.0, 3.0.0 등 모두 가능)
- ^2.10.1: 2.10.1 이상 3.0.0 미만 (메이저 버전 내에서만 업데이트)
- ~2.10.1: 2.10.1 이상 2.11.0 미만 (마이너 버전 내에서만 업데이트)
- ==2.10.1: 정확히 2.10.1 버전만 사용
- >2.10.1: 2.10.1 초과 버전
- <2.10.1: 2.10.1 미만 버전
```

- 테스트 코드

```shell
APP_ENV=dev poetry run pytest tests/test_email_service.py -v
```

### CORS(Cross Origin Resource Sharing)

- 인터넷은 어떠한 곳에서든 자원을 요청할 수 있다. 특히 웹앱을 만드는 경우 자기 자신이 신뢰하는 곳으로부터 자원을 요청하고 이를 기반으로 서비스 하게 된다
- 헤더에 포함된 `ACCESS-CONTROL-ALLOW-ORIGIN` 을 통해 서로 같은 origin 혹은 허용된 origin 인지 체크한 후 리소스를 수용하게 된단

### Scheduler

- [APScheduler](https://pypi.org/project/APScheduler/) 활용
- Trigger 종류
  - CronTrigger: Unix cron 스타일의 스케줄링
  - IntervalTrigger: 정해진 시간 간격으로 반복 실행
  - DateTrigger: 지정된 날짜/시간에 한 번만 실행
- `_lock`: 어떤 operation 도중에 인스턴스를 참조하는것을 방지하기 위해 `_lock` 을 통해 동시성 제어
- Method
  - `start` : `AsyncIoSchedueler` 인스턴스의 `start` 를 통해 Schduler 시작
  - `shutdown`: Scheduler 종료
  - `add_job(job_id, func, trigger, description, replace_existing)`
    - `job_id`: Job 의 id
    - `func` : 실행할 함수
    - `trigger`: 어떤 방식으로 트리거할 건지(_Trigger 종류_ 참고)

### Lifespan

- Server 의 런타임 주기를 `@asynccontextmanager` _decorator_ 로 관리

```python
	@asynccontextmanager
	async def lifespan(app: FastAPI):
		# Start app
		# - Connect db
		# - Connect redis
		yield
		# End app
		# - Disconnect db
		# - Disconnect redis
```

### 미들웨어(Middleware)

- 서버 클라이언트간 요청/응답 사이에서 동장하는 소프트웨어. 모든 요청이 실제 라우터 핸들러에 도달하기 전에 혹은 응답이 클라이언트에 도달하기전에 특정 작업을 수행한다
- 사용 처:
  - 로깅
  - 인증/인가 체크
  - CORS 처리
  - 요청 통계 수집
  - 에러 핸들링
  - 응답 헤더 수정
  - Rate limit 처리
  - CORS 처리

```python
app.add_middleware(
	CORSMiddleware,
	allow_origins=["*"],
	allow_credentials=True,
	allow_methods=["*"],
	allow_headers=["*"]
)
```

### Trending System

- 플랫폼에서는 데이터가 쌓이기 때문에 현재 시간 기준 어떤 아이템이 트렌딩중인지 알려줄 수 있어야 한다.
- 구현에서 중요하게 생각한 포인트:
  - 실시간 점수 기록
    - 특정 시간(window) 사이즈 만큼 데이터 반영
    - window 이전 데이터는 삭제하여 redis 메모리 부담 감소
  - 배치 처리
    - 스케줄러 활용하여 특정 시간마다 score update
    - 시간 가중치 적용하여 최근 데이터에 더 높은 점수 부여
  - redis + mongodb
  - 인덱스를 활용한 효율적인 조회

### Search

- 검색은 클라이언트로 부터 들어온 _query_ 혹은 _keyword_ 를 기반으로 유사한 혹은 일치하는 데이터를 반환하는 서비스다
- DB 에서 가져오기 때문에 query 문을 어떻게 구성하는가에 따라서 성능에 영향을 미치며 query 뿐만 아니라 indexing, 캐싱 등에 따라 검색 성능이 달라진다
- 디코디드에서는 다음과 같은 작업을 진행하였다:
  - 인덱스 생성
    - 각 document 필드를 빠르게 조회하기 위한 index 생성
  - 캐싱 적용 w/ redis
  - 쿼리 최적화
  - 배치 처리
  - 성능 모니터링

### Transactional

- 모든 API 오퍼레이션은 atomic 한 state change 를 보장해야한다. single 오퍼레이션은 에러가 나면 raise하여 상태 변화를 막을 수 있지만 여러개의 operation 이 있을때 atomic 한 변화를 보장하기 위해 _transactional_ 을 사용하게 된다
- FastAPI 자체적으로 transactional 을 지원하지 않고 Replica set 을 두고 진행해야 한다

```yaml
services:
	mongodb-primary:
		image: mongo:latest
		command: ["--replSet", "rs0"(id), "bind_ip_all", "--port", "27017"]
	mongodb-secondary:
		image: mongo:latest
		command: ["--replSet", "rs0", "--bind_ip_all", "--port", "27018"(다른 포트)]
		ports:
		- "27017:27017"
		volumes:
		- mongodb-primary-data:/data/db
	mongodb-setup:
		image: mongo:latest
		depends_on:
			mongodb-primary:
				condition: service_healthy
		ports:
		- "27018:27018"
		volumes:
		- mongodb-secondary-data:/data/db
		command: >
			mongosh --host mongodb-primary:27017 --eval '
				rs.initiate({
					_id: "rs0",
					members: [
						{_id: 0, host: "mongodb-primary", priority: 2},
						{_id: 1, host: "mongodb-secondary", priority: 1}
					]
			})'
```

- Replica set auth를 위한 key 값 세팅

```bash
openssl rand -base64 756 > mongodb/keyfile
chmod 400 mongodb/keyfile

# dev
mkdir -p mongodb/dev/keyfile
openssl rand -base64 756 > mongodb/dev/keyfile/keyfile
chmod 400 mongodb/dev/keyfile/keyfile

# prod keyfile 생성
mkdir -p mongodb/prod/keyfile
openssl rand -base64 756 > mongodb/prod/keyfile/keyfile
chmod 400 mongodb/prod/keyfile/keyfile
```

- Decorator* 를 통해 *transactional\* 을 구현했다
  - `transactional` 데코레이터 안으로 들어올 `func` 을 paramater 로 받기
  - `wraps()`: `func` 의 속성을 그대로 가져오기 위함(e.g `doc-string` , `func-name` , etc..)

```python
# Parameter type
P = ParamSpec("P")
# Return type
T = TypeVar("T")

def transactional(func: Callable[P, T]) -> Callable[P, T]:
	@wraps(func)
	async def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
		async with await backend.client.start_session() as session:
			try:
				kwargs["session"] = session
				result = await func(*args, **kwrags)
				return result
			except
				raise HttpException
	return wrapper
```

### Redis

- in-memory(feat. RAM) 형식의 key-value 데이터 베이스
- RAM 에서 IO 가 이루어져 접근이 빠르다
- Instance 생성
  - Singletone: Redis 연결 자체는 cost 한 작업으로 런타임내 인스턴스는 하나만 생성

```python
class RedisManager:
	pass

# Global 하게
redis_manager = RedisManager()
```

- Key 설정
  - 값에 매핑되는 키값을 저장(e.g `day:image:*`, `hour:item:*`)
- Sorted Set
  - 특정 키에 매핑되는 Set 을 생성
  - `zincrby` 와 `zrevrange` 혹은 `zrange` 를 같이 사용

```python
redis.zincrby(f"{key}", 1, ${value}) # key -> { "value": 1 } if new else { "value": 2 }

redis.zrevrange(f"{key}, 0, 5, withScores=True)
# 키에 해당 하는 set 을 반환하고
# 5 개(start=0, end=5)의 항목을 반환하고
# 점수와 함께(withScores=True)
# rev: 역순으로 반환
```

- Pipeline
  - 여러 개의 명렁어를 redis 서버에 요청할때 사용
  - `redis.pipeline()`
- Trending 알고리즘
  - 각 action_type 마다 가중치값들을 설정하고 점수를 매기는 것
  - Metrics 를 측정하고 hourly 혹은 daily 로 상위 카테고리 분석
- 노출률
  - 외부 링크타고 들어오는 경우
  - 외부로 공유하는 경우
  - 좋아요(혹은 북마크)
  - 링크 제공할 경우
- Error 핸들링
  - Redis 작업은 business logic 과는 관련이 없으므로 에러가 난다고 해도 _hard failure(e.g HTTP Exception)_ 보다는 _soft failure_ 방식으로 처리
  - Multi-threading 으로 retry 로직을 처리하는 thread 로 처리
  - 내부적으로 queue 로 관리
- pubsub
  - 이벤트 시스템 구축에 사용
  - Redis 에서 publish 하고 server 쪽에서 subscribe.
  - 클라이언트(e.g 웹앱)은 서버와 웹소켓으로 연결하여 이벤트 발생시 서버가 보낸 데이터를 수신
  - 각 (클라이언트)웹소켓 연결시 마다 _pubsub_ 인스턴스를 생성하고 _active subscribers_ set 관리. 연결 종료 후에는 clean up(e.g `pubsub.close()`) 진행
- Rate-limit
  - 클라이언트 `ip` 를 key 로 하여 rate 트래킹
  - expired 는 보통 1 min 으로 처리하여 메모리 부담 줄임
  - now(`datetime.now()`)가 window(rate-limit 을 걸고자하는 크기. 예를 들면 unixtime 기준으로 60 차이는 1 분을 의미) 보다 작은 경우는 redis 에서 제거
  - 제거 후 score 를 셋을 때 limit 개수 보다 큰 경우 `429` 에러 반환
  - middleware 를 추가하여 `Header` 에 rate limit 관련 요소들 추가
- Docker redis 접근 명령어

```bash
	docker exec -it redis-dev redis-cli -a ${REDIS_PASSWORD}
```

### Logger

- 런타임 환경에 따른 LOG_LEVEL 을 분리하여 로깅 관리
  - DEBUG (가장 상세. 개발환경에서만)
  - INFO
  - WARNING
  - ERROR
  - CRITICAL
- 서버 root 디렉토리에 `./logs` 를 생성하여 관리
- `logging_config` 딕셔너리를 생성하여 config 를 관리
  - `formatters`: Log 형식 설정
  - `handlers`
    - `console` : 콘솔 출력 핸들러
      - class: "logging.StreamHandler"(스트림(콘솔)에 출력하는 핸들러)
      - formatter: "default"(위에서 정의한 포맷 사용)
      - stream: `"ext://sys.stdout"`(표준 출력(stdout)으로 전송)
    - `file` : ERROR 레벨 이전의 로그 핸들러
    - `error` : ERROR 레벨 이상의 로그 핸들러
  - `loggers` : 각 모듈별(e.g db, redis, api, etc..) 로거 관리
    - handler: 위에서 정의한 handler 설정 import
    - level: 어떤 레벨부터 logging 할지 설정
    - propagate: root 로거로 전파 여부(True 일시 중복 저장가능성 존재)
  - `filter` : 특정 level 로그를 필터링할때 사용
    - 예를 들면, `api.log` 는 `ERROR` 레벨을 제외한 로그를 저장

### Authentication

- Secret Key 생성 후 env 에 세팅

```
echo "JWT_SECRET_KEY=$(openssl rand -hex 32)" >> .env.prod
```

- 보통 서버에서의 인가는 JWT 를 통해 진행한다. `Header.Payload.Signature` 로 이루어져 있으며 signature 에 들어가는 crypto-scheme 은 서버에서 설정하기 나름이다
- 대부분의 경우 HMAC 은 지원하고 있어 현 서버도 이를 채택하였다.
- 인가가 필요한 API(a.k.a protected router) 와 필요없는 API(a.k.a public router) 를 분리하였다. _protected_ 의 경우 로그인 이후나 API 키 발급을 통해 API 호출시 서버쪽에서 발급한 jwt 토큰을 헤더에 포함한시킨다
- 해당 토큰을 서버에서 verify 하고 검증이 완료되면 리소스 사용허가를 해주게 된다

### Test

**파일 구조**

- `conftest.py`
  - Test 할때 공통으로 사용하는 configuration(e.g db, env, etc)
  - `pytest.fixture` 데코레이터 사용. 자동으로 모든 테스트 메소드에 적용하고 싶으면 `autouse=True` 추가
- `test_*.py`
  **CLI**

```bash
// 모든 테스트
pytest tests

// 특정 파일
pytest tests/test_*.py

// 특정 파일의 함수
pytest tests/teset*.py::test_works
```

## MongoDB

### Query

- `$match` : 조건에 맞는 문서 query
  - 보통 `$or` 이나 `$and` 같이 쓰인다

```python
{
 "$match": {
	"$or": condition # condition 에 하나라도 맞는부분 필터링
  }
}
```

- `$addFields` : 각 문서에 새로운 필드 추가하는 쿼리로 보통 match 해서 나온 점수 필드를 추가

```python
{
	"$addFields": {
	    # matchScore 라는 각 문서에 추가
		"matchScore": {
			"$sum": [
				{
					"cond": [{"$or"}, 10, 0]
				},
				..
			]
		}
	}
}
```

### 정규 표현식 옵션

- `i` : case-insensitive(대소문 구별 안함)
- `m`: multiline(여러줄 매칭)
- `x`: extended (정규식에서 공백 무시)
- `s`: dot-all (점(.)이 개행 문자도 매칭)
  사용 예

```python
{"$regexMatch": {"input": "name.ko", "regex": {token}, "options": i or m or x or s}}
```

# DevOps

## Docker

# Home Server

AWS 비용을 생각하여 맥미니를 홈서버로 구축하는 것으로 결정하였다.

## nginx

- 외부에서 들어오는 요청을 받아 적절한 곳에 요청을 뿌려주는 라우터같은 서버
- `conf.d` 파일을 수정하여 어떤 포트에 listen 하고 있을거며 어떤 서버에 뿌려줄지를 결정
- 수정 후 syntax 테스트 및 재설정

```bash
sudo nginx -t
sudo nginx -s reload
```

- nginx 는 nginx 자체 오류가 나거나 서버에서 400-500 에러가 나면 `Access-Control-Allow-Origin` 과 `Access-Control-Allow-Credentials` 부분을 제거한 _Response_ 를 응답하게 된다.
- 이슈
  - 디코디드 플랫폼 개발시 이미지 요청을 `POST` 요청시 위 필드가 포함되지 않는 Response 를 반환하여 _CORS_ 에 걸리는 이슈 발생. 이슈 해결을 다음과 같이 진행:
    - 백엔드(fastAPI) 쪽에 `allow_origin` 이 `dev` 일 경우 `*` 로 설정돼어있음을 확인하여도 동일하게 이슈 발생
    - 특이하게 이미지 요청 제외한 다른 POST 요청은 `200` 을 반환하는 것을 확인하고 이미지와 관련된 이슈임을 파악
    - 문제의 원인을 어느 정도 파악하고 검색을 이쪽 위주로 진행한 결과 nginx 쪽에 body 용량을 설정하는 부분이 있음을 파악. `client_max_body_size` 와 `client_body_buffer_size`
    - 처음에는 `client_max_body_size` 만을 설정했었는데 해당 필드는 말그대로 최댓값 설정이여서 `client_body_buffer_size` 를 `100M` 설정함으로써 이슈 해결(뭔가 요청한 값보다는 작았을 것으로 예상)
  - 웹소켓 설정
    - 설정없이 ws 연결을 진행하니 단순 _http_ 요청으로 들어오는 것을 확인
    - 웹소켓 프로토콜을 위한 필드값들을 헤더에 추가하여 해결
- 로드 밸런싱
  - nginx 자체에서 로드밸런싱을 가능하게 할 수 있다. 두 개의 replica 서버를 띄우고 nginx 에 reverse_proxy 설정을 해놓으면 알아서 진행하는거 같다
- https 설정
  - 보통 클라우드 플레어를 사용하게 되면 `브라우저 <> 클라우드 플레어` 간 ssl 설정이 기본으로 들어가게 되지만 `클라우드 플레어 <> 서버` 간 보안은 없는 flexible 한 보안으로 들어가게 된다
  - 클라우드 플레어에서 인증서를 발급하여 nginx 로컬 파일에 저장하여 완전 보안모드로 진행할 수도 있다.

## Storage

- SSD(512GB)

```text
연간 데이터 증가

- MongoDB 데이터: ~11GB
- 인덱스: ~2GB
- 로그: ~1GB
- 연간 총 증가: ~14GB

Replica Set 구성 시 실제 사용량

- Primary + Secondary = 연간 ~28GB
- 시스템 예약 공간 20% = ~102GB
- 실제 사용 가능 용량: ~410GB
```

따라서:

- 4-5년은 충분히 안정적 운영 가능
- 3년 차부터 용량 모니터링 강화
- 4년 차에 증설/마이그레이션 계획 수립
  - 외부 스토리지(추가 SSD)
  - 클라우드 마이그레이션
  - 혹은 새로운 하드웨어(?)

# Reference

- [Replica set 구축](https://velog.io/@youngeui_hong/Docker%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC-MongoDB-Replica-Set-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0)
- [Cloudflare https](https://usage.tistory.com/199)
- [vnc 포트 변경](https://picory.com/entry/MAC-%EC%9B%90%EA%B2%A9-%EC%A0%91%EC%86%8D-VNC-%ED%8F%AC%ED%8A%B8-%EB%B3%80%EA%B2%BD)
- CORS
  - https://coding-groot.tistory.com/91
  - https://kk-programming.tistory.com/63
  - https://doqtqu.tistory.com/311
- Unsafe browser : https://stackoverflow.com/questions/71953553/disable-cors-in-brave-browser
- Big file size: https://alucard001.medium.com/solved-big-file-size-cannot-upload-file-through-nginx-error-500-69039e78042e
- Telegram Bot Rust: https://crates.io/crates/teloxide
- Web Server Rust: https://crates.io/crates/axum
- [uv-docs](https://docs.astral.sh/uv/)
- [cqrs-python](https://github.com/marcosvs98/cqrs-architecture-with-python)
-
