# Code Organization

Every script follows INFRASTRUCTURE → ORCHESTRATOR → FUNCTIONS

## Section Definitions

**INFRASTRUCTURE:**
- Imports and constants
- NO functions, NO logic

**Constants placement:**
- Module-specific constants (used by ONE module only): define in that module's INFRASTRUCTURE
- Shared constants (used by 2+ modules): define in the project's config module (e.g. `config.py`, `constants.py`), import from there
- Same constant defined in 2+ files is PROHIBITED — always centralize

**ORCHESTRATOR:**
- ONE function (named: tool_name_workflow for MCP, freely chosen for scripts)
- Calls only (function composition)
- ZERO functional logic (no calculations, transformations, business rules)
- Meta-logic allowed: conditional workflow execution, parameter routing

**FUNCTIONS:**
- Ordered by call sequence
- One responsibility each
- Can call other functions internally
- All functions must be called by the module's orchestrator (directly or indirectly)

**Exception:** Utility modules (constants-only, client.py, helpers) may omit ORCHESTRATOR and/or FUNCTIONS sections.

## Comment Rules

**Three types of allowed comments only:**

### 1. Section Markers
```python
# INFRASTRUCTURE
# ORCHESTRATOR
# FUNCTIONS
```

### 2. Function Header Comments
```python
# Load validated customer data from CSV
def load_customer_data(file_path: str) -> pd.DataFrame:
    return pd.read_csv(file_path)
```

One line describing WHAT. Never HOW. Placed directly above function definition.

### 3. Cross-Module Import Comments
```python
# From data_loader.py: Load and validate CSV
from .data_loader import load_validated_data
```

Format: `# From <module>.py: <what it does>`

### PROHIBITED: Inline Comments
```python
# WRONG
def process_data(df):
    df = df.dropna()  # Remove missing values  <- PROHIBITED
    return df

# RIGHT
def process_data(df):
    df = df.dropna()
    return df
```

## Module Complexity Thresholds

### Hard Ceiling — 400 LOC

No source file exceeds 400 LOC. Hard limit, not a heuristic. When a file approaches the ceiling, split by concern — extract a structural responsibility into its own module. Cosmetic shrinking (trimming blank lines, condensing comments) is not a split and does not count.

This complements the split-candidate heuristic below: 200 LOC with functional groups = consider splitting; 400 LOC = MUST split, regardless of internal structure.

### Split Candidate

A new module is warranted when ANY of these are exceeded:

1. **Lines of Code:** > 200 LOC with distinct functional groups
2. **Function Count:** > 15 functions (likely multiple responsibilities)
3. **Single Responsibility:** Module handles multiple unrelated concerns

**Additional Indicators:**
- Function > 50 LOC → Extract helper functions (not new module)
- > 5 cross-module imports → Review dependencies, may indicate over-coupling

### LOC Threshold Interpretation

LOC thresholds are HEURISTICS for flagging split candidates, not magic numbers to hit. When a file sits near the threshold:

1. **First question:** Does a concern-based extraction improve readability? (e.g. browser handling vs reporting vs CLI — three clear concerns that belong in three separate files.) If yes → split by concern.
2. **Second question:** If no structural concern emerges and the file sits just above the threshold (e.g. 202 LOC against a 200-rule) → **LEAVE IT**. Do NOT trim blank lines, condense comments, or cosmetic-split to hit the number.
3. **Never** propose a refactor whose only justification is "hit the LOC target". A refactor without a readability/concern argument is noise, not improvement.

## Import Convention

- Prefer absolute imports (`from src.module.submodule import name`)
- Document cross-module imports with comment: `# From src/module.py: what it does`
- No relative imports in projects that use absolute import style consistently

## Inter-Module Dependencies

When module A needs functionality from module B:
- Module A imports specific functions from module B
- Module A's orchestrator calls imported functions
- If a function is only used by another module, it belongs in THAT module
