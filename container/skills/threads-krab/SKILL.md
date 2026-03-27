---
name: threads-krab
description: |
  threads_krab API를 통해 Threads(Meta) 포스트를 작성하고 발행한다.
  단일 포스트 또는 연결된 스레드(체인) 발행을 지원한다.
  Victor(재표형)의 Threads 계정에 글을 올릴 때 사용하라.
triggers:
  - "threads에 올려줘"
  - "쓰레드 글 올려줘"
  - "threads 발행"
  - "포스트 올려줘"
  - "/threads-krab"
---

# threads-krab — Threads 발행 스킬

Victor의 Threads 계정에 글을 발행하는 API 래퍼.

## 환경

- **API Base URL**: `https://axzymzqxl1.execute-api.ap-northeast-2.amazonaws.com`
- **HMAC Secret**: 환경변수 `KRAB_HMAC_SECRET` 또는 `/workspace/group/store/kv.json`의 `krab_hmac_secret` 키에서 로드
- **Approver ID**: `786310533439160381` (Victor Jaepyo Jo)

HMAC Secret이 없으면 Victor에게 요청하라.

## 인증 (HMAC-SHA256 서명)

모든 요청에 두 헤더가 필요하다:

- `x-krab-timestamp`: 현재 Unix epoch (초)
- `x-krab-signature`: `sha256={hex}`

**서명 방식 (canonical string):**
```
{HTTP_METHOD}\n{path}\n{timestamp}\n{sha256_hex_of_body}
```

## Python 헬퍼 (표준 라이브러리만 사용)

```python
import hmac, hashlib, time, json, urllib.request, urllib.error

SECRET = "<KRAB_HMAC_SECRET>"
BASE_URL = "https://axzymzqxl1.execute-api.ap-northeast-2.amazonaws.com"

def sign(method, path, body_bytes):
    ts = str(int(time.time()))
    body_hash = hashlib.sha256(body_bytes).hexdigest()
    canonical = f"{method.upper()}\n{path}\n{ts}\n{body_hash}"
    sig = hmac.new(SECRET.encode(), canonical.encode(), hashlib.sha256).hexdigest()
    return ts, f"sha256={sig}"

def api(method, path, payload, retries=3, backoff=2):
    for attempt in range(1, retries + 1):
        body = json.dumps(payload, ensure_ascii=False).encode()
        ts, sig = sign(method, path, body)
        req = urllib.request.Request(
            BASE_URL + path, data=body,
            headers={
                "Content-Type": "application/json",
                "x-krab-timestamp": ts,
                "x-krab-signature": sig,
            },
            method=method,
        )
        try:
            with urllib.request.urlopen(req, timeout=20) as res:
                return json.loads(res.read())
        except urllib.error.HTTPError as e:
            err_body = e.read().decode(errors="replace")
            print(f"  HTTP {e.code} (attempt {attempt}/{retries}): {err_body}")
            if e.code < 500 or attempt == retries:
                raise RuntimeError(f"HTTP {e.code}: {err_body}")
        except (urllib.error.URLError, TimeoutError) as e:
            print(f"  Network error (attempt {attempt}/{retries}): {e}")
            if attempt == retries:
                raise
        time.sleep(backoff ** attempt)
```

## 엔드포인트

### 1. 초안 생성

```
POST /threads/drafts
```

```python
payload = {
    "content": "글 내용",
    "source": "nanoclaw",
    # 스레드 연결 시 (선택)
    "replyToProviderId": "<직전_포스트_ID>"
}
draft = api("POST", "/threads/drafts", payload)
job_id = draft["job_id"]
```

### 2. 승인 (발행)

```
POST /threads/approve
```

```python
result = api("POST", "/threads/approve", {
    "jobId": job_id,
    "approvedBy": "786310533439160381",
    "approvedByLabel": "Victor Jaepyo Jo"
})
provider_post_id = result["provider_post_id"]
```

### 3. 단일 포스트 발행 (헬퍼)

```python
def publish_post(content, reply_to=None):
    payload = {"content": content, "source": "nanoclaw"}
    if reply_to:
        payload["replyToProviderId"] = reply_to
    draft = api("POST", "/threads/drafts", payload)
    result = api("POST", "/threads/approve", {
        "jobId": draft["job_id"],
        "approvedBy": "786310533439160381",
        "approvedByLabel": "Victor Jaepyo Jo",
    })
    return result["provider_post_id"]
```

## 스레드(체인) 발행

여러 글을 하나의 스레드로 연결할 때는 각 글이 **직전 글에 reply** 해야 한다.

```python
def publish_thread(posts):
    """
    posts: List[str] — 발행할 글 목록 (순서대로)
    실패 시 이미 발행된 글을 모두 삭제 후 에러 반환
    """
    published = []   # (index, post_id) 튜플
    prev_id = None

    for i, content in enumerate(posts):
        print(f"[{i+1}/{len(posts)}] 발행 중..." + (f" → reply to {prev_id}" if prev_id else " (root)"))
        try:
            post_id = publish_post(content, reply_to=prev_id)
            published.append(post_id)
            prev_id = post_id
            print(f"[{i+1}/{len(posts)}] ✅ {post_id}")
            if i < len(posts) - 1:
                time.sleep(1)
        except Exception as e:
            print(f"[{i+1}/{len(posts)}] ❌ 실패: {e}")
            # 롤백: 이미 발행된 글 전부 삭제
            if published:
                print(f"롤백 중 — {len(published)}개 삭제...")
                rollback_failed = []
                for pid in published:
                    try:
                        api("DELETE", f"/threads/posts/{pid}", {})
                        print(f"  🗑 삭제됨: {pid}")
                    except Exception as re:
                        rollback_failed.append(pid)
                        print(f"  ⚠ 삭제 실패: {pid} — {re}")
                if rollback_failed:
                    print(f"  수동 삭제 필요: {rollback_failed}")
            raise RuntimeError(
                f"[{i+1}/{len(posts)}] 실패로 전체 스레드 롤백됨. 원인: {e}"
            )

    return published
```

### 사용 예시

```python
posts = [
    "첫 번째 글 내용",
    "두 번째 글 내용",
    "세 번째 글 내용 — 재표형의 가재가 작성했습니다. 🦀",
]
ids = publish_thread(posts)
print("발행 완료:", ids)
```

## ⚠️ 알려진 엣지케이스

### 1. 코드 블록 / 백틱 → HTTP 500

Threads API는 마크다운 코드 블록(` ``` `)을 지원하지 않는다.
콘텐츠에 백틱이 포함된 경우 500 에러가 발생한다.

**해결책**: 코드 블록을 일반 텍스트로 대체한다.

```python
# ❌ 이렇게 하면 500 에러
content = """
설치 방법:
```bash
npm install -g gstack
```
"""

# ✅ 이렇게 대체
content = "설치 방법: npm install -g gstack (터미널에서 실행)"
```

### 2. `---` 구분선 → 주의

`---`는 자체적으로는 문제없지만, YAML frontmatter처럼 파싱되는 상황에서 500이 발생할 수 있다.
콘텐츠 분리는 `---` 대신 빈 줄을 사용한다.

### 3. 체인 구조: `prev_id` vs `first_id`

잘못된 패턴 (부채꼴 — 모두 1번에 reply):
```python
# ❌ 모든 글이 첫 번째 글에 달림 → Threads에서 스레드처럼 안 보임
for i, content in enumerate(posts):
    reply_to = first_id if i > 0 else None
```

올바른 패턴 (체인 — 각 글이 직전 글에 reply):
```python
# ✅ 1→2→3→4→5 체인 구조
prev_id = None
for content in posts:
    post_id = publish_post(content, reply_to=prev_id)
    prev_id = post_id
```

### 4. 오래된 포스트에 reply → HTTP 500

일정 시간이 지난 포스트 ID에 `replyToProviderId`를 붙이면 Threads API가 500을 반환한다.
스레드 발행은 한 세션에서 연속으로 완료해야 한다. 중간에 실패하면 전체 롤백 후 재발행하라.

## 전체 플로우 요약

```
글 내용 준비
    ↓
백틱·코드블록 제거 (엣지케이스 #1)
    ↓
publish_thread(posts) 호출
    ↓
성공: provider_post_id 목록 반환
실패: 자동 롤백 → 에러 메시지
```
