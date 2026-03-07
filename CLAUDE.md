# devhub

멀티 플랫폼 소셜 API 통합 클라이언트. Dev.to, Bluesky, Twitter/X, Reddit를 하나의 async 인터페이스로 제공.

## 구조

```
devhub/
├── __init__.py      # Public API (Hub, Post, Comment, DevTo, Bluesky, Twitter, Reddit)
├── types.py         # Post, Comment, UserProfile, PostResult, RateLimit dataclass
├── hub.py           # Hub — 멀티 플랫폼 오케스트레이터, from_env() 팩토리
├── base.py          # PlatformAdapter ABC (모든 어댑터의 베이스)
├── devto.py         # Dev.to (httpx 직접, 공식 SDK 없음)
├── bluesky.py       # Bluesky (atproto SDK)
├── twitter.py       # Twitter/X (twikit 읽기 + tweepy 쓰기 하이브리드)
└── reddit.py        # Reddit (asyncpraw)
```

## 핵심 패턴

### PlatformAdapter ABC
- 모든 어댑터가 동일한 메서드 구현: get_trending, search, get_post, get_comments, write_post, write_comment, upvote
- `is_configured` → 환경변수 존재 여부로 판단
- async context manager (`async with DevTo(...) as devto`)
- 반환값은 항상 통합 타입 (Post, Comment, PostResult)

### Hub
- `Hub.from_env()` → .env에서 설정된 플랫폼만 자동 활성화
- `hub.search()` → 활성 플랫폼 전체에 asyncio.gather로 병렬 검색
- `hub.publish()` → 여러 플랫폼에 동시 게시

### 데이터 모델
- dataclass 기반
- `post.raw` → 플랫폼 원본 dict 항상 보존
- `post.platform` → 어떤 플랫폼에서 온 데이터인지 식별

## 코드 스타일
- Python 3.10+, 전체 async
- 타입 힌트 필수 (소문자 generic)
- httpx로 HTTP 호출 (requests 사용하지 않음)
- 환경변수: python-dotenv 로드, 플랫폼별 prefix (DEVTO_, BLUESKY_, TWITTER_, REDDIT_)

## 빌드
```bash
pip install -e ".[all,dev]"
pytest
mypy devhub/
ruff check devhub/
```

### Twitter 하이브리드 구조
- **읽기**: twikit (무료, 로그인 기반) 우선 → tweepy fallback
- **쓰기**: tweepy (공식 API v2) 전용
- **검색**: tweepy + Bearer Token (OAuth2) — `TWITTER_BEARER_TOKEN` 환경변수
- twikit 로그인 실패 시 `self._twikit = None`으로 graceful fallback (에러 안 냄)
- 쿠키: `~/.devhub/twitter_cookies.json` (connect 시 로드/로그인, close 시 저장)
- `is_configured()`: twikit OR tweepy(OAuth1) OR bearer_token 어느 한쪽이라도 있으면 True
- `_has_tweepy_creds`: OAuth1 4개 키 OR bearer_token만으로도 True
- extras: `twitter` (둘 다), `twitter-read` (twikit만), `twitter-write` (tweepy만)

## 의존성
- 코어: httpx, python-dotenv
- Bluesky: atproto
- Twitter: twikit (읽기) + tweepy (쓰기)
- Reddit: asyncpraw
- Dev.to: 코어만으로 충분 (httpx)
- 개발: pytest, pytest-asyncio, mypy, ruff
