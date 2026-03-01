# devhub

Unified async Python client for developer communities.

Dev.to, Bluesky, Twitter/X, Reddit — one interface, one data model.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)

## The Problem

Developer communities are fragmented across platforms. Each has its own SDK, auth flow, data format, and rate limiting. Writing a bot, dashboard, or cross-poster means juggling:

```python
import tweepy                    # Twitter — OAuth 1.0a
import asyncpraw                 # Reddit — OAuth2
from atproto import Client       # Bluesky — AT Protocol
import httpx                     # Dev.to — REST API (no official SDK)
```

Four SDKs, four auth patterns, four data formats. And if you want to search across all of them? You're writing glue code.

## The Solution

```python
from devhub import Hub

async with Hub.from_env() as hub:
    # Search across all configured platforms at once
    posts = await hub.search("MCP server", limit=5)
    # → [Post(platform="reddit", ...), Post(platform="devto", ...), ...]

    # Cross-post to multiple platforms
    results = await hub.publish(
        Post(title="Building MCP Servers", body="...", tags=["python", "mcp"]),
        platforms=["devto", "bluesky"],
    )
```

Or use individual platforms directly:

```python
from devhub import DevTo

async with DevTo(api_key="...") as devto:
    trending = await devto.get_trending(limit=10)
    await devto.write_post(title="...", body="...", tags=["python"])
    await devto.write_comment(post_id="123", body="Nice approach!")
```

## Features

- **Unified data model** — `Post`, `Comment`, `UserProfile` work the same across all platforms
- **Fully async** — `asyncio` native, no sync wrappers
- **Platform-aware** — respects each platform's conventions (subreddits for Reddit, tags for Dev.to, character limits for Twitter)
- **Graceful degradation** — only loads platforms with configured API keys, ignores the rest
- **Raw data preserved** — `post.raw` always contains the original platform response

## Supported Platforms

| Platform | Read | Write | Search | Trending | Auth |
|----------|:----:|:-----:|:------:|:--------:|------|
| **Dev.to** | O | O | O | O | API Key |
| **Bluesky** | O | O | O | O | App Password |
| **Twitter/X** | O | O | O | O | OAuth 1.0a |
| **Reddit** | O | O | O | O | OAuth2 |

## Install

```bash
# All platforms
pip install "devhub[all]"

# Specific platforms only
pip install "devhub[devto]"
pip install "devhub[bluesky]"
pip install "devhub[twitter]"
pip install "devhub[reddit]"
```

## Configuration

Set API keys via environment variables or `.env` file:

```env
# Dev.to — https://dev.to/settings/extensions
DEVTO_API_KEY=

# Bluesky — https://bsky.app/settings → App Passwords
BLUESKY_HANDLE=your.handle.bsky.social
BLUESKY_APP_PASSWORD=

# Twitter/X — https://developer.x.com/en/portal/dashboard
TWITTER_API_KEY=
TWITTER_API_SECRET=
TWITTER_ACCESS_TOKEN=
TWITTER_ACCESS_SECRET=

# Reddit — https://www.reddit.com/prefs/apps
REDDIT_CLIENT_ID=
REDDIT_CLIENT_SECRET=
REDDIT_USERNAME=
REDDIT_PASSWORD=
```

Only platforms with valid keys are activated. No keys = no errors, just skipped.

## API Overview

### Hub (Multi-Platform)

```python
from devhub import Hub, Post

async with Hub.from_env() as hub:
    # Discovery
    hub.platforms                             # ["devto", "reddit", "bluesky"]

    # Read
    posts = await hub.search("query")        # Search all platforms
    posts = await hub.search("query", platforms=["reddit"])
    trending = await hub.get_trending()      # Trending from all platforms

    # Write
    result = await hub.publish(post, platforms=["devto", "bluesky"])

    # Rate limits
    limits = await hub.rate_limits()         # All platforms at once
```

### Individual Platform

```python
from devhub import Reddit

async with Reddit.from_env() as reddit:
    # Read
    posts = await reddit.get_trending(limit=20)
    posts = await reddit.search("MCP server", limit=10)
    post = await reddit.get_post("abc123")
    comments = await reddit.get_comments("abc123")
    user = await reddit.get_user("username")

    # Write
    result = await reddit.write_post(
        title="...", body="...", subreddit="r/Python",
    )
    result = await reddit.write_comment(post_id="abc123", body="...")
    result = await reddit.upvote("abc123")

    # Meta
    reddit.is_configured                     # True
    limit = await reddit.check_rate_limit()  # RateLimit(remaining=98, reset_at=...)
```

### Data Models

```python
from devhub import Post, Comment, PostResult

# Post — unified across all platforms
post.id                 # Platform internal ID
post.platform           # "devto" | "bluesky" | "twitter" | "reddit"
post.title              # Empty string for Twitter/Bluesky
post.body               # Content text
post.author             # Username
post.url                # Permalink
post.tags               # ["python", "mcp"]
post.score              # Upvotes / likes
post.comment_count      # Number of comments
post.created_at         # datetime
post.raw                # Original platform response (dict)

# Comment
comment.id
comment.post_id
comment.body
comment.author
comment.parent_id       # For nested replies
comment.score

# PostResult — write operation result
result.success          # True/False
result.url              # Permalink to created content
result.post_id          # Platform ID of created content
result.error            # Error message if failed
```

## Architecture

```
devhub/
├── __init__.py          # Public API: Hub, Post, Comment, DevTo, Bluesky, ...
├── types.py             # Post, Comment, UserProfile, PostResult, RateLimit
├── hub.py               # Hub — multi-platform orchestrator
├── base.py              # PlatformAdapter ABC
├── devto.py             # Dev.to adapter (httpx)
├── bluesky.py           # Bluesky adapter (atproto)
├── twitter.py           # Twitter/X adapter (tweepy)
└── reddit.py            # Reddit adapter (asyncpraw)
```

### Adding a New Platform

Implement `PlatformAdapter`:

```python
from devhub.base import PlatformAdapter

class Mastodon(PlatformAdapter):
    name = "mastodon"

    def is_configured(self) -> bool:
        return bool(os.getenv("MASTODON_ACCESS_TOKEN"))

    async def get_trending(self, limit=20) -> list[Post]: ...
    async def search(self, query, limit=10) -> list[Post]: ...
    async def get_post(self, post_id) -> Post: ...
    async def get_comments(self, post_id) -> list[Comment]: ...
    async def write_post(self, **kwargs) -> PostResult: ...
    async def write_comment(self, post_id, body) -> PostResult: ...
    async def upvote(self, post_id) -> PostResult: ...
```

Register in `hub.py` and it's available everywhere.

## Development

```bash
git clone https://github.com/SonAIengine/devhub.git
cd devhub
pip install -e ".[all,dev]"

pytest                    # Tests
mypy devhub/              # Type check
ruff check devhub/        # Lint
```

## License

[MIT](LICENSE)
