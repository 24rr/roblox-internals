# VM Shuffles

Guide on locating and understanding VM_SHUFFLE macros in Roblox's Luau VM. Covers shuffle definitions, how to locate elements like `luaT_typenames` and `luaT_eventname`, and comparing original positions with shuffled ones. Includes examples for `VM_SHUFFLE3`, `VM_SHUFFLE5`, `VM_SHUFFLE7`, and others, along with steps to analyze offsets in structures like `lua_State` and `Proto`. Written by **@trypt.amine**.

## Definitions

```c
#define VM_SHUFFLE_COMMA ,
#define VM_SHUFFLE_SEMICOLON ;
#define VM_SHUFFLE3(s, a1, a2, a3) a1 s a2 s a3
#define VM_SHUFFLE4(s, a1, a2, a3, a4) a2 s a4 s a3 s a1
#define VM_SHUFFLE5(s, a1, a2, a3, a4, a5) a3 s a4 s a2 s a5 s a1
#define VM_SHUFFLE6(s, a1, a2, a3, a4, a5, a6) a1 s a3 s a2 s a6 s a5 s a4
#define VM_SHUFFLE7(s, a1, a2, a3, a4, a5, a6, a7) a7 s a5 s a6 s a3 s a1 s a2 s a4
#define VM_SHUFFLE8(s, a1, a2, a3, a4, a5, a6, a7, a8) a8 s a3 s a6 s a4 s a5 s a1 s a7 s a2
#define VM_SHUFFLE9(s, a1, a2, a3, a4, a5, a6, a7, a8, a9) a4 s a7 s a8 s a6 s a1 s a9 s a5 s a3 s a2
```

### Locating them

#### VM_SHUFFLE3 & VM_SHUFFLE5

```c
const char* const luaT_typenames[] = {
    "nil",
    "boolean",
    LUAVM_SHUFFLE3(VM_SHUFFLE_COMMA,
        "userdata",
        "number",
        "vector"
    ),
    "string",
    VM_SHUFFLE5(VM_SHUFFLE_COMMA,
        "table",
        "function",
        "userdata",
        "thread",
        "buffer"
    )
};
```

1. Locate `luaT_typenames` by searching for `thread`.
2. Compare the Original one with the one you found.

| Original | Position | Shuffle | Shuffle Position |
|----------|----------|---------|------------------|
| nil | | nil | |
| boolean | | boolean | |
| userdata | a1 | userdata | a1 |
| number | a2 | number | a2 |
| vector | a3 | vector | a3 |
| string | | string | |
| table | a1 | userdata | a3 |
| function | a2 | thread | a4 |
| userdata | a3 | function | a2 |
| thread | a4 | buffer | a5 |
| buffer | a5 | table | a1 |

#### VM_SHUFFLE7 & VM_SHUFFLE8

```c
const char* const luaT_eventname[] = {
    // ORDER TM
    VM_SHUFFLE7(VM_SHUFFLE_COMMA,
        "__index",
        "__newindex",
        "__mode",
        "__namecall",
        "__call",
        "__iter",
        "__len"
    ),
    "__eq",
    VM_SHUFFLE8(VM_SHUFFLE_COMMA,
        "__add",
        "__sub",
        "__mul",
        "__div",
        "__idiv",
        "__mod",
        "__pow",
        "__unm"
    ),
    VM_SHUFFLE5(VM_SHUFFLE_COMMA,
        "__lt",
        "__le",
        "__concat",
        "__type",
        "__metatable"
    ),
};
```

1. Locate `luaT_eventname` by searching for `__concat`.
2. Compare with original.

| Original | Position | Shuffle | Shuffle Position |
|----------|----------|---------|------------------|
| __index | a1 | __len | a7 |
| __newindex | a2 | __call | a5 |
| __mode | a3 | __iter | a6 |
| __namecall | a4 | __mode | a3 |
| __call | a5 | __index | a1 |
| __iter | a6 | __newindex | a2 |
| __len | a7 | __namecall | a4 |
| __eq | | __eq | |
| __add | a1 | __unm | a8 |
| __sub | a2 | __mul | a3 |
| __mul | a3 | __mod | a6 |
| __div | a4 | __div | a4 |
| __idiv | a5 | __idiv | a5 |
| __mod | a6 | __add | a1 |
| __pow | a7 | __pow | a7 |
| __unm | a8 | __sub | a2 |
| __lt | | __concat | |
| __le | | __type | |
| __concat | | __le | |
| __type | | __metatable | |
| __metatable | | __lt | |

#### VM_SHUFFLE4

```c++
typedef struct CallInfo {
    VM_SHUFFLE4(VM_SHUFFLE_SEMICOLON,
        StkId base,
        StkId func,
        StkId top,
        const Instruction* savedpc
    );
    int nresults;
    unsigned int flags;
} CallInfo;
```

1. Locate `dumpthread`, search for `{"type":"thread","cat":%d,"size":%d}`
2. Locate `->func`:

```c
fprintf(f, ",\"env\":");
fprintf(f, "\"%p\"", (const void *)th->Pad_58);
ci = (lCallInfo *)th->base_ci;
ci_max = (lCallInfo *)th->ci;
if ( ci <= ci_max )  {
    while ( 1 ) {
        tcl = (__int64 *)ci->func;
        if ( *(_DWORD *)(ci->func + 12) == 8 )
            break;
        if ( ++ci > ci_max )
            goto LABEL_9;
    }
    v9 = *tcl;
    if ( *tcl ) {
        if ( !*(_BYTE *)v9 + 3 ) {
            P = (lua_Proto *)(*(_QWORD *)v9 + 0x18 ^ (v9 + 0x18));
            if ( &P->source != (uintptr_t *)P->source ) {
                fprintf(f, ",\"source\":\"");
                dumpstringdata(...);
                fprintf(f, "\",\"line\":%d", P->linedefined);
            }
        }
    }
}
LABEL_9:
if ( th->top > th->stack ) {
    fprintf(f, ",\"stack\":[");
    ...
    fprintf(f, "]");
    _ci = (lCallInfo *)th->ci;
    first = 1;
    fprintf(f, ",\"stacknames\":[");
    v = th->stack;
    if ( v < th->top ) {
        do {
            if ( *(int *)(v + 12) >= 5 ) { //  if (!iscollectable(v))
                v17 = th->ci;
                if ( (unsigned __int64)_ci < v17 ) {
                    do {
                        v18 = _ci + 1;
                        if ( v < _ci[1].func )
                            break;
                        ++_ci;
                    } while ( (unsigned __int64)v18 < v17 );
                }
                if ( !first )
                    sub_7FF69D0421F8(44i64, f);
                func = (_DWORD *)_ci->func;
                first = 0;
                if ( v == _ci->func ) {
                    ...
                } else {
                    if ( func[3] != 8 || *(_BYTE *)*(_QWORD *)func + 3i64 )
                        goto LABEL_45;
                    v26 = _ci->savedpc;
                    v27 = ...
                    if ( v26 )
                        LODWORD(v26) = ...;
                    v28 = 0;
                    v29 = 0i64;
                    if ( ... <= 0 )
                        goto LABEL_45;
                    v30 = ...;
                    v31 = 0i64;
                    while ( 1 ) {
                        v32 = v31 - v30;
                        if ( (unsigned int)(__int64)(v - _ci->base) >> 4 == ...
                            && (int)v26 >= *(_DWORD *)(v32 + v27 + 64)
                            && (int)v26 < *(_DWORD *)(v32 + v27 + 68) ) 
                        {
                            break;    
                        }
                    }
                } 
            }
            v += 16i64;
        } while ( v < th->top );
    }
}
```

3. This gets us the offsets for `base`, `func` & `savedpc`. The odd one out is `top`.
4. Compare original with found one.

| Original | Position | Shuffle | Shuffle Position |
|----------|----------|---------|------------------|
| base | a1 | func | a2 |
| func | a2 | savedpc | a4 |
| top | a3 | top | a3 |
| savedpc | a4 | base | a1 |

#### VM_SHUFFLE6

```c++
struct lua_State {
    CommonHeader;
    uint8_t status;
    uint8_t activememcat;
    bool isactive;
    bool singlestep;
    VM_SHUFFLE6(VM_SHUFFLE_SEMICOLON,
        StkId top,
        StkId base,
        global_State* global,
        CallInfo* ci,
        StkId stack_last,
        StkId stack
    );
    CallInfo* end_ci;
    CallInfo* base_ci;
    int stacksize;
    int size_ci;
    unsigned short nCcalls;
    unsigned short baseCcalls;
    int cachedslot;
    VM_SHUFFLE3(VM_SHUFFLE_SEMICOLON,
        Table* gt,
        UpVal* openupval,
        GCObject* gclist
    );
    TString* namecall;
    extra_Space* extra_space;
};
```

1. What we need `base` & `global` as we already found the other members earlier.
2. We need to find math_random, search for "randomseed".

```c++
g = (global_State *)((char *)L->global - (char *)&L->global);
v3 = (__int64)(L->top - L->base) >> 4;
```

3. Compare to original

| Original | Position | Shuffle | Shuffle Position |
|----------|----------|---------|------------------|
| top | a1 | ci | a4 |
| base | a2 | top | a1 |
| global | a3 | stack | a6 |
| ci | a4 | stack_last | a5 |
| stack_last | a5 | global | a3 |
| stack | a6 | base | a2 |

#### VM_SHUFFLE9

```c++
VM_SHUFFLE9(VM_SHUFFLE_SEMICOLON,
    int sizecode,     /* X */
    int sizep,        /* X */
    int sizelocvars,  /* X */
    int sizeupvalues, /* X */
    int sizek,        /* X */
    int sizelineinfo, /* X */
    int linegaplog2,
    int linedefined,
    int bytecodeid
);
```

1. Locate `luaF_freeproto` by searching for `lua_newstate`, search for `Failed to create Lua state`.

```c++
  v19 = lua_newstate(0i64, a2);
  v20 = v19;
  if ( v19 ) { /* init state .*/ }
  sub_7FF69C867300("Failed to create Lua state");
```

2. Locate `close_state`.

```c++
lua_State *__fastcall lua_newstate(
        void *(__fastcall *a1)(void *, void *, unsigned __int64, unsigned __int64),
        void *a2) {
  // [COLLAPSED LOCAL DECLARATIONS. PRESS KEYPAD CTRL-"+" TO EXPAND]

  L = (lua_State *)allocate(0i64, 0i64, 0i64, 0x1C58ui64);
  g = L;
  if ( L )
  {
    /* init state */
    if ( luaD_rawrunprotected(g, f_luaopen, 0i64) )
    {
      close_state(g);
      return 0i64;
    }
    return g;
  }
  return L;
}
```

3. Locate `deletegco`.

```c++
__int64 __fastcall close_state(lua_State *L) {
  // [COLLAPSED LOCAL DECLARATIONS. PRESS KEYPAD CTRL-"+" TO EXPAND]
  p_global = &L->global;
  Pad_60 = L->Pad_60;
  v3 = (char *)L->global - (char *)&L->global;
  for ( i = L->stack; Pad_60; Pad_60 = L->Pad_60 ) {
    /* free stack */
  }
  luaM_visitgco(L, L, deletegco);
  /* other freeing */
}
```

4. Go to `freeobj`.

```c++
char __fastcall deletegco(__int64 a1, __int64 a2, __int64 a3) {
  freeobj(a1, a3, a2);
  return 1;
}
```

5. Locate `luaF_freeproto`.

```c++
void __fastcall freeobj(lua_State *L, void *Object, lua_Page *page) {
  // [COLLAPSED LOCAL DECLARATIONS. PRESS KEYPAD CTRL-"+" TO EXPAND]

    switch ( *(_BYTE *)Object ) {
        case 5:
          /* ... */
        case 7:
          /* ... */
        case 8:
          /* ... */
        case 9:
          /* ... */
        case 10:
          /* ... */
        case 11:
          luaF_freeproto(L, Object, page);
          return;
        case 12:
            /* ... */
        default:
            return;
    }
    while ( v12 != Object ) {
        /* ... */
    }
    /* ... */
}
```

6. Locate `sizecode`, `sizep`, `sizelocvars`, `sizeupvalues`, `sizek` & `sizelineinfo`.

```c++
void __fastcall luaF_freeproto(lua_State *L, lua_Proto *f, lua_Page *page) {
    // [COLLAPSED LOCAL DECLARATIONS. PRESS KEYPAD CTRL-"+" TO EXPAND]
    luaM_free_(
          L, 
          f->code ^ (unsigned __int64)&f->code, 4i64 * (int)f->sizecode,
          f->memcat
    );
    luaM_free_(
          L, 
          f->p ^ (unsigned __int64)&f->p, 
          8i64 * (int)f->sizep, f->memcat
    );
    luaM_free_(
        L, 
        f->k ^ (unsigned __int64)&f->k, 
        16i64 * (int)f->sizek,
        f->memcat
    );
    lineinfo = (lua_Proto *)f->lineinfo;
    if ( &f->lineinfo != (uint8_t **)lineinfo )
        luaM_free_(
            L, 
            (char *)f - (char *)lineinfo + 0x58, 
            (int)f->sizelineinfo, 
            f->memcat
    );
    luaM_free_(
        L, 
        (char *)&f->locvars - f->locvars, 
        24i64 * (int)f->sizelocvars, 
        f->memcat
    );
    luaM_free_(
        L, 
        (char *)&f->upvalues - f->upvalues, 
        8i64 * (int)f->sizeupvalues,
        f->memcat
    );
    debuginsn = f->debuginsn;
    if ( (uint8_t **)((char *)&f->debuginsn + (_QWORD)debuginsn) )
        luaM_free_(
            L, 
            (char *)&f->debuginsn + (_QWORD)debuginsn, 
            (int)f->sizecode, 
            f->memcat
    );
    if ( f->execdata )
        (*(void (__fastcall **)(lua_State *, lua_Proto *))(
            (char *)L->global - (char *)&L->global + 3376)
        )(L, f);
    typeinfo = (lua_Proto *)f->typeinfo;
    if ( typeinfo != (lua_Proto *)&f->typeinfo )
        luaM_free_(
            L,
            (char *)typeinfo - (char *)f - 112,
            f->sizetypeinfo,
            f->memcat
    );
    luaM_freegco_((__int64)L, (__int64)f, 176i64, f->memcat, (__int64)page);
}
```

7. We got `linedefined` @ VM_SHUFFLE4.
8. All we need is `linegaplog2`.
9. We need to locate `pusherror` as that contains `currentline`.
10. Search for `%s:%d: %s` & find the code similar to this:

```c++
ci = (lCallInfo *)L->ci;
v5 = ci->func;
if ( *(_DWORD *)(ci->func + 12) == 8 && !*(_BYTE *)(*(_QWORD *)v5 + 3i64) ) {
    v6 = ...;
    ((void (__fastcall *)(char *, const char *, char *))luaO_chunkid)(...);
    line = currentline(v7, ci);
    luaO_pushfstring(L, "%s:%d: %s", v9, line, msg);
    return;
}
```

11. Go to `currentline`

```c++
__int64 __fastcall currentline(lua_State *L, lCallInfo *ci) {
  // [COLLAPSED LOCAL DECLARATIONS. PRESS KEYPAD CTRL-"+" TO EXPAND]

    savedpc = ci->savedpc;
    if ( savedpc ) {
        code = ... + 8;
        LODWORD(savedpc) = ...
    }
    v4 = ...
    v5 = (lua_Proto *)v4->lineinfo;
    if ( &v4->lineinfo == (uint8_t **)v5 )
        return 0i64;
    else
        return *(_DWORD *)((char *)&v4->abslineinfo + 4 * ((__int64)(int)savedpc >> *(_BYTE *)((*(_QWORD *)(*(_QWORD *)ci->func + 24i64) ^ (*(_QWORD *)ci->func + 24i64)) + 0x8C)) - *(_QWORD *)((*(_QWORD *)(*(_QWORD *)ci->func + 24i64) ^ (*(_QWORD *)ci->func + 24i64)) + 0x48)) + (unsigned int)*((unsigned __int8 *)&v4->lineinfo + (int)savedpc - (_QWORD)v5);
}
```

12. The `+ 0x8C` is the offset for `linegaplog2`.
13. Compare to Original (once again!)

| Original            | Position | Shuffle            | Shuffle Position |
|---------------------|----------|---------------------|------------------|
| sizecode            | a1       | sizeupvalues        | a4               |
| sizep               | a2       | linegaplog2         | a7               |
| sizelocvars         | a3       | linedefined         | a8               |
| sizeupvalues        | a4       | sizelineinfo        | a6               |
| sizek               | a5       | sizecode            | a1               |
| sizelineinfo        | a6       | (a6) bytecodeid     | a9               |
| linegaplog2         | a7       | sizek               | a5               |
| linedefined         | a8       | sizelocvars         | a3               |
| bytecodeid          | a9       | sizep               | a2               |