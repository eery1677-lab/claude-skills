---
name: meta-instagram-graph-api-pitfalls
description: A checklist of real, hard-won bugs and false leads encountered while integrating Meta's Instagram Graph API (access tokens, business-account linking, reading follower/post/insights data, publishing) into a project, plus the manual Meta Business Suite/Account Center steps around it (creating a Facebook Page, linking it to an Instagram account, figuring out which Page is linked to which account). Use this whenever setting up, debugging, or reviewing Instagram/Facebook/Meta Graph API integration in any project — connecting a business Instagram account, generating or refreshing access tokens, fetching account stats or media, creating or linking a Facebook Page, or figuring out why a "valid-looking" token, stuck UI flow, or app configuration isn't working. This cost a multi-hour live troubleshooting session to work out the first time (wrong flow chosen, misleading error message, permissions hunted through the wrong UI) — read this before repeating that search from scratch.
---

# Meta / Instagram Graph API — Known Pitfalls

## Why this exists

Meta has **two genuinely different ways** to get an Instagram Graph API token, they look similar in the developer dashboard, and picking the wrong one for a given account produces error messages that point you toward completely the wrong fix. This skill exists because that wrong turn cost real hours in one project before the actual root cause (an account with no linked Facebook Page) was found. Read section 1 first — it decides everything else.

## 1. There are two unrelated OAuth flows — pick based on whether a Facebook Page exists

| | (a) "Facebook 로그인이 포함된 API 설정" | (b) "Instagram 로그인이 포함된 API 설정" |
|---|---|---|
| Also called | Facebook Login for Business / Instagram Graph API (classic) | Instagram API with Instagram Login / Business Login for Instagram |
| Requires | Instagram account linked to an actual **Facebook Business Page** | Nothing — just the Instagram professional account itself |
| API host | `graph.facebook.com` | `graph.instagram.com` |
| How you get the token | `/me/accounts` → pick a Page → that Page's `instagram_business_account` field → its id, or a full OAuth consent flow | App dashboard → Instagram → this wizard page → **"토큰 생성"** button next to the account → login → copy the 60-day token directly |
| Permissions | `instagram_basic`, `pages_show_list`, `pages_read_engagement`, etc. | Token is already scoped to the account; no separate page permission dance |

**Decide which one before writing any code.** Check the target Instagram account's actual linkage first: in the Instagram app/site, go to Settings → Account Center → the account's page → **"관리 중인 페이지"** (Pages managed). If a real business Page shows up there, flow (a) works. If it's empty or only shows an unrelated personal page, **flow (a) is not possible for this account, full stop** — no amount of app configuration, permission requests, or OAuth-scope tweaking fixes it. Go straight to flow (b); it needs no Page at all and is usually faster to set up for small/solo businesses that never bothered creating a Facebook Page.

A very common real-world case: a business has an Instagram professional account but never created a matching Facebook Page (or has an unrelated personal Facebook profile). Flow (a) will simply never work for that account — this isn't a bug or a missing permission, it's a missing prerequisite outside your app's control.

## 2. Don't trust the wording of "Cannot parse access token"

`Invalid OAuth access token - Cannot parse access token` sounds like "you pasted a malformed/truncated string" — and sometimes that is the cause. But it's also exactly what you get when you're calling the right-*looking*-token against the wrong host/flow, or (in the case that cost the most time) when the underlying account genuinely has no Facebook Page connection at all and the token being tested was never a valid Graph API token in the first place (e.g. it came from a third-party embed widget, not Meta directly).

Before spending more time on "did I copy the token correctly" theories: verify the account's real linkage state directly (section 1's check) and confirm where the token string actually came from (a real Meta Developer app's Graph API Explorer / dashboard "토큰 생성" button — not a website-embed widget's own settings panel, which often exposes a similarly-labeled but non-Meta key).

## 3. Facebook posting only ever works on Pages, never personal profiles

Meta has blocked programmatic posting to personal Facebook profiles for years — this isn't a permission you can request, it's not offered by any API version. If the business only has a personal profile and no Page, Facebook auto-posting is **flatly impossible** until a real Facebook Business Page is created first. Don't burn time hunting for a personal-profile posting permission; it doesn't exist.

## 3b. Hashtag search (`ig_hashtag_search`) is also Facebook-Page-only — it does not exist on `graph.instagram.com`

If you're on flow (b) (Instagram Login, no Facebook Page — see section 1) hoping to use hashtag search to discover trending content or competitor posts without needing a Facebook Page: it doesn't work. `GET https://graph.instagram.com/v21.0/ig_hashtag_search?...` returns `"Object with ID 'ig_hashtag_search' does not exist"` — the object itself isn't exposed on that host, full stop, not a permissions issue. Retrying against `graph.facebook.com` with the same flow-(b) token just produces the same "Cannot parse access token" error from section 2, because that token was never valid on that host to begin with.

**Bottom line**: hashtag search, like publishing, is gated behind having a real Facebook Page linked to the account (flow (a)). There is no page-free path to it. If external/competitor content discovery matters for a flow-(b) account, the only real options are creating a Facebook Page (unlocking flow (a) and this endpoint), or falling back to manually-curated competitor account tracking instead of API-driven discovery.

## 4. `graph.instagram.com`'s `/me` field names: `user_id` vs `id`

When calling `/me?fields=user_id,username,...` on the Instagram-Login flow, the field you need for all subsequent `/{id}/media` calls is **`user_id`** (the real Instagram-scoped account ID) — not the bare `id` field, which is a different, app-scoped identifier. Mixing them up doesn't error immediately; it just makes every subsequent media/insights call fail or return nothing, which reads as a permissions problem rather than an ID mix-up.

```python
info_res = requests.get(f"{base}/me", params={
    "fields": "user_id,username,followers_count,media_count", "access_token": access_token,
}, timeout=15)
ig_user_id = info_res.json().get("user_id")   # not "id"
```

## 5. `graph.instagram.com` timestamps break Python's `fromisoformat` on 3.9–3.10

Media timestamps come back like `2026-08-28T10:34:43+0000` — a UTC offset **without a colon**. `datetime.datetime.fromisoformat()` on Python ≤3.10 cannot parse that (it requires `+00:00`) and raises — which, inside a broad `try/except: continue`, fails *silently*: every date-based stat (days since last post, average posting interval) just comes back `None`/`null` with no visible error anywhere. This is very easy to miss because the rest of the response (follower count, likes, comments) parses fine, so the feature looks "mostly working."

**Fix** — normalize before parsing:
```python
ts_norm = ts.replace("Z", "+00:00")
m_off = re.match(r"^(.*[+-]\d{2})(\d{2})$", ts_norm)
if m_off:
    ts_norm = f"{m_off.group(1)}:{m_off.group(2)}"
dt = datetime.datetime.fromisoformat(ts_norm)
```
If any date-derived field in an Instagram feature is suspiciously always-null, check this first before assuming the data itself is missing — fetch one raw media object directly and look at the actual `timestamp` string.

## 6. Refresh the long-lived token proactively — don't wait for it to fail

The token from either flow is long-lived (~60 days) but Meta doesn't tell you the exact expiry up front when you first receive it — assume 60 days from issuance and store that as `expires_at`. Refresh **before** it expires (e.g. within 3 days of the stored expiry), not after a call starts failing:

```python
GET https://graph.instagram.com/refresh_access_token?grant_type=ig_refresh_token&access_token={token}
```

This mirrors the same pattern used for other long-lived-token platforms (e.g. Threads' `th_refresh_token` flow) — check expiry on every real use (or via a daily scheduled job so it happens even if the feature isn't used interactively that day), refresh if close to expiring, and silently keep using the old token if the refresh call itself fails (don't hard-fail a feature just because a refresh attempt errored while the current token is still technically valid).

## 7. When hunting for a specific permission in Graph API Explorer, stop and check the official docs instead

The Explorer's permission picker is a category-based dropdown (Commerce / User Data / Events Groups Pages / Other) that can be incomplete or misleading for a given app depending on exactly which products have been added to it — a permission that official docs say should exist can be simply absent from the picker for reasons that aren't obvious from the UI. Don't spend excessive time scrolling through it hoping to spot the right entry.

Instead: go straight to the current official Meta developer docs page for the specific flow you're using (permission and endpoint names have genuinely changed over time — don't rely on memorized/older names), and prefer the app dashboard's own guided flow (the "Instagram 로그인이 포함된 API 설정" wizard's own **"토큰 생성"** button) over manually reconstructing an OAuth authorize URL by hand when a guided option exists — it handles the permission set correctly without you needing to know the exact current scope names at all.

## 8. Linking an Instagram Business account to a Facebook Page via the web Account Center can get stuck — switch to the mobile app

Flow: Facebook Settings & Privacy → search "Instagram" → 연결된 계정(Linked accounts) → Instagram → 계정 연결 → 연결 → a "Instagram 메시지 설정 선택" (message permission toggle) dialog appears with a "계속" (Continue) button. On the web, this exact button can get **reproducibly stuck** — it visibly registers the click (focus ring, no JS error), but never advances to the next step. Confirmed this wasn't a one-off by retrying from scratch multiple times (closing/reopening the modal, toggling the switch off first, pressing Enter instead of clicking, waiting 10+ seconds) — same result every time, in the same browser session.

**Fix**: do the identical flow in the **Instagram mobile app** instead — 앱 → 계정 센터 → 프로필 및 개인정보 → select the target Instagram profile → 연결된 프로필 → 연결된 계정 → Instagram → 계정 연결 → 연결 → (skip "비즈니스 포트폴리오에 추가"if offered — not required for basic Page linking) → 계속 → enter the Facebook password when prompted → done. The exact same step sequence that hangs on web completes normally on the app. If you hit a stuck "계속"/"Continue" button anywhere in this Account Center linking flow, don't keep retrying the same browser path — switch platforms first.

## 9. To find out which Facebook Page (if any) is actually linked to a given Instagram account, use Meta Business Suite's "Instagram으로 이동" link — not the Account Center's page list

A Facebook account can manage multiple Pages, and Account Center's "관리 중인 페이지" (Pages you manage) list shows *all* of them — it does **not** indicate which one, if any, is the Page actually linked to a *specific* Instagram account. Guessing from the Page's name is unreliable (a Page can be named after an unrelated old project and still be the one that's linked, or vice versa).

The reliable check: open **Meta Business Suite** (business.facebook.com) with that Page selected in the top-left switcher, find "Facebook 페이지 수정 | **Instagram으로 이동**" near the Page name/avatar (a small Instagram icon badge on the avatar and an "N 팔로워" count next to the Instagram icon are also visible here if a link exists), and click it. It navigates to `instagram.com/_u/<username>` — the actual linked Instagram username is right there in the resulting URL. This is the fastest way to disambiguate "which of my several Pages is connected to which of my several Instagram accounts" without trial-and-error through the linking flow itself.

## 10. Facebook Page creation's phone field defaults to a US country code — silently rejects a correctly-formatted local number

When filling in a new Page's contact phone number, the country-code dropdown next to the field defaults to **US+1** regardless of the browser's locale or the address already entered in the same form. Typing a Korean number in its normal format (e.g. `063-274-1677`) into that field fails validation with a generic "올바른 전화번호를 입력하세요" (enter a valid phone number) message — a warning that doesn't say *why*, which reads like a typo problem rather than a country-code problem. Fix: change the dropdown to the correct country first (search "대한민국"/"KR+82"), then re-enter the number **without the leading 0** (`63-274-1677`, not `063-274-1677`) — only then does it validate. Check the country-code dropdown before debugging the number format itself whenever this field rejects an otherwise-correct-looking phone number.

## Reference implementation

A full working implementation of flow (b) — settings storage, `/me` lookup, media fetch, health-metric computation, and the proactive refresh from section 6 — exists in `/Users/kambo/제이멘토와실습/나만의콘텐츠생성기/server.py`: search for `INSTAGRAM_GRAPH_HOST`, `_get_valid_instagram_token`, and `_fetch_instagram_account_health` for the exact, tested code (verified against a real Instagram business account with real follower/post/engagement data before this skill was written).
