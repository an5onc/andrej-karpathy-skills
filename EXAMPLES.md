# Examples

Real-world code examples demonstrating the seven principles. Each example shows what LLMs commonly do wrong and how to fix it.

---

## 1. Think Before Coding

### Example 1: Hidden Assumptions

**User Request:** "Add a feature to export user data"

**❌ What LLMs Do (Wrong Assumptions)**

```python
def export_users(format='json'):
    """Export all users to a file."""
    users = User.query.all()
    
    if format == 'json':
        with open('users.json', 'w') as f:
            json.dump([u.to_dict() for u in users], f)
    elif format == 'csv':
        with open('users.csv', 'w') as f:
            writer = csv.DictWriter(f, fieldnames=['id', 'email', 'name'])
            writer.writeheader()
            writer.writerows([u.to_dict() for u in users])
    
    return f"Exported {len(users)} users"
```

**Problems:**
- Assumed it should export ALL users (what about pagination? privacy?)
- Assumed file location without asking
- Assumed which fields to include
- Assumed CSV fieldnames without checking actual data structure

**✅ What Should Happen (Surface Assumptions)**

```
Before implementing, I need to clarify:

1. **Scope**: Export all users or filtered subset? (privacy implications)
2. **Format**: You mentioned "export" - did you mean:
   - Download file in browser?
   - Background job with email notification?
   - API endpoint returning data?
3. **Fields**: Which user fields? (some might be sensitive)
4. **Volume**: How many users typically? (affects approach)

Simplest approach: Add an API endpoint that returns paginated JSON.
Would need more info for file-based exports.

What's your preference?
```

### Example 2: Multiple Interpretations

**User Request:** "Make the search faster"

**❌ What LLMs Do (Pick Silently)**

```python
# Adds caching, database indexes, and async processing
# without asking which "faster" matters
@lru_cache(maxsize=1000)
async def search(query: str) -> List[Result]:
    # ... 200 lines of optimization code
```

**✅ What Should Happen**

```
"Make the search faster" could mean:

1. **Faster response time** (< 100ms instead of 500ms)
   - Add database indexes
   - Cache frequent queries
   - Estimated effort: 2 hours

2. **Handle more concurrent searches** (throughput)
   - Add async/await
   - Connection pooling
   - Estimated effort: 4 hours

3. **Faster perceived speed** (UX)
   - Show partial results immediately
   - Progressive loading
   - Estimated effort: 3 hours

Current search takes ~500ms for typical queries. 
Which aspect matters most for your use case?
```

---

## 2. Simplicity First

### Example 1: Over-abstraction

**User Request:** "Add a function to calculate discount"

**❌ What LLMs Do (Overengineered)**

```python
from abc import ABC, abstractmethod
from enum import Enum
from typing import Protocol, Union
from dataclasses import dataclass

class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, amount: float) -> float:
        pass

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percentage: float):
        self.percentage = percentage
    
    def calculate(self, amount: float) -> float:
        return amount * (self.percentage / 100)

class FixedDiscount(DiscountStrategy):
    def __init__(self, fixed_amount: float):
        self.fixed_amount = fixed_amount
    
    def calculate(self, amount: float) -> float:
        return min(self.fixed_amount, amount)

@dataclass
class DiscountConfig:
    strategy: DiscountStrategy
    min_purchase: float = 0.0
    max_discount: float = float('inf')

class DiscountCalculator:
    def __init__(self, config: DiscountConfig):
        self.config = config
    
    def apply_discount(self, amount: float) -> float:
        if amount < self.config.min_purchase:
            return 0
        discount = self.config.strategy.calculate(amount)
        return min(discount, self.config.max_discount)

# Usage requires 30+ lines of setup for a simple calculation
```

**✅ What Should Happen (Simple)**

```python
def calculate_discount(amount: float, percent: float) -> float:
    """Calculate discount amount. percent should be 0-100."""
    return amount * (percent / 100)

# Usage
discount = calculate_discount(100.0, 10.0)  # $10 off
```

**When to add complexity:** Only when you actually need multiple discount types. If that requirement comes later, refactor then.

### Example 2: Speculative Features

**User Request:** "Save user preferences to database"

**❌ What LLMs Do (Unnecessary Features)**

```python
class PreferenceManager:
    def __init__(self, db, cache=None, validator=None):
        self.db = db
        self.cache = cache or InMemoryCache()
        self.validator = validator or DefaultValidator()
    
    def save(self, user_id: int, prefs: dict, 
             merge: bool = True, 
             validate: bool = True,
             notify: bool = False) -> bool:
        """Save preferences with optional merging, validation, and notifications."""
        
        if validate:
            errors = self.validator.validate(prefs)
            if errors:
                raise ValidationError(errors)
        
        existing = self.get(user_id) if merge else {}
        
        if merge:
            existing.update(prefs)
            prefs = existing
        
        self.db.execute(
            "UPDATE users SET preferences = ? WHERE id = ?",
            (json.dumps(prefs), user_id)
        )
        
        if self.cache:
            self.cache.set(f"prefs:{user_id}", prefs)
        
        if notify:
            self.notify_preference_change(user_id, prefs)
        
        return True
    
    def notify_preference_change(self, user_id: int, prefs: dict):
        # Another 30 lines for a feature nobody asked for
        pass
```

**✅ What Should Happen (Just What Was Asked)**

```python
def save_preferences(db, user_id: int, preferences: dict):
    """Save user preferences to database."""
    db.execute(
        "UPDATE users SET preferences = ? WHERE id = ?",
        (json.dumps(preferences), user_id)
    )
```

**Add later if needed:** Caching (when performance matters), validation (when bad data appears), merging (when requirement emerges).

---

## 3. Surgical Changes

### Example 1: Drive-by Refactoring

**User Request:** "Fix the bug where empty emails crash the validator"

**❌ What LLMs Do (Too Much)**

```diff
  def validate_user(user_data):
-     # Check email format
-     if not user_data.get('email'):
+     """Validate user data."""
+     email = user_data.get('email', '').strip()
+     
+     # Validate email
+     if not email:
          raise ValueError("Email required")
-     
-     # Basic email validation
-     if '@' not in user_data['email']:
+     if '@' not in email or '.' not in email.split('@')[1]:
          raise ValueError("Invalid email")
      
-     # Check username
-     if not user_data.get('username'):
+     # Validate username
+     username = user_data.get('username', '').strip()
+     if not username:
          raise ValueError("Username required")
+     if len(username) < 3:
+         raise ValueError("Username too short")
+     if not username.isalnum():
+         raise ValueError("Username must be alphanumeric")
      
      return True
```

**Problems:**
- "Improved" email validation beyond the bug fix
- Added username validation nobody asked for
- Changed comments
- Added docstring

**✅ What Should Happen (Surgical)**

```diff
  def validate_user(user_data):
      # Check email format
-     if not user_data.get('email'):
+     email = user_data.get('email', '')
+     if not email or not email.strip():
          raise ValueError("Email required")
      
      # Basic email validation
-     if '@' not in user_data['email']:
+     if '@' not in email:
          raise ValueError("Invalid email")
      
      # Check username
      if not user_data.get('username'):
          raise ValueError("Username required")
      
      return True
```

**Only changed:** The specific lines that fix empty email handling.

### Example 2: Style Drift

**User Request:** "Add logging to the upload function"

**❌ What LLMs Do (Reformat Everything)**

```diff
- def upload_file(file_path, destination):
+ def upload_file(file_path: str, destination: str) -> bool:
+     """Upload file to destination with logging."""
+     logger.info(f"Uploading {file_path} to {destination}")
+     
      try:
-         with open(file_path, 'rb') as f:
-             data = f.read()
-         
-         response = requests.post(destination, files={'file': data})
-         
-         if response.status_code == 200:
-             return True
-         else:
-             return False
+         with open(file_path, "rb") as f:
+             data = f.read()
+         response = requests.post(destination, files={"file": data})
+         success = response.status_code == 200
+         
+         if success:
+             logger.info(f"Upload successful: {file_path}")
+         else:
+             logger.error(f"Upload failed: {file_path}, status={response.status_code}")
+         
+         return success
      except Exception as e:
-         print(f"Error: {e}")
+         logger.exception(f"Upload error for {file_path}: {e}")
          return False
```

**Problems:**
- Changed quote style ('' to "")
- Added type hints nobody asked for
- Added docstring
- Reformatted whitespace
- Changed boolean return logic

**✅ What Should Happen (Match Existing Style)**

```diff
+ import logging
+ 
+ logger = logging.getLogger(__name__)
+ 
  def upload_file(file_path, destination):
+     logger.info(f'Starting upload: {file_path}')
      try:
          with open(file_path, 'rb') as f:
              data = f.read()
          
          response = requests.post(destination, files={'file': data})
          
          if response.status_code == 200:
+             logger.info(f'Upload successful: {file_path}')
              return True
          else:
+             logger.error(f'Upload failed: {file_path}, status={response.status_code}')
              return False
      except Exception as e:
-         print(f"Error: {e}")
+         logger.exception(f'Upload error: {file_path}')
          return False
```

**Matched:** Single quotes, no type hints, existing boolean pattern, spacing style.

---

## 4. Goal-Driven Execution

### Example 1: Vague vs. Verifiable

**User Request:** "Fix the authentication system"

**❌ What LLMs Do (Vague Approach)**

```
I'll fix the authentication system by:
1. Reviewing the code
2. Identifying issues
3. Making improvements
4. Testing the changes

[Proceeds to make changes without clear success criteria]
```

**✅ What Should Happen (Verifiable Goals)**

```
To "fix authentication," I need to define success criteria. 
What specific issue are we solving?

If the issue is "users stay logged in after password change":

Plan:
1. Write test: Change password → verify old session invalidated
   Verify: Test fails (reproduces bug)

2. Implement: Invalidate sessions on password change
   Verify: Test passes

3. Check edge cases: Multiple active sessions, concurrent changes
   Verify: Additional tests pass

4. Verify no regression: Existing auth tests still pass
   Verify: Full test suite green

Current test coverage for auth: [shows coverage]
What's the specific auth issue you're seeing?
```

### Example 2: Multi-Step with Verification

**User Request:** "Add rate limiting to the API"

**❌ What LLMs Do (All at Once)**

```python
# Implements full rate limiting with Redis, multiple strategies,
# configuration system, and monitoring in one 300-line commit
# No clear verification steps
```

**✅ What Should Happen (Incremental with Verification)**

```
Plan for rate limiting:

1. Add basic in-memory rate limiting (single endpoint)
   Verify: 
   - Test: 100 requests → first 10 succeed, rest get 429
   - Manual: curl endpoint 11 times, see rate limit error

2. Extract to middleware (apply to all endpoints)
   Verify:
   - Test: Rate limits apply to /users and /posts
   - Existing endpoint tests still pass

3. Add Redis backend (for multi-server)
   Verify:
   - Test: Rate limit persists across app restarts
   - Test: Two app instances share rate limit counter

4. Add configuration (rates per endpoint)
   Verify:
   - Test: /search allows 10/min, /users allows 100/min
   - Config file parsed correctly

Each step is independently verifiable and deployable.
Start with step 1?
```

### Example 3: Test-First Verification

**User Request:** "The sorting breaks when there are duplicate scores"

**❌ What LLMs Do (Fix Without Reproducing)**

```python
# Immediately changes sort logic without confirming the bug
def sort_scores(scores):
    return sorted(scores, key=lambda x: (-x['score'], x['name']))
```

**✅ What Should Happen (Reproduce First)**

```python
# 1. First, write a test that reproduces the issue
def test_sort_with_duplicate_scores():
    """Test sorting when multiple items have same score."""
    scores = [
        {'name': 'Alice', 'score': 100},
        {'name': 'Bob', 'score': 100},
        {'name': 'Charlie', 'score': 90},
    ]
    
    result = sort_scores(scores)
    
    # The bug: order is non-deterministic for duplicates
    # Run this test multiple times, it should be consistent
    assert result[0]['score'] == 100
    assert result[1]['score'] == 100
    assert result[2]['score'] == 90

# Verify: Run test 10 times → fails with inconsistent ordering

# 2. Now fix with stable sort
def sort_scores(scores):
    """Sort by score descending, then name ascending for ties."""
    return sorted(scores, key=lambda x: (-x['score'], x['name']))

# Verify: Test passes consistently
```

---

## 5. Read Before You Write

### Example 1: Assumed File Structure

**User Request:** "Add a new endpoint to the user controller"

**❌ What LLMs Do (Assume Structure)**

```python
# Without reading the file, assumes a class-based controller
class UserController:
    def get_user(self, user_id: int):
        return User.query.get(user_id)
    
    # Adds new method assuming class structure
    def delete_user(self, user_id: int):
        user = User.query.get(user_id)
        db.session.delete(user)
        db.session.commit()
```

**Problems:**
- Didn't read the actual file first
- Assumed class-based structure (might be function-based)
- Assumed SQLAlchemy patterns (might use different ORM)
- Assumed db.session exists

**✅ What Should Happen (Read First)**

```
Before adding the endpoint, let me read the existing controller to understand:
1. The file structure (class-based vs function-based)
2. The ORM/database patterns used
3. Existing error handling conventions
4. Authentication/authorization patterns

[Reads file first, then matches existing patterns exactly]
```

### Example 2: Hallucinated API

**User Request:** "Use the caching library to cache this response"

**❌ What LLMs Do (Guess API)**

```python
from cache_lib import Cache, TTL

cache = Cache(backend="redis", ttl=TTL.MINUTES_5)

@cache.memoize(expire=300)
def get_expensive_data():
    return compute_expensive_thing()
```

**Problems:**
- `cache_lib` might not exist or have different API
- `TTL` enum might not exist
- `memoize` decorator syntax might be wrong
- `backend` parameter might not be valid

**✅ What Should Happen (Check First)**

```
I see you want to use a caching library. Before implementing:

1. What's the exact package name? (I don't want to guess the import)
2. Is there existing caching code in this project I should follow?

Or if you can point me to existing usage in the codebase, 
I'll match that pattern exactly.
```

---

## 6. Prefer Reversible Actions

### Example 1: Destructive Cleanup

**User Request:** "Clean up the unused files in this directory"

**❌ What LLMs Do (Delete Immediately)**

```bash
rm -rf old_configs/
rm -rf deprecated/
rm unused_helper.py
rm legacy_utils.py
```

**Problems:**
- Deleted files permanently without confirmation
- "Unused" was an assumption, not verified
- No backup or soft-delete option
- Hard to recover if wrong

**✅ What Should Happen (Reversible First)**

```
I found these potentially unused files:
- old_configs/ (last modified 2 years ago)
- deprecated/ (no imports found)
- unused_helper.py (no references in codebase)
- legacy_utils.py (imported only in test files)

Before deleting, I recommend:
1. Move to `_archive/` directory instead of deleting
2. Or rename with `.bak` extension
3. Commit the removal separately so it's easy to revert

Which approach would you prefer? Or should I verify 
usage more thoroughly first?
```

### Example 2: Database Migration

**User Request:** "Drop the unused columns from the users table"

**❌ What LLMs Do (Destructive Migration)**

```sql
ALTER TABLE users DROP COLUMN legacy_status;
ALTER TABLE users DROP COLUMN old_preference_json;
ALTER TABLE users DROP COLUMN deprecated_flag;
```

**Problems:**
- Data loss is permanent
- No verification columns are actually unused
- No backup of data
- Can't rollback after execution

**✅ What Should Happen (Safe Approach)**

```
Before dropping columns, I recommend a safer approach:

1. First, verify they're truly unused:
   - Search codebase for column references
   - Check for any ORM models using them
   - Review recent queries in logs

2. If dropping, use a reversible migration:
   ```sql
   -- Migration: rename instead of drop
   ALTER TABLE users RENAME COLUMN legacy_status TO _deprecated_legacy_status;
   
   -- Keep for 30 days, then run cleanup migration
   ```

3. Or backup first:
   ```sql
   CREATE TABLE users_column_backup AS 
   SELECT id, legacy_status, old_preference_json FROM users;
   ```

Which approach would you prefer?
```

---

## 7. Honest Uncertainty

### Example 1: Library Version Uncertainty

**User Request:** "How do I use the new streaming API in langchain?"

**❌ What LLMs Do (Fabricate Details)**

```python
from langchain.streaming import StreamHandler, StreamConfig

config = StreamConfig(
    buffer_size=1024,
    timeout_ms=5000,
    on_chunk=lambda x: print(x)
)

handler = StreamHandler(config)
chain.stream(handler)
```

**Problems:**
- API might have changed since training
- Class names might be wrong
- Parameter names might be fabricated
- Confident tone hides uncertainty

**✅ What Should Happen (Flag Uncertainty)**

```
I can outline the general approach for streaming in LangChain, 
but I should flag: my knowledge might be outdated as this API 
changes frequently.

The pattern I'm aware of is:
```python
# This was accurate as of [date], verify current docs
for chunk in chain.stream(input):
    print(chunk)
```

I'd recommend checking:
1. Current LangChain docs for your version
2. The `stream()` method signature in your installed version
3. Any existing streaming code in your project

What version of LangChain are you using?
```

### Example 2: Ambiguous Bug Report

**User Request:** "Why is the API slow?"

**❌ What LLMs Do (Confident Diagnosis)**

```
The API is slow because:
1. You're not using connection pooling
2. The database queries aren't indexed
3. The JSON serialization is inefficient

Here's the fix:
[Proceeds to implement changes without verification]
```

**Problems:**
- No evidence for any of these claims
- Didn't measure or profile
- Implementing fixes for guessed problems
- Confident tone prevents user from questioning

**✅ What Should Happen (Honest Assessment)**

```
I don't have enough information to diagnose this yet. 
"Slow" could mean many things:

Questions to narrow down:
1. What's the actual latency? (500ms vs 30s are different problems)
2. Which endpoints are slow? (All of them, or specific ones?)
3. When did it start? (Recent change, or always been this way?)
4. What does profiling/tracing show?

Common causes I'd investigate, in order:
- Database queries (most common) - check slow query log
- N+1 queries - add logging to count queries per request
- External API calls - add timing around integrations
- Memory/CPU constraints - check server metrics

Would you like help setting up profiling, or do you have 
metrics I can look at?
```

---

## Anti-Patterns Summary

| Principle | Anti-Pattern | Fix |
|-----------|-------------|-----|
| Think Before Coding | Silently assumes file format, fields, scope | List assumptions explicitly, ask for clarification |
| Simplicity First | Strategy pattern for single discount calculation | One function until complexity is actually needed |
| Surgical Changes | Reformats quotes, adds type hints while fixing bug | Only change lines that fix the reported issue |
| Goal-Driven | "I'll review and improve the code" | "Write test for bug X → make it pass → verify no regressions" |
| Read Before You Write | Assumes class structure, hallucinates API signatures | Read file first, ask about unfamiliar libraries |
| Prefer Reversible Actions | `rm -rf` without confirmation, `DROP COLUMN` without backup | Rename/archive first, ask before destructive ops |
| Honest Uncertainty | Fabricates library API with confidence | Flag outdated knowledge, ask clarifying questions |

## Key Insight

The "overcomplicated" examples aren't obviously wrong—they follow design patterns and best practices. The problem is **timing**: they add complexity before it's needed, which:

- Makes code harder to understand
- Introduces more bugs
- Takes longer to implement
- Harder to test

The "simple" versions are:
- Easier to understand
- Faster to implement
- Easier to test
- Can be refactored later when complexity is actually needed

**Good code is code that solves today's problem simply, not tomorrow's problem prematurely.**
