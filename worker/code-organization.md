# Code Organization

**CRITICAL:** Every script follows INFRASTRUCTURE → ORCHESTRATOR → FUNCTIONS

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

A new module is warranted when ANY of these are exceeded:

1. **Lines of Code:** > 400 LOC with distinct functional groups
2. **Function Count:** > 15 functions (likely multiple responsibilities)
3. **Single Responsibility:** Module handles multiple unrelated concerns

**Additional Indicators:**
- Function > 50 LOC → Extract helper functions (not new module)
- > 5 cross-module imports → Review dependencies, may indicate over-coupling

## Import Convention

- Prefer absolute imports (`from src.module.submodule import name`)
- Document cross-module imports with comment: `# From src/module.py: what it does`
- No relative imports in projects that use absolute import style consistently

## Inter-Module Dependencies

When module A needs functionality from module B:
- Module A imports specific functions from module B
- Module A's orchestrator calls imported functions
- If a function is only used by another module, it belongs in THAT module
