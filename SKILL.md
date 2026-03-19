---
name: common-pitfalls
description: Use when encountering Python dependency errors, React state issues, CSS ellipsis problems, Ant Design Upload/Table issues, Zustand persistence problems, react-dnd drag vibration, data sync design decisions, AI/OCR text matching failures, cross-platform Unicode (NFD/NFC) issues, Pydantic model_dump pitfalls, FastAPI request format mismatches, database safety concerns, localStorage limitations, import data uniqueness, or VPS/remote server operations
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
