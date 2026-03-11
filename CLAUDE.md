# hash — CLAUDE.md

## Purpose

Layer 3 of Ka-Boost. Provides the only associative array (hash table / dictionary) available in Karel — which has no native map type. Two independent backends share an identical public API surface: **hasharray** (static, fixed-size ARRAY) and **hashpath** (dynamic, auto-resizing PATH). Both are generic: instantiated via GPP `.klt` config + `%class` expansion; the value type is arbitrary (STRING, INTEGER, REAL, or any custom STRUCTURE).

Used directly by: `hash-registers` (register name → id mapping), `graph/kd_tree` (node → index lookup), `TPE` (open program dictionary), `xmlib` (tag → parent mapping).

---

## Repository Layout

```
lib/hash/
├── package.json               -- rossum manifest; includes lib/hasharray, lib/hashpath, types
├── README.md                  -- original pre-Ka-Boost docs (stale API, ignore for current usage)
├── lib/
│   ├── hasharray/
│   │   ├── hash.klc           -- static array backend class template
│   │   ├── hashclass.klh      -- public method declarations (put/get/delete/clear_table)
│   │   ├── hashclass.private.klh  -- private method declarations (djb2/jsStr/hMod/hGetIndex/...)
│   │   └── hashclass_full.klh -- public + private combined (used in tests only)
│   └── hashpath/
│       ├── hashpath.klc       -- dynamic PATH backend class template (linear hashing)
│       ├── hashpath.klh       -- public method declarations (init/destructor/put/get/delete)
│       ├── hashpath.klt       -- t_linear_hsh PATH header type + hash_header_define macro
│       └── hashpath.private.klh -- private method declarations
└── types/
    ├── hashstring.klt         -- STRING[16] value type, 12-char keys
    ├── hashint.klt            -- INTEGER value type, 12-char keys
    ├── hashreal.klt           -- REAL value type, 12-char keys
    ├── hashenv.klt            -- T_ENV{typ:SHORT, id:SHORT} for register mapping, 24-char keys
    └── hashtypetest.klt       -- TYPETEST struct (used only in tests)
test/
    ├── test_hash.kl           -- hasharray tests: STRING/REAL/custom struct, collisions, hash functions
    ├── test_pathhsh.kl        -- hashpath tests: expand on 10 INTEGER inserts
    └── tst_hshenv.kl          -- hashenv integration test: name→register mapping + display
```

---

## Core Macros

### `t_hash(name, value_type, class_name)` — from `define_type.m`

Generates the hash entry STRUCTURE. Must be called in the TYPE section before instantiating the class.

```
t_hash(hashname, hval_def, myclass)
-- Expands to:
TYPE
  hashname FROM myclass = STRUCTURE
    key : STRING[HSH_KEY_SIZE]
    val : hval_def
  ENDSTRUCTURE
```

`HSH_KEY_SIZE` defaults to 12; set it in your `.klt` config before calling `t_hash`.

### `hash_header_define(parent)` — from `hashpath.klt`

Generates the PATH header STRUCTURE for hashpath. Must be called once per class instantiation (with the same `parent` name used in `t_hash`).

```
hash_header_define(hshint)
-- Expands to:
TYPE
  t_linear_hsh FROM hshint = STRUCTURE
    n          : INTEGER  -- occupied bucket count
    i          : INTEGER  -- bit depth; table size = 2^i
    occupancy  : REAL
  ENDSTRUCTURE
```

### `hash_type_define(class_name)` — user-defined in custom `.klt` configs

When the value type is a custom STRUCTURE (not a primitive), define this macro in your `.klt` before `%class` expansion. The `.klc` template calls `hash_type_define(class_name)` if defined, so the TYPE declaration appears inside the correct Karel program scope.

```
-- Example from hashenv.klt:
%define hash_type_define(obj_name)
TYPE
  T_ENV FROM obj_name = STRUCTURE
    typ : SHORT
    id  : SHORT
  ENDSTRUCTURE
```

---

## Backend Comparison

| | **hasharray** | **hashpath** |
|---|---|---|
| Storage | Fixed `ARRAY[N]` | Dynamic `PATH` (Karel linked list) |
| VAR declaration | `tbl : ARRAY[N] OF hashname` | `tbl : PATH pathheader = t_linear_hsh, nodedata = hashname` |
| Resize | No — `put` returns FALSE when full | Auto-expands at >80% occupancy; auto-shrinks on delete |
| `init` required | No (just call `clear_table` first) | Yes — `init` clears and sets n=0, i=0 |
| `tbl` in API | Indirect: `tblProg : STRING; tblName : STRING` (BYNAME) | Direct: `tbl : PATH ...` passed by reference |
| Hash function | jsStr (Java 1.5) — djb2 also available | jsStr only |
| Collision | Linear probing + wrap; table-full = FALSE | Linear hashing — probes + intelligent bucket-winner resolution |
| Use case | Fixed memory, embedded loops | Variable-size tables, register maps, open-ended lookup |

---

## Full API Reference

### hasharray — `lib/hasharray/hash.klc` + `hashclass.klh`

All methods take `tblProg : STRING; tblName : STRING` — these are the program name and variable name of the `ARRAY[N] OF hashname` table, accessed via Karel's `BYNAME` built-in. The table variable can live in any loaded Karel program.

```
put(key : STRING; value : hval_type; tblProg : STRING; tblName : STRING) : BOOLEAN
  -- Insert or overwrite key. Returns FALSE if key='' or table is full.

get(key : STRING; out_val : hval_type; tblProg : STRING; tblName : STRING) : BOOLEAN
  -- Lookup key; copies value into out_val. Returns FALSE if key='' or not found.

delete(key : STRING; tblProg : STRING; tblName : STRING) : BOOLEAN
  -- Remove key. Clears key field to ''. Returns FALSE if key='' or not found.

clear_table(tblProg : STRING; tblName : STRING; clrData : hval_type) : BOOLEAN
  -- Zero all entries: set key='' and val=clrData for entire array. Always returns TRUE.
```

**Private (available via `hashclass_full.klh` in tests only):**

```
djb2(str : STRING) : INTEGER          -- DJB2 hash: seed=5381, hash = hash*33 + char
jsStr(str : STRING) : INTEGER         -- Java 1.5: hash = ABS(31*hash + ord(char))
hMod(key : STRING; len : INTEGER) : INTEGER   -- jsStr(key) MOD len + 1
hGetIndex(key : STRING; get : BOOLEAN; tbl) : INTEGER
  -- Linear probing. get=TRUE: match key=key. get=FALSE: match key=key OR key=''.
  -- Returns 0 if no slot found.
```

### hashpath — `lib/hashpath/hashpath.klc` + `hashpath.klh`

Methods take the PATH directly (not BYNAME). Must call `init` before first use; call `destructor` to free nodes.

```
init(tbl : PATH pathheader = t_linear_hsh, nodedata = hashname)
  -- Clear all nodes, set n=0, i=0 (table size 2^0 = 1 bucket initially).

destructor(tbl : PATH pathheader = t_linear_hsh, nodedata = hashname)
  -- Delete all PATH nodes, uninitialize header.

put(key : STRING; value : hval_type; tbl : PATH ...) : BOOLEAN
  -- Insert or overwrite. Auto-expands if occupancy > 0.8. Returns FALSE only on collision failure.

get(key : STRING; out_val : hval_type; tbl : PATH ...) : BOOLEAN
  -- Lookup. Truncates key to HSH_KEY_SIZE before hash. Returns FALSE if not found.

delete(key : STRING; tbl : PATH ...) : BOOLEAN
  -- Remove key. Decrements n. Auto-shrinks if occupancy would drop below threshold.
```

**Key private routines (hashpath.private.klh):**
```
expandOccupancy(tbl) : BOOLEAN    -- tbl.occupancy > MAX_OCCUPANCY (0.8)
shrinkOccupancy(tbl) : BOOLEAN    -- tbl.n / 2^(tbl.i-1) < MAX_OCCUPANCY
expand(tbl)                       -- APPEND_NODE × (2^(i+1) - current), tbl.i++, rehash(TRUE)
shrink(tbl)                       -- tbl.i--, rehash(FALSE), DELETE_NODE excess
rehash(grow_shrink, tbl)          -- reassign entries whose hMod changes after resize
hPutIndex(key, tbl) : INTEGER     -- collision resolution: predict which key should win bucket
```

---

## Built-in Type Configs (`types/`)

| File | `hashname` | `hval_def` | `hval_type` | `HSH_KEY_SIZE` | Notes |
|------|-----------|-----------|------------|---------------|-------|
| `hashstring.klt` | `h_string` | `STRING[16]` | `STRING` | 12 | Basic string map |
| `hashint.klt` | `h_int` | `INTEGER` | `INTEGER` | 12 | Integer values |
| `hashreal.klt` | `h_real` | `REAL` | `REAL` | 12 | Real values |
| `hashenv.klt` | `h_env` | `T_ENV` | `T_ENV` | 24 | register metadata; requires `hash_type_define` |
| `hashtypetest.klt` | `h_typ` | `TYPETEST` | `TYPETEST` | — | test only |

`T_ENV` (hashenv): `typ : SHORT` (register type code), `id : SHORT` (register number). Used by `hash-registers`.

---

## Core Patterns

### Pattern 1 — hasharray with a primitive value type

Simplest case. Table lives in the calling program, accessed by name.

```
-- TYPE section (before %class)
%include hashstring.klt          -- sets HSH_KEY_SIZE=12, hashname=h_string, hval_def=STRING[16]
t_hash(hashname, hval_def, hashstr)

VAR
  tbl : ARRAY[20] OF hashname
  outVal : hval_def
  b : BOOLEAN

-- Class instantiation (once per program that uses it)
%class hashstr('hash.klc', 'hashclass.klh', 'hashstring.klt')

-- Usage
b = hashstr__clear_table('myprog', 'tbl', '')
b = hashstr__put('tool_name', 'gripper', 'myprog', 'tbl')
b = hashstr__get('tool_name', outVal, 'myprog', 'tbl')   -- outVal = 'gripper'
b = hashstr__delete('tool_name', 'myprog', 'tbl')
```

### Pattern 2 — hashpath with auto-expansion (INTEGER values)

Use when the number of entries is unknown at compile time.

```
-- TYPE section
%include define_type.m
%include hashpath.klt
hash_header_define(hshint)           -- emit t_linear_hsh FROM hshint

%include hashint.klt
t_hash(hashname, hval_def, hshint)

VAR
  htbl : PATH pathheader = t_linear_hsh, nodedata = hashname
  val  : hval_def
  b    : BOOLEAN

%class hshint('hashpath.klc', 'hashpath.klh', 'hashint.klt')

-- Usage (always init first; destructor at end)
hshint__init(htbl)
b = hshint__put('alpha', 10, htbl)
b = hshint__put('beta',  20, htbl)
b = hshint__get('alpha', val, htbl)   -- val = 10
b = hshint__delete('beta', htbl)
hshint__destructor(htbl)
```

### Pattern 3 — hashenv with custom struct value

Required when value type is a user-defined STRUCTURE. `hash_type_define` must be defined and then undefined to avoid double-definition if included elsewhere.

```
-- TYPE section
%include define_type.m
%include register_types.klt          -- defines DATA_REG, DATA_POSREG, etc.
%include hashenv.klt
hash_type_define(hashenv)            -- emit TYPE T_ENV FROM hashenv = STRUCTURE ...
%undef hash_type_define              -- prevent re-emission if hashenv is included again
t_hash(hashname, hval_def, hashenv)

VAR
  tbl FROM env : ARRAY[20] OF hashname   -- table in separate 'env' program
  reg : hval_def
  b   : BOOLEAN

%class hashenv('hash.klc', 'hashclass.klh', 'hashenv.klt')

-- Usage
b = hashenv__clear_table('env', 'tbl', nullenv)
reg.typ = DATA_REG ; reg.id = 1
b = hashenv__put('Laser_Power', reg, 'env', 'tbl')
b = hashenv__get('Laser_Power', reg, 'env', 'tbl')
```

### Pattern 4 — Table in a separate program (BYNAME cross-program access)

hasharray's `tblProg`/`tblName` parameters resolve the variable at runtime using Karel's `BYNAME`. The table variable can live in any loaded program.

```
-- In program 'env':
VAR tbl : ARRAY[20] OF hashname

-- In program 'myapp':
b = hashenv__put('key', reg, 'env', 'tbl')   -- writes into [env]tbl
b = hashenv__get('key', reg, 'env', 'tbl')   -- reads from [env]tbl
```

This is how `hash-registers` stores register metadata in a dedicated program and how `TPE` stores its program-open dictionary in `tpelib`.

### Pattern 5 — Table iteration (raw loop over array)

There is no iterator method. To enumerate all entries, walk the array directly:

```
VAR i : INTEGER
FOR i = 1 TO ARRAY_LEN(tbl) DO
  IF tbl[i].key <> '' THEN
    -- process tbl[i].key and tbl[i].val
    registers__set_comment(tbl[i].val.typ, tbl[i].val.id, tbl[i].key)
  ENDIF
ENDFOR
```

hashpath: same but use `PATH_LEN(tbl)` and bracket-index the PATH nodes directly.

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Calling `put`/`get` before `clear_table` (hasharray) | Stale `UNINIT` keys cause probing to behave incorrectly; `hGetIndex` forcibly sets `''` but uninitialized entries in fresh arrays are unpredictable | Always call `clear_table` with an appropriate zero-value before first use |
| Calling hashpath `put`/`get` before `init` | PATH_LEN = 0; hGetIndex loops 0 times and returns 0 immediately; all puts return FALSE | Always call `init` on a PATH-based hash before use |
| Forgetting `hash_header_define` for hashpath | Karel compiler error: `t_linear_hsh` not defined | Call `hash_header_define(class_name)` before `t_hash(...)` in TYPE section |
| Key longer than `HSH_KEY_SIZE` | Key silently truncated (hashpath truncates in `get`); hasharray probes with full key vs stored truncated key → miss | Set `HSH_KEY_SIZE` in `.klt` before instantiation; use `hashenv.klt` (24) for longer names |
| Two `%class` expansions with same `class_name` | Duplicate routine declarations; Karel compile error | Use a unique `class_name` per instantiation (e.g. `hashenv` vs `hashstr`) |
| Using hasharray and expecting resize | `put` returns FALSE silently when table is full; data is lost | Either size the array large enough upfront or switch to hashpath backend |
| Not calling `%undef hash_type_define` after custom type | If the `.klt` is re-included in the same translation unit, the TYPE is emitted twice | Add `%undef hash_type_define` immediately after `hash_type_define(class_name)` call |
| Passing `tblProg`/`tblName` for the wrong program | BYNAME resolves to wrong memory; silent corrupt read/write | Verify the program hosting the table variable is loaded and the names match exactly |

---

## Dependencies

**hash depends on:**
- `errors` — `CHK_STAT`, `karelError`, `ER_ABORT`, `INVALID_INDEX`
- `math` — `math__pow` (hashpath: computing 2^i for table size)
- `Strings` — `ord` (character ordinal in jsStr hash function)
- `ktransw-macros` — `t_hash`, `define_type.m`, `namespace.m`, `header_guard.m`

**Modules that depend on hash (direct consumers):**
- `hash-registers` — hashenv backend for named FANUC register access
- `graph` — hashpath for k-d tree node-to-index mapping during tree construction
- `TPE` — hasharray (via custom `TPEPROGRAMS` type) for tracking open TP program handles
- `xmlib` — hasharray (via custom `T_XML_GRAPH` type) for XML tag→parent graph
- `iterator` — optionally (some iterator configs reference hash for TP bridging)

---

## Build / Integration Notes

- The `lib/hash/package.json` has no `.kl` source files itself — it is a pure umbrella package. All code lives in `lib/hasharray/` and `lib/hashpath/`.
- rossum resolves `lib/hasharray` and `lib/hashpath` as `includes` (not `depends`), meaning they are header-included rather than linked as separate `.pc` files. The `.klc` is expanded inline at compile time via `%class`.
- To use hash in a consumer module, add `"hash"` to that module's `depends` in `package.json`. rossum will make `hasharray` and `hashpath` available.
- Test programs: `test_hash.kl` uses `hashclass_full.klh` (exposes private routines) — never use `hashclass_full.klh` in production code.
- `DEBUG_PUT` and `DEBUG_GET` flags in `hashpath.klc` (default FALSE) can be flipped to TRUE to emit TPDISPLAY trace during development.
