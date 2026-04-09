---
name: common-pitfalls
description: Use when encountering Python dependency errors, React state issues, CSS ellipsis problems, Ant Design Upload/Table issues, Zustand persistence problems, react-dnd drag vibration, data sync design decisions, AI/OCR text matching failures, cross-platform Unicode (NFD/NFC) issues, Pydantic model_dump pitfalls, FastAPI request format mismatches, database safety concerns, localStorage limitations, import data uniqueness, VPS/remote server operations, LiveKit/Gemini/OpenAI realtime voice AI latency tuning, Japanese speech_end_offset/silence_duration settings, SIP NAT traversal with Docker Desktop for Mac, cloud PBX vs SIP trunk compatibility issues, LLM API cost estimation (thinking tokens), model deprecation risks, Claude Agent/Sub-agent file passing, Express dashboard security (localhost binding), SQLite WAL mode transfer, cron automation with AI bots, Gemini Vision API multi-image accuracy degradation, PDF multi-page processing pitfalls, or Docker Compose volume reference loss
---

# Common Development Pitfalls

## Overview

Frequently encountered bugs and their solutions from real debugging sessions. Reference when facing similar symptoms.

## Python Dependencies (Python 3.13+)

| Error | Cause | Fix |
|-------|-------|-----|
| `No module named 'email_validator'` | pydantic EmailStr needs it | `pydantic[email]>=2.6.1` |
| `No module named 'greenlet'` | async SQLAlchemy needs it | `greenlet>=3.0.0` |
| `AttributeError: module 'bcrypt' has no attribute '__about__'` | bcrypt/passlib incompatibility | `bcrypt==4.0.1` (pin version) |

## React State Management

### Object Reference Comparison Fails

**Symptom:** `filter((o) => o !== record)` doesn't remove item

**Cause:** New objects created in render (e.g., `map((o) => ({...o, _key: ...}))`) have different references

**Fix:** Compare by unique key:
```typescript
const getOrderKey = (o: Order) => `${o.id}-${o.sku}-${o.name}`;
setOrders(prev => {
  const idx = prev.findIndex(o => getOrderKey(o) === getOrderKey(record));
  return idx !== -1 ? [...prev.slice(0, idx), ...prev.slice(idx + 1)] : prev;
});
```

### Zustand Persist Missing Data

**Symptom:** Page refresh loses user data, permissions fail

**Cause:** `partialize` doesn't include all needed fields

**Fix:** Include all required fields:
```typescript
partialize: (state) => ({
  token: state.token,
  isAuthenticated: state.isAuthenticated,
  user: state.user,  // Don't forget this!
}),
```

## CSS Text Ellipsis

### Ellipsis Not Working in Flexbox

**Symptom:** `text-overflow: ellipsis` ignored in flex child

**Cause:** Flex items expand to content by default

**Fix:** Add `minWidth: 0` to flex child:
```css
.flex-child {
  flex: 1;
  min-width: 0;  /* Required for ellipsis in flexbox */
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
```

### Ant Design Table Ellipsis Not Working

**Symptom:** `ellipsis: true` column shows full text

**Cause:** Column needs explicit `width`

**Fix:**
1. Add `width` to column definition
2. For custom cell components, add to `td`:
```typescript
const tdStyle = isEllipsis ? { maxWidth: 0, overflow: 'hidden' } : {};
return <td {...props} style={{...props.style, ...tdStyle}}>{children}</td>;
```

## Ant Design Upload

### originFileObj is undefined

**Symptom:** "No valid files" error after upload

**Cause:** Direct file push doesn't set `originFileObj`

**Fix:** Wrap file properly in `beforeUpload`:
```typescript
beforeUpload: (file) => {
  const uploadFile: UploadFile = {
    uid: file.uid || `${Date.now()}-${file.name}`,
    name: file.name,
    size: file.size,
    type: file.type,
    originFileObj: file as unknown as UploadFile['originFileObj'],
  };
  setFileList(prev => [...prev, uploadFile]);
  return false;
},
```

## react-dnd Drag and Drop

### Items Vibrate/Shake During Drag

**Symptom:** Other items vibrate when dragging to reorder

**Cause:** `hover` handler fires too frequently, updating state on every pixel movement

**Fix:** Only trigger move when mouse crosses element center:
```typescript
const [, dropForReorder] = useDrop({
  accept: ItemTypes.FIELD,
  hover: (item, monitor) => {
    if (!ref.current) return;
    const dragIndex = item.index;
    const hoverIndex = index;
    if (dragIndex === hoverIndex) return;

    const rect = ref.current.getBoundingClientRect();
    const hoverMiddleY = (rect.bottom - rect.top) / 2;
    const clientOffset = monitor.getClientOffset();
    if (!clientOffset) return;
    const hoverClientY = clientOffset.y - rect.top;

    // Only move when crossing center line
    if (dragIndex < hoverIndex && hoverClientY < hoverMiddleY) return;
    if (dragIndex > hoverIndex && hoverClientY > hoverMiddleY) return;

    onMoveField(dragIndex, hoverIndex);
    item.index = hoverIndex;
  },
});
```

**Also add CSS transition for smoothness:**
```typescript
style={{
  transition: 'transform 0.15s ease, opacity 0.15s ease',
  opacity: isDragging ? 0.4 : 1,
}}
```

## Ant Design Table Advanced

### editable + expandable Column Conflict

**Symptom:** Adding `editable: true` to columns breaks header rendering

**Cause:** Header generation logic may overwrite special column properties

**Fix:** Don't blindly apply properties to all columns. Check each column's existing properties before modification.

### Field Visible but Column Not Showing

**Symptom:** Field is checked in settings (visibleColumns), but column doesn't appear in table

**Cause:** Column definition missing in `allColumnDefinitions` object

**Fix:** When adding new fields to field definitions (e.g., `constants/fields.ts`), also add corresponding column definition:
```typescript
// In allColumnDefinitions object
note: {
  title: '備考',
  dataIndex: 'note',
  key: 'note',
  width: 200,
  ellipsis: true,
  editable: true,
  inputType: 'text',
},
```

**Checklist when adding new field:**
1. Add to field definitions (`constants/fields.ts` or API)
2. Add column definition to `allColumnDefinitions`
3. Add to TypeScript type if needed (`OrderRow` etc.)

## Architecture Design

### Data Sync: One-Way vs Bidirectional

**Problem:** Bidirectional sync (Template ↔ System) causes confusion about source of truth on startup

**Solution:** Prefer one-way sync (Template → System):
- Template file is single source of truth
- System reads from template on startup
- UI doesn't have add/delete field buttons
- Changes require editing template + restart

**Benefits:**
- Clear ownership of data
- No sync conflict resolution needed
- Simpler mental model for users

## useState Initializer Stale Data

### Component Shows Previous Data After Prop Change

**Symptom:** Modal content shows data from previous selection, not current

**Cause:** `useState(() => initialValue)` runs ONCE on mount. If parent changes props without unmounting the component, state keeps old data.

**Fix:** Add `key` prop to force remount:
```tsx
// Parent
<MappingEditor key={selectedChannel.id} initialMappings={selectedChannel.mappings} />
```

**When this happens:**
- Ant Design Modal does NOT destroy children on close (`open=false` hides, doesn't unmount)
- Conditional rendering `{selected && <Component />}` doesn't unmount if `selected` never becomes null
- Tabs that keep content mounted

## Mutually Exclusive Field Groups

### Related Fields Should Not Be Simultaneously Active

**Symptom:** User maps `full_address` but `prefecture`/`city`/`street_address` remain mapped

**Cause:** No exclusion logic between field groups

**Fix:** In drop/change handler, clear conflicting group:
```typescript
const DETAIL_FIELDS = ['prefecture', 'city', 'street_address', 'address_extra'];

const handleDrop = (targetKey: string, sourceField: string) => {
  setMappings(prev => {
    const updated = { ...prev, [targetKey]: sourceField };
    if (targetKey === 'full_address') {
      DETAIL_FIELDS.forEach(key => { delete updated[key]; });
    } else if (DETAIL_FIELDS.includes(targetKey)) {
      delete updated['full_address'];
    }
    return updated;
  });
};
```

**Pattern:** Any time two field groups represent the same data at different granularities, enforce mutual exclusion.

## Shared State Scope Changes

### Moving State From Per-Item to Global

**Symptom:** Removing per-item state breaks references in other files; constants deleted but still imported

**Cause:** Incomplete migration - removed the source but not all consumers

**Fix:** Before moving state scope:
1. Grep for ALL references to the old storage key/constant/type
2. Update every consumer, not just the primary file
3. Update initial values that depended on the old source
4. Run lint to catch unused/missing imports

## AI/OCR Text Fuzzy Matching

### Exact Matching Fails Against AI-Extracted Text

**Symptom:** Keyword registered as `YAC川崎（新）` doesn't match Gemini Vision API output `YAC|川崎(新)` or `YAC||崎(新)`

**Cause:** AI/OCR output is noisy and inconsistent:
- Full-width `（）` vs half-width `()`
- Random separators: `/`, `|`, `||`
- Missing characters: `川` dropped entirely between requests
- Same PDF produces different text on each API call

**Fix — 2-layer matching strategy:**

**Layer 1: Aggressive normalization** — strip everything except alphanumeric + CJK:
```python
import re, unicodedata

def normalize_for_match(text: str) -> str:
    normalized = unicodedata.normalize('NFC', text)
    # Full-width → half-width
    result = []
    for ch in normalized:
        cp = ord(ch)
        if 0xFF01 <= cp <= 0xFF5E:
            result.append(chr(cp - 0xFEE0))
        else:
            result.append(ch)
    normalized = ''.join(result)
    # Keep ONLY alphanumeric + CJK (strips |, /, (), spaces, etc.)
    normalized = re.sub(r'[^a-zA-Z0-9\u3040-\u309f\u30a0-\u30ff\u4e00-\u9fff]', '', normalized)
    return normalized.lower()
```

**Layer 2: Fuzzy subsequence matching** — handles missing characters:
```python
def fuzzy_subsequence_ratio(keyword: str, location: str) -> float:
    """keyword chars appearing in order within location"""
    if not keyword:
        return 0.0
    j = 0
    matched = 0
    for c in keyword:
        while j < len(location):
            if location[j] == c:
                matched += 1
                j += 1
                break
            j += 1
    return matched / len(keyword)

# Usage: try exact substring first, fallback to fuzzy (≥60%)
norm_kw = normalize_for_match(keyword)    # "yac川崎新"
norm_loc = normalize_for_match(location)  # "yac崎新" (川 missing)
if norm_kw in norm_loc:
    return True  # exact match
ratio = fuzzy_subsequence_ratio(norm_kw, norm_loc)
if ratio >= 0.6:
    return True  # yac→崎→新 = 5/6 = 83% match
```

**Key lessons:**
- Never use exact string matching against AI/OCR output
- Normalization alone is insufficient — AI can drop characters entirely
- Log both original and normalized values for debugging
- Each Gemini API call may return different text for the same PDF

### Debug Logging Placement Causes UnboundLocalError

**Symptom:** Adding `logger.info()` before variable initialization causes `UnboundLocalError` → 500 on every request

**Cause:** Debug logging referencing `api_logger` placed before `api_logger = logging.getLogger(...)` definition

**Fix:** Always define logger at the TOP of the function, before any code that might use it:
```python
async def process_files(...):
    import logging
    api_logger = logging.getLogger("my_api")
    # NOW safe to use api_logger anywhere below
```

**Pattern:** When adding debug logging to existing functions, define the logger immediately after the function signature, before any business logic.

## Cross-Platform: macOS ↔ Linux/Docker

### Japanese Filenames Not Found on Linux

**Symptom:** `FileNotFoundError` on Linux/Docker for files that exist and work fine on macOS

**Cause:** macOS stores filenames in Unicode NFD (decomposed form). For example, `ゆうパケット` becomes `ゆうハ+゜+ケ+ッ+ト` (dakuten separated). On macOS, the OS auto-converts NFD↔NFC transparently, so you never notice. On Linux, no such conversion happens — NFC string ≠ NFD filename.

**Fix:** Normalize both sides before comparison:
```python
import unicodedata
from pathlib import Path

def resolve_path(directory: Path, filename: str) -> Path:
    direct = directory / filename
    if direct.exists():
        return direct

    nfc_name = unicodedata.normalize('NFC', filename)
    nfd_name = unicodedata.normalize('NFD', filename)

    for entry in directory.iterdir():
        entry_nfc = unicodedata.normalize('NFC', entry.name)
        if entry_nfc == nfc_name or entry.name == nfd_name:
            return entry

    return direct  # Let caller handle the error
```

**Key lesson:** Any code that handles CJK filenames and runs on both macOS and Linux MUST normalize Unicode. This is invisible on macOS — only fails in production (Linux/Docker).

## Pydantic / FastAPI

### dict.get(key, default) Returns None Instead of Default

**Symptom:** `TypeError: can only concatenate str (not "NoneType") to str` when accessing Pydantic model fields

**Cause:** `model.model_dump()` includes ALL fields — unset fields have key present with value `None`. `dict.get('key', '')` returns `None` (not `''`) because the key exists.

```python
# Pydantic model with optional field
class Order(BaseModel):
    city: Optional[str] = None

order = Order()
d = order.model_dump()  # {'city': None}
d.get('city', '')       # Returns None, NOT ''!
```

**Fix:** Use `or` pattern:
```python
value = d.get('city') or ''           # None → ''
value = (d.get('name') or '')[:30]    # Safe truncation
combined = (d.get('city') or '') + (d.get('address') or '')
```

### Frontend Request Format Mismatch

**Symptom:** 422 Validation Error on API calls that worked before

**Cause:** Frontend sends FormData (multipart) but backend expects JSON body (Pydantic model), or vice versa. Often happens when API is refactored (e.g., file upload → Base64 string) but frontend isn't updated simultaneously.

**Fix:** Always verify both sides match:

| Backend expects | Frontend must send |
|---|---|
| `request: MyModel` (Pydantic) | `JSON` with `Content-Type: application/json` |
| `file: UploadFile` (FastAPI) | `FormData` with `Content-Type: multipart/form-data` |

**Pattern:** When changing API request format, update frontend AND backend in the same commit. Test the specific endpoint, not just the bulk/batch version.

## Database Safety

### Never Delete DB Files

**Symptom:** All data lost after "quick fix" (password reset, schema change, etc.)

**Cause:** Deleting `.db` file and restarting recreates empty database. Git doesn't help — `.gitignore` excludes `*.db`.

**Rules:**
1. **Password reset** → Use SQL UPDATE via dedicated script, never delete DB
2. **Schema change** → Use ALTER TABLE in migration code, not drop+recreate
3. **Docker volumes** → NEVER use `docker compose down -v` (deletes volumes including DB)
4. **Backups** → Set up automated backups (cron + cloud storage). Git is NOT a DB backup

```python
# Example: reset_password.py (safe approach)
import asyncio, sys
from app.models import async_session, User
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"])

async def reset(username, new_password):
    async with async_session() as session:
        user = await session.execute(
            select(User).where(User.username == username)
        )
        user = user.scalar_one()
        user.hashed_password = pwd_context.hash(new_password)
        await session.commit()
```

### SQLAlchemy create_all() Doesn't ALTER Existing Tables

**Symptom:** New column added to model but doesn't appear in database

**Cause:** `create_all()` only creates new tables. It never modifies existing ones.

**Fix:** Add explicit migration in `init_db`:
```python
# In init_db()
try:
    await session.execute(text(
        "ALTER TABLE my_table ADD COLUMN new_field TEXT DEFAULT ''"
    ))
    await session.commit()
except Exception:
    await session.rollback()  # Column already exists
```

### Cascade Delete Destroys Related Data

**Symptom:** Deleting a parent record silently removes all child records

**Cause:** `cascade="all, delete-orphan"` on relationships

**Fix:** Before any parent deletion, show affected count to user:
```python
child_count = await session.execute(
    select(func.count()).where(Child.parent_id == parent_id)
)
# Show: "Deleting this will also remove {count} related records"
```

## Data Import / Batch Processing

### File-Internal Duplicates Missed

**Symptom:** Same record imported twice, causing double-counted inventory

**Cause:** Only checking DB for duplicates, not checking within the import file itself

**Fix:** Always validate uniqueness in BOTH sources:
```python
# 1. Check file-internal duplicates
seen_keys = set()
for row in import_data:
    key = (row['brand'], row['model'])
    if key in seen_keys:
        errors.append(f"Duplicate in file: {key}")
    seen_keys.add(key)

# 2. Check DB duplicates
existing = await get_existing_keys(session)
for key in seen_keys:
    if key in existing:
        errors.append(f"Already exists in DB: {key}")
```

## Frontend Storage

### localStorage Not Shared Across Devices

**Symptom:** Settings saved on one device don't appear on another

**Cause:** localStorage is browser-specific, not synced

**Decision guide:**
| Data type | Storage | Reason |
|---|---|---|
| Auth token | localStorage | Browser-specific is fine |
| Pre-login cache (theme) | localStorage + DB | Need before auth, sync after |
| User preferences | Server DB | Must sync across devices |
| App config (shared) | Server file/DB | All users need same data |

## VPS / Remote Server Safety

### Never Modify SSH Configuration Remotely

**Symptom:** Locked out of VPS after modifying `authorized_keys` or `sshd_config`

**Cause:** Terminal line breaks can corrupt SSH key format. Network restrictions may prevent Claude Code from SSH connections, leading to misdiagnosis.

**Rules:**
1. **NEVER touch `~/.ssh/authorized_keys`** on remote servers
2. **NEVER modify `sshd_config`** remotely
3. If SSH fails from Claude Code, it's likely a network restriction — don't "fix" the server
4. Deploy via user's terminal, not Claude Code SSH

## Realtime Voice AI (LiveKit + Gemini/OpenAI)

### speech_end_offset は日本語では1.5秒が適正値

**Symptom:** AIがユーザーの発話中に割り込む。日本語の句間ポーズを発話終了と誤検出

**Cause:** `speech_end_offset`（Gemini）や`silence_duration_ms`（OpenAI）を短くしすぎると、日本語の自然な間（0.5〜1秒程度）を「話し終わった」と判定する

**テスト結果:**

| 設定値 | 結果 |
|---|---|
| 0.5秒 / 300ms | 頻繁に割り込み。文の途中で応答開始 |
| 1.0秒 / 500ms | 改善するが依然割り込みあり |
| 1.5秒 / 700ms | 割り込み大幅減。デフォルト（約2秒）より速く応答 |

**Fix:**
```python
# Gemini
from google.genai import types
realtime_input_config=types.RealtimeInputConfig(
    automatic_activity_detection=types.AutomaticActivityDetection(
        speech_end_offset=types.Duration(seconds=1, nanos=500_000_000),  # 1.5秒
    ),
)

# OpenAI
from openai.types.beta.realtime.session import TurnDetection
turn_detection=TurnDetection(
    type="server_vad",
    threshold=0.5,
    prefix_padding_ms=300,
    silence_duration_ms=700,  # 0.7秒
)
```

**Key lesson:** 英語ベースのデフォルト値をそのまま日本語に適用してはいけない。日本語は句間ポーズが長いため、攻めすぎると会話が成立しない。

### System Prompt の長さがリアルタイム音声APIのレイテンシに直結する

**Symptom:** AIの初回応答が遅い。ユーザーが話し終わってから数秒待たされる

**Cause:** Realtime APIはsystem_promptを毎ターンの推論コンテキストに含める。長いプロンプト = 推論時間増加

**Fix:** 電話サポート用プロンプトは1,000文字以内を目標に圧縮。ルールの本質を維持しつつ箇条書きを簡潔にする。

```yaml
# Before: 約2,200文字 — 冗長な説明、対話フローの詳細例
# After:  約700文字  — 箇条書きのみ、例示削除

# 圧縮のコツ:
# - 「〜してください。〜の場合は〜してください。」→ 箇条書き1行
# - 対話フロー例は削除（AIは理解している）
# - 重複ルールを統合
```

**同時に効果がある設定:**
- `temperature` 0.8→0.4（生成速度向上、電話対応では一貫性が重要）
- RAG検索結果 `n_results` 3→2（ツール結果のトークン数削減）

### Docker Desktop for Mac で SIP が動かない

**Symptom:** Asterisk（Docker内）がSIPサーバーにREGISTER成功するが、着信のSIP INVITEが一切届かない

**Cause:** Docker Desktop for MacはVM経由のネットワークを使用。`network_mode: host`はVM内のネットワークを指し、Mac本体ではない。さらにポートマッピングも`vpnkit`プロキシ層を経由するため、SIPのUDP通信（ペイロード内IP/Portとパケット送信元の一致要件）と相性が極めて悪い。

**Fix:** ローカルMacでSIPをテストする方法はない（現実的ではない）。Linux VPSにデプロイする。

```text
# ローカルMacの問題:
# 1. network_mode: host → VM内のみ、Mac本体ではない
# 2. ブリッジ+ポートマッピング → vpnkitプロキシで二重NAT
# 3. ルーターNAT + Docker NAT = SIPパケットがロスト

# 解決策の優先順位:
# 1. Linux VPSにデプロイ（推奨・確実）
# 2. OrbStackを使う（Dockerより透過的なネットワーク）
# 3. Asteriskをmac本体にネイティブインストール
```

**Key lesson:** SIP/RTPのようなUDPプロトコルはNAT越えが複雑。Docker Desktop for Macでの動作を前提にしない。開発初期からVPSでの動作確認を計画に含める。

### ナイセンクラウド等のクラウドPBXはSIPトランクではない

**Symptom:** クラウドPBXにREGISTER成功するが、着信時にSIP INVITEが送信されない

**Cause:** 多くのクラウドPBXサービス（ナイセンクラウド等）は標準SIP INVITEによる着信配信をサポートしていない。アプリ専用のプッシュ通知（Acrobits SIPIS等）で着信を配信しており、Asterisk等の標準SIPクライアントでは着信を受けられない。

**Fix:** AI自動応答にはTwilio SIP Trunk等の標準SIPトランクプロバイダーを使用する。

**確認すべき項目（開発着手前）:**
1. SIPサービスが標準SIP INVITEで着信を配信するか？
2. IP認証ベースのSIPトランク接続に対応しているか？
3. Asterisk等の汎用SIPクライアントでの動作実績があるか？

**Key lesson:** REGISTERが成功する≠着信を受けられる。クラウドPBXとSIPトランクは別物。Phase 0で必ず確認すること。

## LLM API Integration

### 思考トークン（Thinking Tokens）でコストが50倍になる

**Symptom:** Gemini 2.5 Flashで842件分析を「約$0.03」と見積もったが実際は$7.76

**Cause:** Gemini 2.5 Flashはデフォルトで思考トークンを大量生成。1リクエストあたり入力388 + 出力101に対し、**思考2,599トークン**（全体の84%）。思考トークンの単価は入力/出力の数倍。

**Fix:** コスト見積もり前に必ず1件の実測:
```javascript
const result = await model.generateContent({...});
const u = result.response.usageMetadata;
console.log('入力:', u.promptTokenCount);
console.log('出力:', u.candidatesTokenCount);
console.log('思考:', u.thoughtsTokenCount);  // ← これを見落とす
console.log('合計:', u.totalTokenCount);
// 見積もり = 合計 × 件数 × 単価
```

**思考を無効化する場合（REST API直接）:**
```javascript
generationConfig: {
  thinkingConfig: { thinkingBudget: 0 }  // 思考トークン完全ゼロ
}
// 注: SDK経由だと効かない場合がある。REST API直接呼出しが確実
```

**Key lesson:** 分類・抽出タスクに思考は不要。`thinkingBudget: 0`でコスト1/250に削減可能。

### LLMモデルは突然廃止される

**Symptom:** `gemini-2.0-flash`が404 Not Found「no longer available to new users」

**Cause:** Googleがモデルを予告なく新規利用停止にした

**Fix:**
```typescript
// NG: ハードコード
const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });

// OK: 設定ファイルで管理
const model = genAI.getGenerativeModel({ model: config.geminiModel });
// .env: GEMINI_MODEL=gemini-2.5-flash
```

**Key lesson:** 外部APIのモデル名は環境変数で管理。ハードコードしない。

### サブスク内で完結する方法を最優先に検討する

**Symptom:** 外部API（Gemini等）のコスト・レート制限・モデル廃止に振り回される

**Cause:** 最初から外部APIありきで設計

**Fix — 優先順位:**
1. **サブスク内で完結** — OpenClaw/Claude Codeのボットに委任（コスト0）
2. **Claude Agent並列** — Sonnet Agentを並列起動してバッチ処理（サブスク内）
3. **外部API（思考なし）** — どうしても必要な場合のみ

**Key lesson:** ユーザーが既にサブスクを持つサービスで処理できないか最初に検討。外部APIは最後の手段。

## Claude Agent / Sub-agent

### Sub-agentに渡すJSONファイルはpretty-printする

**Symptom:** Sub-agentが「ファイルが10,000トークン制限を超えている」と報告。offset/limitも効かない

**Cause:** `JSON.stringify(data)`は1行のJSONを出力。Readツールは行単位でoffset/limitを適用するため、1行=全体となり分割読みできない

**Fix:**
```javascript
// NG: 1行JSON — Sub-agentのReadツールで読めない
fs.writeFileSync('data.json', JSON.stringify(data));

// OK: pretty-print — 行分割でoffset/limitが効く
fs.writeFileSync('data.json', JSON.stringify(data, null, 1));
```

**Key lesson:** Agent間でファイルを受け渡す場合、pretty-print JSONを使う。巨大ファイルは事前に分割（50〜80件/ファイル）。

### 分類タスクにはSonnet、推論タスクにはOpus

**Symptom:** Opusで842件の分析を実行 → $3.20。Sonnetなら$1.92

**Cause:** 分類・抽出・スコアリングは「考える」必要が少なく、Sonnetで十分な精度

**判断基準:**
| タスク | 推奨モデル | 理由 |
|---|---|---|
| 分類・ラベル付け | Sonnet | パターンマッチに近い |
| JSON抽出・構造化 | Sonnet | スキーマ固定で精度差なし |
| 複雑な推論・設計 | Opus | 深い思考が必要 |
| コード生成・リファクタ | Opus | 文脈理解が重要 |

## Express Dashboard Security

### 内部ツールでもlocalhostバインドは最初から

**Symptom:** ダッシュボードが`0.0.0.0:3000`で全インターフェースにリッスン、外部からアクセス可能

**Cause:** Express `app.listen(PORT)` はデフォルトで全インターフェースにバインド

**Fix — 3重防御:**
```typescript
// 1. localhostバインド
const BIND = process.env.DASHBOARD_BIND || "127.0.0.1";
app.listen(Number(PORT), BIND, () => { ... });

// 2. UFWファイアウォール
// $ ufw deny 3000/tcp

// 3. SSHトンネルでのみアクセス
// $ ssh -L 3000:127.0.0.1:3000 vps
```

**Key lesson:** 「内部ツールだから」で後回しにしない。最初からlocalhostバインド。

## SQLite Transfer (VPS Deploy)

### WALモード使用中のDB転送はテーブルが見えないことがある

**Symptom:** rsyncでjobs.dbを転送したが、VPS側で「no such table」エラー

**Cause:** SQLiteのWALモード（`journal_mode = WAL`）では、未コミットの変更が`.db-wal`ファイルに保持される。`.db`ファイルだけ転送すると不完全

**Fix:**
```bash
# 送信側: SQL dumpで確実にエクスポート
sqlite3 local.db ".dump table_name" > dump.sql

# 受信側: インポート
sqlite3 remote.db < dump.sql
```

**Key lesson:** SQLiteのDB転送はファイルコピーではなく`.dump`→インポートが確実。

## Cron Automation with AI Bots

### cronのトリガーはシンプルに、処理はAIボットに委任

**Symptom:** シェルスクリプトで「JSON抽出→Claude CLI→結果パース→DB保存」の複雑なパイプライン → 途中で壊れやすい

**Cause:** 中間ステップが多いほど壊れやすい。エラーハンドリングもスクリプト側で全て書く必要がある

**Fix:**
```bash
# NG: 複雑なパイプライン
sqlite3 db "SELECT ..." > /tmp/data.json
claude --model sonnet --print -p "..." > /tmp/result.json
python3 -c "import json; ..." # パース&保存

# OK: AIボットに1メッセージ送るだけ
openclaw agent --agent analyzer -m "新着案件を分析して" --deliver
```

**前提:** ボットのワークスペースにCLAUDE.mdで手順を記述しておく。ボットが手順を読み、自律的に処理・エラーハンドリング・完了報告を行う。

**Key lesson:** AIボットが使える環境なら、スクリプトで全手順を書くよりボットに委任する方がメンテナンス性が高い。

## Gemini Vision API / PDF Processing

### 画像PDFの複数ページを一括送信すると特定フィールドの精度が著しく低下する

**Symptom:** 2ページの画像PDFをGemini Vision APIに一括送信すると、`delivery_location`（納品先）に店舗名ではなく商品コード（`CTJ J`）が入る。同じPDFを1ページずつ別々に送信すると正確に抽出される。

**Cause:** Gemini Vision APIは複数画像を同時に処理すると、各画像のフィールド抽出精度が低下する。特に画像間で類似したレイアウトがある場合、フィールドの取り違えが発生する。

**Fix:**
```python
# ❌ 画像PDF: 一括送信（精度低下）
pil_images = [PIL.Image.open(io.BytesIO(img)) for img in all_images]
response = model.generate_content(pil_images + [prompt])

# ✅ 画像PDF: ページ単位で個別送信
for i, image_bytes in enumerate(all_images):
    pil_image = PIL.Image.open(io.BytesIO(image_bytes))
    response = model.generate_content([pil_image, prompt])
    results.extend(parse_json_response(response.text))
```

**Note:** テキストPDFはページ区切りマーカー付きで一括送信しても精度が保たれる。画像PDFのみこの制約がある。

**Key lesson:** Gemini Vision APIで複数画像を扱う場合、精度が重要なフィールドがあるなら1画像ずつ個別送信する。一括送信はコスト削減になるが精度トレードオフがある。

### GeminiにJSON配列を要求しても単一オブジェクトが返る

**Symptom:** プロンプトで「JSON配列で返してください」と指示しても、1件の場合に `{...}` で返すことがある

**Fix:** JSON解析で配列優先 + 単一オブジェクトフォールバック:
```python
def parse_json_response(text: str) -> Optional[List[dict]]:
    # 配列形式を優先検出
    match = re.search(r'\[[\s\S]*\]', text)
    if match:
        data = json.loads(match.group())
        if isinstance(data, list):
            return data
    # フォールバック: 単一オブジェクト → 配列に変換
    match = re.search(r'\{[\s\S]*\}', text)
    if match:
        data = json.loads(match.group())
        if isinstance(data, dict):
            return [data]
    return None
```

**Key lesson:** LLM出力は指示通りの形式で返る保証がない。解析側で後方互換のフォールバックを必ず用意する。

## Docker Compose Volume Management

### `docker compose down` でボリューム参照が切れてDBが空になる

**Symptom:** `docker compose down` → `docker compose up -d` を実行したところ、DBが空になった（テーブルは存在するがデータなし）

**Cause:** `docker compose down` がコンテナとネットワークを削除する際、named volumeは削除されないが、新しい `up -d` で別のボリュームが作成されることがある。特にcompose projectの名前が変わった場合やvolumeの参照が不整合になった場合に発生。

**Fix:**
```bash
# ❌ docker compose down は使わない（ボリューム参照が切れるリスク）
docker compose down
docker compose up -d

# ✅ コンテナだけ削除して再作成（ボリュームは確実に保持）
docker rm -f container-name
docker compose up -d --build
```

**Recovery:** 旧ボリュームが残っている場合は `docker volume ls` で探し、データをコピー:
```bash
# 旧ボリュームからDBを復元
cp /var/lib/docker/volumes/old_volume/_data/database/app.db \
   /var/lib/docker/volumes/new_volume/_data/database/app.db
```

**Key lesson:** Dockerでの再起動は `docker rm -f` + `up -d` が最も安全。`down` はボリューム参照を壊すリスクがある。再起動後は必ずDBのデータ件数を確認する。

## Quick Checklist

Before debugging, check these common causes:

- [ ] Python: Using `pydantic[email]` not just `pydantic`?
- [ ] Python: bcrypt version pinned for Python 3.13+?
- [ ] Python: Using `get(key) or default` for Pydantic model_dump() dicts?
- [ ] React: Comparing objects by key, not reference?
- [ ] React: Zustand persist includes all needed fields?
- [ ] React: Component needs `key` to reset state on prop change?
- [ ] CSS: Flex child has `min-width: 0` for ellipsis?
- [ ] Ant Design: Table column has `width` for ellipsis?
- [ ] Ant Design: Upload file has `originFileObj` set?
- [ ] react-dnd: Using center-crossing logic for reorder?
- [ ] Architecture: Is one-way sync sufficient?
- [ ] New field: Added column definition to `allColumnDefinitions`?
- [ ] Fields: Mutually exclusive groups enforced (e.g., full_address vs detail fields)?
- [ ] State migration: All consumers updated when moving state scope?
- [ ] AI/OCR matching: Using fuzzy matching, not exact string comparison?
- [ ] Debug logging: Logger defined before first use in function?
- [ ] Cross-platform: CJK filenames use Unicode normalization (NFD/NFC)?
- [ ] API change: Frontend request format matches backend expectation?
- [ ] Import: Checking duplicates in BOTH file and DB?
- [ ] DB safety: Using ALTER/UPDATE instead of delete+recreate?
- [ ] Storage: Shared settings on server, not localStorage?
- [ ] VPS: Never modifying SSH config remotely?
- [ ] Voice AI: Japanese speech_end_offset ≥ 1.5s? (not 0.5s)
- [ ] Voice AI: System prompt under 1,000 chars for realtime API?
- [ ] Voice AI: temperature ≤ 0.4 for phone support?
- [ ] SIP: Cloud PBX supports standard SIP INVITE? (verified before dev)
- [ ] Docker Mac: NOT testing SIP/RTP on Docker Desktop for Mac?
- [ ] LLM API: 思考トークンを含めた実測コストを確認した?
- [ ] LLM API: モデル名を環境変数で管理（ハードコードしていない)?
- [ ] LLM API: サブスク内で完結する方法を先に検討した?
- [ ] Agent: Sub-agentに渡すJSONはpretty-print?
- [ ] Agent: 分類タスクにSonnet（Opusではなく）を使用?
- [ ] Express: `app.listen`にlocalhostバインドを指定?
- [ ] SQLite: WALモードのDB転送は`.dump`を使用?
- [ ] Cron: 複雑なパイプラインではなくAIボットに委任?
- [ ] Gemini Vision: 画像PDFは1ページずつ個別送信（一括送信しない）?
- [ ] Gemini: JSON解析に配列+単一オブジェクト両方のフォールバックがある?
- [ ] Docker: `docker compose down` ではなく `docker rm -f` + `up -d` で再起動?
- [ ] Docker: 再起動後にDBデータ件数を確認した?
