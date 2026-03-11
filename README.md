# hash

Hash table (associative array) implementation for FANUC Karel — two backends, fully generic value types.

## Overview

Karel has no native map or dictionary type. This module provides one via GPP-based generics: you pick a value type (any primitive or STRUCTURE), configure it in a `.klt` file, and expand a class template with `%class`. Two backends are available:

- **hasharray** — fixed-size `ARRAY[N]` storage, no resizing. Simple to use; fails silently when full.
- **hashpath** — dynamic `PATH`-based storage, auto-resizes at 80% occupancy. Use when entry count is unknown at compile time.

Both expose the same CRUD operations. The backends differ in how the table variable is passed to methods (BYNAME string pair vs. direct PATH reference).

---

## Files

| File | Purpose |
|------|---------|
| `lib/hasharray/hash.klc` | Static array backend — class template expanded by `%class` |
| `lib/hasharray/hashclass.klh` | Public method header: `put`, `get`, `delete`, `clear_table` |
| `lib/hasharray/hashclass.private.klh` | Private method header (hash functions, probing internals) |
| `lib/hasharray/hashclass_full.klh` | Public + private combined — test use only |
| `lib/hashpath/hashpath.klc` | Dynamic PATH backend — linear hashing class template |
| `lib/hashpath/hashpath.klh` | Public method header: `init`, `destructor`, `put`, `get`, `delete` |
| `lib/hashpath/hashpath.klt` | Defines `t_linear_hsh` PATH header + `hash_header_define` macro |
| `lib/hashpath/hashpath.private.klh` | Private method header |
| `types/hashstring.klt` | Config: STRING[16] values, 12-char keys |
| `types/hashint.klt` | Config: INTEGER values, 12-char keys |
| `types/hashreal.klt` | Config: REAL values, 12-char keys |
| `types/hashenv.klt` | Config: `T_ENV{typ, id}` for register name→id mapping, 24-char keys |
| `types/hashtypetest.klt` | Example custom struct config (tests only) |
| `test/test_hash.kl` | hasharray tests: STRING/REAL/struct types, collisions, hash functions |
| `test/test_pathhsh.kl` | hashpath tests: 10-entry expand cycle |
| `test/tst_hshenv.kl` | hashenv integration: name→register mapping + TP display |

---

## API Reference

### hasharray

All methods identify the table variable at runtime via `tblProg` (Karel program name) and `tblName` (variable name), resolved using Karel's `BYNAME` built-in. The table can live in any loaded program.

```
put(key       : STRING;
    value     : hval_type;
    tblProg   : STRING;
    tblName   : STRING) : BOOLEAN
```
Insert or overwrite `key`. Returns `FALSE` if `key = ''` or the table is full. Overwrites silently if key already exists.

```
get(key       : STRING;
    out_val   : hval_type;
    tblProg   : STRING;
    tblName   : STRING) : BOOLEAN
```
Look up `key`; copy value into `out_val`. Returns `FALSE` if not found or `key = ''`.

```
delete(key    : STRING;
       tblProg : STRING;
       tblName : STRING) : BOOLEAN
```
Remove `key` (sets its slot to `''`). Returns `FALSE` if not found.

```
clear_table(tblProg  : STRING;
            tblName  : STRING;
            clrData  : hval_type) : BOOLEAN
```
Reset all slots: `key = ''`, `val = clrData`. Always returns `TRUE`. **Call this before first use.**

### hashpath

Methods take the PATH variable directly (not by BYNAME). The PATH's header (`t_linear_hsh`) tracks occupancy and bit-depth.

```
init(tbl : PATH pathheader = t_linear_hsh, nodedata = hashname)
```
Required before first use. Clears all nodes, sets `n = 0`, `i = 0`.

```
destructor(tbl : PATH pathheader = t_linear_hsh, nodedata = hashname)
```
Delete all PATH nodes and uninitialize the header. Call when done.

```
put(key   : STRING;
    value : hval_type;
    tbl   : PATH ...) : BOOLEAN
```
Insert or overwrite. Automatically expands table (doubles buckets) if occupancy > 80%.

```
get(key     : STRING;
    out_val : hval_type;
    tbl     : PATH ...) : BOOLEAN
```
Look up `key`. Truncates key to `HSH_KEY_SIZE` before hashing. Returns `FALSE` if not found.

```
delete(key : STRING;
       tbl : PATH ...) : BOOLEAN
```
Remove key. Automatically shrinks table if occupancy would drop too low.

### Hash functions (hasharray private — exposed via `hashclass_full.klh` in tests)

```
djb2(str : STRING) : INTEGER    -- seed=5381, hash = hash*33 + ord(char)
jsStr(str : STRING) : INTEGER   -- Java 1.5: hash = ABS(31*hash + ord(char))
hMod(key : STRING; len : INTEGER) : INTEGER   -- jsStr(key) MOD len + 1
```

Both backends use `jsStr` as the primary hash. `djb2` is available in hasharray only. Neither is cryptographic — they're fast and good enough for Karel's small table sizes.

---

## Common Patterns

### 1. Simple string map (hasharray)

Use when you have a fixed set of keys and know an upper bound on count.

```
-- TYPE section
%include hashstring.klt          -- HSH_KEY_SIZE=12, hashname=h_string, hval_def=STRING[16]
t_hash(hashname, hval_def, hashstr)

VAR
  tbl    : ARRAY[50] OF hashname
  outVal : hval_def
  b      : BOOLEAN

-- Class expansion (once per translation unit)
%class hashstr('hash.klc', 'hashclass.klh', 'hashstring.klt')

BEGIN
  -- Always clear before first use
  b = hashstr__clear_table('myprog', 'tbl', '')

  b = hashstr__put('tool_type', 'gripper', 'myprog', 'tbl')
  b = hashstr__get('tool_type', outVal, 'myprog', 'tbl')
  -- outVal is now 'gripper'

  b = hashstr__delete('tool_type', 'myprog', 'tbl')
END myprog
```

### 2. Auto-expanding integer map (hashpath)

Use when entry count is variable or unknown at compile time.

```
-- TYPE section
%include define_type.m
%include hashpath.klt
hash_header_define(hshint)       -- defines t_linear_hsh FROM hshint

%include hashint.klt             -- HSH_KEY_SIZE=12, hashname=h_int, hval_def=INTEGER
t_hash(hashname, hval_def, hshint)

VAR
  htbl : PATH pathheader = t_linear_hsh, nodedata = hashname
  val  : hval_def
  b    : BOOLEAN

%class hshint('hashpath.klc', 'hashpath.klh', 'hashint.klt')

BEGIN
  hshint__init(htbl)

  b = hshint__put('counter_a', 10, htbl)
  b = hshint__put('counter_b', 20, htbl)
  -- table expands automatically if needed

  b = hshint__get('counter_a', val, htbl)   -- val = 10

  hshint__destructor(htbl)
END myprog
```

### 3. Custom struct values (hashenv pattern)

When the value is a user-defined STRUCTURE, define `hash_type_define` in the `.klt` and undef it after to prevent double-emission.

```
-- TYPE section
%include define_type.m
%include register_types.klt      -- DATA_REG, DATA_POSREG, etc.
%include hashenv.klt
hash_type_define(hashenv)        -- emit: TYPE T_ENV FROM hashenv = STRUCTURE ...
%undef hash_type_define
t_hash(hashname, hval_def, hashenv)

VAR
  tbl FROM env : ARRAY[20] OF hashname   -- table stored in program 'env'
  reg : hval_def
  b   : BOOLEAN

%class hashenv('hash.klc', 'hashclass.klh', 'hashenv.klt')

BEGIN
  reg = nullenv                  -- clear value (caller must define nullenv)
  b = hashenv__clear_table('env', 'tbl', reg)

  reg.typ = DATA_REG ; reg.id = 3
  b = hashenv__put('Laser_Power', reg, 'env', 'tbl')

  reg.typ = DATA_POSREG ; reg.id = 1
  b = hashenv__put('Home_Position', reg, 'env', 'tbl')

  -- Retrieve by name
  b = hashenv__get('Laser_Power', reg, 'env', 'tbl')
  -- reg.typ = DATA_REG, reg.id = 3
END myprog
```

### 4. Cross-program table (BYNAME)

hasharray's `tblProg`/`tblName` strings resolve the table variable at runtime. The table variable can live in a dedicated "storage" program, separate from the code that uses it.

```
-- Program 'regstore':
VAR tbl : ARRAY[30] OF hashname

-- Program 'myapp':
b = hashenv__put('speed_reg', reg, 'regstore', 'tbl')  -- writes into [regstore]tbl
b = hashenv__get('speed_reg', reg, 'regstore', 'tbl')  -- reads from [regstore]tbl
```

Both programs must be loaded on the controller simultaneously. This is how `hash-registers` separates register metadata storage from business logic.

### 5. Iterating all entries

There is no built-in iterator. Walk the array or PATH directly:

```
-- hasharray
VAR i : INTEGER
FOR i = 1 TO ARRAY_LEN(tbl) DO
  IF tbl[i].key <> '' THEN
    -- use tbl[i].key and tbl[i].val
  ENDIF
ENDFOR

-- hashpath
FOR i = 1 TO PATH_LEN(htbl) DO
  IF NOT UNINIT(htbl[i].key) THEN
    IF htbl[i].key <> '' THEN
      -- use htbl[i].key and htbl[i].val
    ENDIF
  ENDIF
ENDFOR
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Skipping `clear_table` before first use (hasharray) | Uninitialized key fields cause probing to produce incorrect results | Always call `clear_table` with an appropriate zero value before any `put` or `get` |
| Skipping `init` before first use (hashpath) | `PATH_LEN = 0`; all puts/gets return `FALSE` immediately | Always call `init` on a PATH-based hash before use |
| Missing `hash_header_define` for hashpath | Karel compiler error: `t_linear_hsh` not defined | Call `hash_header_define(class_name)` in the TYPE section before `t_hash(...)` |
| Key too long for `HSH_KEY_SIZE` | Key silently truncated; hasharray may miss the entry on lookup | Increase `HSH_KEY_SIZE` in your `.klt` to match the longest key you use |
| Using hasharray for an unbounded set of entries | `put` returns `FALSE` silently when full; data is silently dropped | Either size the array generously or switch to hashpath |
| Two `%class` expansions with the same `class_name` | Duplicate Karel routine declarations; compile error | Use a unique `class_name` for each hash instance |
| Not `%undef`-ing `hash_type_define` after custom type | TYPE emitted twice if `.klt` is included again in same unit | Add `%undef hash_type_define` immediately after calling `hash_type_define(...)` |

---

## Build Flow

`lib/hash` is a rossum umbrella package. It has no Karel source of its own — all code lives in `lib/hasharray` and `lib/hashpath`, which are listed as `includes` in `package.json`.

```sh
rossum .. -w -o -t    # generate build.ninja with tests
ninja                 # compile
kpush                 # deploy to controller
# Tests accessible at:
# http://<robot-ip>/KAREL/kunit?filenames=test_hash,test_pathhsh,tst_hshenv
```

Add `"hash"` to your module's `depends` in `package.json`. rossum will resolve both backends and make their headers available to ktransw.

See the top-level [Ka-Boost readme](../../readme.md) for full build setup instructions.
