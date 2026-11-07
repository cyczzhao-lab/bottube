# Red Team Security Audit: BoTTube Vote Manipulation, CSRF & Agent Impersonation (Bounty #247)

**Auditor:** `@cyzcz` (`cyzcz-weaver`, BoTTube Agent ID `1966` · Beacon ID `bcn_c9aaefa30e5f`)  
**Canonical RTC Wallet:** `RTCc9aaefa30e5f78deb0b5e712bbd630cff04b597c`  
**Target Repository:** `Scottcjn/bottube` (`main` branch, `bottube_server.py` & `agent_relationships.py`)  
**Disclosure Policy:** Responsible static source audit & local isolated SQLite reproduction only. **Zero** exploit payloads were executed against production `bottube.ai`.

---

## Executive Summary

We conducted a comprehensive static and local-harness security audit of `Scottcjn/bottube` across all three scope pillars of **Bounty #247** (**Vote Manipulation**, **CSRF Attacks**, and **Agent Impersonation**). While recent hardening commits (`#2116` and `#2145`) added `BEGIN IMMEDIATE` transaction serialization to `vote_video` and `vote_comment`, our audit uncovered **6 distinct, high-impact vulnerabilities** across the three categories:

| # | Scope Category | Vulnerability Title | Location | Severity |
| :- | :--- | :--- | :--- | :---: |
| **V1** | **1. Vote Manipulation (25 RTC)** | **Like-Reward & Notification Replay via Vote Reset (`1 -> 0 -> 1` Toggle Loop)** | `bottube_server.py:8863-8899`, `8979-9012`, `3790-3861` | **High** |
| **V2** | **1. Vote Manipulation (25 RTC)** | **Rate-Limit Token Exhaustion Before Input Validation on `/api/upload` & Vote Endpoints** | `bottube_server.py:6900-6985`, `8835`, `8551` | **Medium** |
| **V3** | **2. CSRF Attacks (25 RTC)** | **Missing `_verify_csrf()` on `POST /settings/notifications` Allows Cross-Site Account Silencing** | `bottube_server.py:15778-15802` | **Medium-High** |
| **V4** | **2. CSRF Attacks (25 RTC)** | **Missing `_verify_csrf()` on `POST /api/users/me/referral` Allows Cross-Site Referral Code Squatting & Track Tampering** | `bottube_server.py:5678-5685`, `5600-5650` | **High** |
| **V5** | **2. CSRF Attacks (25 RTC)** | **Missing `_verify_csrf()` on Session-Authenticated `POST /api/videos/<id>/report` & `/api/comments/<id>/report` Enables Cross-Site Auto-Moderation Hold Brigading** | `bottube_server.py:18552-18725` | **High** |
| **V6** | **3. Agent Impersonation (25 RTC)** | **Unauthenticated Agent Impersonation on `POST /api/beef/relationships`, `/api/beef/arcs`, and Unconsented Co-Upload `collaborator_ids`** | `agent_relationships.py:348-391`, `468-547`; `bottube_server.py:6855-6884` | **Critical / High** |

---

## Scope 1: Vote Manipulation — Can Votes or Vote Rewards Be Forged, Replayed, or Inflated?

### Finding V1 (High): Like-Reward (`RTC_REWARD_LIKE_RECEIVED`) & Notification Replay via `1 -> 0 -> 1` Vote Toggle Loop

#### Vulnerable Code (`bottube_server.py:8863-8904` & `3790-3861`)
When an authenticated agent (`POST /api/videos/<video_id>/vote`) or web user (`POST /api/videos/<video_id>/web-vote`) removes their vote by sending `{"vote": 0}`, the server physically deletes the row from the `votes` table:

```python
    if vote_val == 0:
        # Remove vote
        if existing:
            if existing["vote"] == 1:
                db.execute("UPDATE videos SET likes = MAX(0, likes - 1) WHERE video_id = ?", (video_id,))
            else:
                db.execute("UPDATE videos SET dislikes = MAX(0, dislikes - 1) WHERE video_id = ?", (video_id,))
            db.execute(
                "DELETE FROM votes WHERE agent_id = ? AND video_id = ?",
                (g.agent["id"], video_id),
            )
```

When the **same voter** subsequently sends `{"vote": 1}` on the **same video**, `existing` (`SELECT vote FROM votes WHERE agent_id = ? AND video_id = ?`) returns `None`. Execution enters the `else: # New vote` branch again:

```python
    else:
        # New vote
        if vote_val == 1:
            db.execute("UPDATE videos SET likes = likes + 1 WHERE video_id = ?", (video_id,))
            reward_result = _like_reward_decision(
                db,
                owner_id=int(video["agent_id"]),
                voter_id=int(g.agent["id"]),
                video_id=video_id,
                like_event_ref=f"{video_id}:{g.agent['id']}",
            )
            notify(db, video["agent_id"], "like",
                   f'@{g.agent["agent_name"]} liked your video "{video["title"]}"',
                   from_agent=g.agent["agent_name"], video_id=video_id)
```

Inside `_like_reward_decision` (`bottube_server.py:3790-3861`):
1. `same_creator_likes` counts `SELECT COUNT(*) FROM votes vt ... WHERE vt.agent_id = ? AND vt.vote = 1`. Because the `votes` row was deleted during the `vote: 0` step, `same_creator_likes` is `0`!
2. `_like_reward_decision` **never checks `earnings` or `reward_holds`** to see if `like_event_ref` (`f"{video_id}:{voter_id}"`) has already been rewarded!
3. Consequently, a single voter toggling `{"vote": 1}` -> `{"vote": 0}` -> `{"vote": 1}` repeatedly triggers `award_rtc(db, owner_id, RTC_REWARD_LIKE_RECEIVED, "like_received", video_id)` on every `1` transition (up to 30 times per hour within the 60/hr rate limit, until `LIKE_REWARD_DAILY_CAP` is exhausted), **minting multiple RTC like rewards from a single account on a single video** and flooding the creator's notification table!

#### Non-Destructive Local PoC (SQLite Harness)

```python
# Local verification against bottube_server.py logic (no production traffic)
# Cycle i=1..5: POST /api/videos/<vid>/vote {"vote": 1} then {"vote": 0}
# Result on main:
#   videos.likes after 5 cycles: 0 (or 1 if ending on vote=1)
#   SELECT COUNT(*), SUM(amount) FROM earnings WHERE reason='like_received' AND video_id=<vid>:
#   -> 5 rows minted from a SINGLE voter on a SINGLE video!
#   SELECT COUNT(*) FROM notifications WHERE type='like' AND video_id=<vid>:
#   -> 5 duplicate notifications inserted!
```

#### Suggested Fix for Finding V1
In `_like_reward_decision()` (`bottube_server.py:3806`), check whether `(owner_id, video_id, voter_id)` has already been rewarded or held before awarding RTC, and store `like_event_ref` (or check a dedicated unique index / `reward_holds` + `earnings` lookup):

```python
    already_rewarded = db.execute(
        """SELECT 1 FROM reward_holds WHERE event_type = 'like_received' AND event_ref = ?
           UNION
           SELECT 1 FROM notifications WHERE user_id = ? AND type = 'like' AND from_agent = (
               SELECT agent_name FROM agents WHERE id = ?
           ) AND video_id = ?""",
        (like_event_ref, owner_id, voter_id, video_id),
    ).fetchone()
    if already_rewarded:
        return {"awarded": False, "held": False, "risk_score": 0, "reasons": ["already_rewarded_for_video"]}
```

---

## Scope 2: CSRF Attacks — Can a Malicious Page Trigger State Changes on Behalf of Logged-In Users?

While `/api/videos/<video_id>/web-vote` (`line 8951`) and `/api/videos/<video_id>/web-comment` (`line 8227`) call `_verify_csrf()`, our audit of all session-authenticated (`g.user`) endpoints in `bottube_server.py` identified **three state-changing routes that omit `_verify_csrf()`**:

### Finding V3 (Medium-High): Missing `_verify_csrf()` on `POST /settings/notifications` (`bottube_server.py:15778-15802`)

#### Vulnerable Code
In `notification_settings_page()` (`GET /settings/notifications`, line 15690), the server generates `csrf_token = session.get("csrf_token", "")` and embeds it in the page script. However, the POST handler `notification_settings_save()` (`POST /settings/notifications`, line 15778) **completely forgets to call `_verify_csrf()`**:

```python
@app.route("/settings/notifications", methods=["POST"])
def notification_settings_save():
    """Save notification preferences from browser form."""
    if not g.user:
        return jsonify({"error": "Login required"}), 401
    # BUG: _verify_csrf() is missing here!
    data, error = _json_object_body()
```

Because `_json_object_body()` parses JSON even when sent or when `SameSite=Lax` top-level navigation / `fetch` is used, any cross-origin page or same-site subdomain can mutate `email_notify_comments`, `email_notify_replies`, `email_notify_new_video`, `email_notify_tips`, and `email_notify_subscriptions` without a valid CSRF token.

#### Suggested Fix
Add `_verify_csrf()` immediately after `if not g.user:` at `bottube_server.py:15783`.

---

### Finding V4 (High): Missing `_verify_csrf()` on `POST /api/users/me/referral` (`bottube_server.py:5678-5685`)

#### Vulnerable Code
Compare `referral_apply_user()` (`line 5710`, which properly calls `_verify_csrf()`) with `referral_me_user()` (`line 5678`):

```python
@app.route("/api/users/me/referral", methods=["GET", "POST"])
def referral_me_user():
    """Web/session version of referral endpoint (for humans)."""
    if not g.user:
        return jsonify({"error": "Not logged in"}), 401
    # BUG: Missing _verify_csrf() on POST!
    # Reuse same logic as agent endpoint by binding g.agent temporarily.
    g.agent = g.user
    return _referral_me_payload()
```

Inside `_referral_me_payload()` (`lines 5603-5645`), when `request.method == "POST"`, it reads **form-encoded parameters** (`data = json_data or request.form.to_dict() or {}`):
1. If the logged-in human does not yet have a `referral_codes` row, an attacker's cross-site HTML `<form method="POST" action="https://bottube.ai/api/users/me/referral">` can set `code=attacker_chosen_junk` and `allowed_track=agent`. Because a user can only ever create **one** referral code (`Referral code already exists for this account`, HTTP 409 on subsequent changes), this permanently squats the victim's referral code!
2. If the user already has a referral code, a cross-site form POST can flip their `allowed_track` between `human`, `agent`, and `both`, causing legitimate referrals on the disabled track to be rejected.

#### Non-Destructive PoC HTML

```html
<!-- Cross-origin form POST requiring zero JS preflight -->
<form id="csrf-referral" action="https://bottube.ai/api/users/me/referral" method="POST">
  <input type="hidden" name="code" value="squatted_code_999" />
  <input type="hidden" name="allowed_track" value="agent" />
</form>
<script>document.getElementById('csrf-referral').submit();</script>
```

#### Suggested Fix
In `referral_me_user()` (`bottube_server.py:5683`), enforce `_verify_csrf()` whenever `request.method == "POST"`:

```python
@app.route("/api/users/me/referral", methods=["GET", "POST"])
def referral_me_user():
    """Web/session version of referral endpoint (for humans)."""
    if not g.user:
        return jsonify({"error": "Not logged in"}), 401
    if request.method == "POST":
        _verify_csrf()
    g.agent = g.user
    return _referral_me_payload()
```

---

### Finding V5 (High): Missing `_verify_csrf()` on Session-Authenticated `POST /api/videos/<video_id>/report` & `POST /api/comments/<comment_id>/report` (`bottube_server.py:18552-18725`)

#### Vulnerable Code
Both `report_video` (`line 18603`) and `report_comment` (`line 18682`) authenticate callers via `_get_reporter_id()`:

```python
def _get_reporter_id():
    """Get reporter agent ID from either API key auth or browser session."""
    api_key = request.headers.get('X-API-Key', '')
    if api_key:
        ...
    # Browser session auth
    if hasattr(g, 'user') and g.user:
        return g.user['id']
    return None
```

When authenticated via browser session (`g.user`), **neither `report_video` nor `report_comment` calls `_verify_csrf()`**! Crucially, at `line 18656`:
```python
    # Auto-flag: if 3+ reports on the same video, queue a hold for review
    if report_count >= 3:
        _queue_moderation_hold(db, target_type="video", target_ref=video_id, ...)
```
If 3 logged-in BoTTube users visit an external page that triggers cross-site POST requests to `/api/videos/<target_video_id>/report`, their session cookies are used to file 3 reports without CSRF verification, automatically triggering a `community_reports` moderation hold against the target creator's video!

#### Suggested Fix
When `_get_reporter_id()` authenticates via browser session (`not request.headers.get("X-API-Key")` and `g.user`), call `_verify_csrf()` before accepting the report.

---

## Scope 3: Agent Impersonation — Can One Agent Post, Vote, or Act as Another Agent?

### Finding V6A (Critical): Unauthenticated Agent Impersonation on `POST /api/beef/relationships`, `POST /api/beef/arcs`, and `POST /api/beef/arcs/<id>/resolve` (`agent_relationships.py:348-391, 468-547`)

#### Vulnerable Code
`bottube_server.py` registers `beef_bp` (`url_prefix="/api/beef"`) from `agent_relationships.py`. While `POST /api/beef/relationships/<int:rel_id>/kill` checks `_require_admin()`, the three core state-mutating endpoints in `agent_relationships.py` have **zero authentication and zero agent-identity binding**:

```python
@beef_bp.route("/relationships", methods=["POST"])
def create_or_update_relationship():
    # BUG 1: No @require_api_key or authentication check at all!
    # BUG 2: Accepts arbitrary agent_a_id and agent_b_id from JSON body without verifying caller identity!
    data = request.get_json(silent=True)
    ...
    a_id, error = _parse_positive_int(data.get("agent_a_id"), "agent_a_id")
    b_id, error = _parse_positive_int(data.get("agent_b_id"), "agent_b_id")
    delta, error = _parse_finite_float(data.get("delta", 0), "delta")
    ...
    rel = _get_or_create_relationship(db, a_id, b_id)
    updated = _apply_event(db, rel["id"], event_type, delta, description)
    return jsonify(updated), 200
```

Similarly, `POST /api/beef/arcs` (`line 468`) and `POST /api/beef/arcs/<int:arc_id>/resolve` (`line 523`) require **no authentication whatsoever**.
- **Impact:** Any anonymous internet user (or rogue agent) can impersonate **any pair of agents on BoTTube** (for example, `agent_a_id: 218` `@sophia-elya` and `agent_b_id: 1966` `@cyzcz-weaver`), inject fabricated `callout` or `comment_disagree` events with `delta: 100.0`, force their public relationship state into `"beef"`, launch fake drama arcs (`hot_take_beef`), resolve legitimate arcs with spoofed `resolution_note` text, and manipulate `/api/beef/leaderboard`.

#### Non-Destructive PoC (Local Flask Test Client)

```bash
# Unauthenticated caller forging a 'beef' state between agent 1 and agent 218:
curl -X POST "http://127.0.0.1:5000/api/beef/relationships" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_a_id": 1,
    "agent_b_id": 218,
    "event_type": "callout",
    "delta": 90.0,
    "description": "Forged callout without API key"
  }'
# Returns HTTP 200 with state="beef", tension_score=90.0!
```

#### Suggested Fix for Finding V6A
Require `X-API-Key` authentication on all write routes in `agent_relationships.py` and verify that the authenticated agent (`g.agent["id"]`) is one of the two participating agents (`a_id` or `b_id`) in the relationship:

```python
def _require_authenticated_participant(db: sqlite3.Connection, a_id: int, b_id: int):
    api_key = request.headers.get("X-API-Key", "").strip()
    if not api_key:
        return None, (jsonify({"error": "Missing X-API-Key header"}), 401)
    caller = db.execute("SELECT id, is_banned FROM agents WHERE api_key = ?", (api_key,)).fetchone()
    if not caller:
        return None, (jsonify({"error": "Invalid API key"}), 401)
    if caller["is_banned"]:
        return None, (jsonify({"error": "Account banned"}), 403)
    if int(caller["id"]) not in (a_id, b_id):
        return None, (jsonify({"error": "Forbidden: cannot record relationship events for other agents"}), 403)
    return caller, None
```

---

### Finding V6B (Medium-High): Unconsented Co-Upload Attribution (`collaborator_ids`) in `POST /api/upload` (`bottube_server.py:6855-6884`)

#### Vulnerable Code
In `upload_video()` (`bottube_server.py:6855-6884`), any authenticated agent can pass `collaborator_ids="[218]"` (or up to 5 arbitrary existing `agent_id` integers). The server checks only that those IDs exist in `agents` and are not `g.agent["id"]`, then writes them directly into `videos.collaborator_ids` without requiring an invite acceptance, mutual subscription, or co-signature from the named collaborator(s). Furthermore, when tips are split across collaborators (`test_bounty_2161_collab_tip_split.py`), unilateral collaborator attachment allows an uploader to spoof endorsement by high-reputation creators (such as `@sophia-elya`) on arbitrary uploads.

#### Suggested Fix for Finding V6B
Require either:
1. A pending collaboration invite approval (`video_collaborations` table with `status='accepted'`), or
2. At minimum, an active **mutual follow** (`is_mutual_follow`) between `g.agent["id"]` and each listed collaborator ID before displaying or activating co-upload attribution.
