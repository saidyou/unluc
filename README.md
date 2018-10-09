
## 前言
测试某游戏时，游戏加载的是luac脚本：
- **文件格式** - 010editor官方bt只能识别luac52版本 
- **opcode对照表** - 这个游戏luac的opcode对照表也被重新排序，unluac需要找到lua vm的opcode对照表，才能反编译。

----------

@[toc]
## luac51格式分析
### Luac文件格式
一个Luac文件包含两部分：文件头与函数体。
#### 文件头格式 
```cpp
typedef struct {
    char signature[4];   //".lua"
    uchar version;
    uchar format;
    uchar endian;
    uchar size_int;
    uchar size_size_t;
    uchar size_Instruction;
    uchar size_lua_Number;
    uchar lua_num_valid;
    uchar luac_tail[0x6];
} GlobalHeader;
```
第一个字段**`signature`**在lua.h头文件中有定义，它是LUA_SIGNATURE，取值为“\033Lua"，其中，\033表示按键<esc>。LUA_SIGNATURE作为Luac文件开头的4字节，它是Luac的Magic Number，用来标识它为Luac字节码文件。Magic Number在各种二进制文件格式中比较常见，通过是特定文件的前几个字节，用来表示一种特定的文件格式。

**`version`** 字段表示Luac文件的格式版本，它的值对应于Lua编译的版本，对于5.2版本的Lua生成的Luac文件，它的值为0x52。

**`format`**字段是文件的格式标识，取值0代表official，表示它是官方定义的文件格式。这个字段的值不为0，表示这是一份经过修改的Luac文件格式，可能无法被官方的Lua虚拟机正常加载。

**`endian`**表示Luac使用的字节序。现在主流的计算机的字节序主要有小端序LittleEndian与大端序BigEndian。这个字段的取值为1的话表示为LittleEndian，为0则表示使用BigEndian。

**`size_int`**字段表示int类型所占的字节大小。size_size_t字段表示size_t类型所占的字节大小。这两个字段的存在，是为了兼容各种PC机与移动设备的处理器，以及它们的32位与64位版本，因为在特定的处理器上，这两个数据类型所占的字节大小是不同的。

**`size_Instruction`**字段表示Luac字节码的代码块中，一条指令的大小。目前，指令Instruction所占用的大小为固定的4字节，也就表示Luac使用等长的指令格式，这显然为存储与反编译Luac指令带来了便利。

**`size_lua_Number`**字段标识lua_Number类型的数据大小。lua_Number表示Lua中的Number类型，它可以存放整型与浮点型。在Lua代码中，它使用LUA_NUMBER表示，它的大小取值大小取决于Lua中使用的浮点数据类型与大小，对于单精度浮点来说，LUA_NUMBER被定义为float，即32位大小，对于双精度浮点来说，它被定义为double，表示64位长度。目前，在macOS系统上编译的Lua，它的大小为64位长度。

**`lua_num_valid`**字段通常为0，用来确定lua_Number类型能否正常的工作。

**`luac_tail`**字段用来捕捉转换错误的数据。在Lua中它使用LUAC_TAIL表示，这是一段固定的字符串内容："\x19\x93\r\n\x1a\n"。

在文件头后面，紧接着的是函数体部分。一个Luac文件中，位于最上面的是一个顶层的函数体，函数体中可以包含多个子函数，子函数可以是嵌套函数、也可以是闭包，它们由常量、代码指令、Upvalue、行号、局部变量等信息组成。
#### 函数体
```cpp
typedef struct {
    //header
    ProtoHeader header;

    //code
    Code code;

    // constants
    Constants constants;

    // functions
    Protos protos;

    // upvalues
  //Upvaldescs upvaldescs;

    // string
    //SourceName src_name;

    // lines
    Lines lines;
    
    // locals
    LocVars loc_vars;
    
    // upvalue names
    UpValueNames names;
} Proto;
```
**`ProtoHeader`**是Proto的头部分。它的定义如下：
```cpp
typedef struct {
    uint32 linedefined;
    uint32 lastlinedefined;
    uchar numparams;
    uchar is_vararg;
    uchar maxstacksize;
} ProtoHeader;
```
**`code`**lua vm操作码
**`constants`** lua语句其实只是将标志符合关键字给去掉了
```lua
printf("Hello %s from %s on %s\n",os.getenv"USER" or "there",_VERSION,os.date())
```
这是一条lua语句，那么在luac文件中可见,可以推测出大概的lua语句
```
���printf����Hello %s from %s on %s
����os����getenv����USER����there�  ���_VERSION����date
```
**`protos`**下一个函数体，函数体是链式存储的
**`lines`**该函数存在多少行lua语句
**`loc_vars`**函数内部局部变量个数
**`names`**函数内部局部变量名

### luac.exe存储luac过程分析
luac的函数体在luac.exe中分main函数和一般函数，存储过程如下：
1、存储main函数头、代码、lua语句字符串进行存储
2、存储普通函数
  2.1、存储普通函数体
  2.2、遍历普通函数链表，直至结束
3、存储main函数行数、参数、参数名


```
#### 具体过程
分析luac.exe存储过程，很容易分析出luac的文件格式。
```cpp
int luaU_dump (lua_State* L, const Proto* f, lua_Writer w, void* data, int strip)
{
 DumpState D;
 D.L=L;
 D.writer=w;
 D.data=data;
 D.strip=strip;
 D.status=0;
 DumpHeader(&D);
 DumpFunction(f,NULL,&D);
 return D.status;
}

```
lua源码luac.c中，**`DumpHeader`** 存储了luac头格式，而在**`DumpFunction`**则对函数体进行了存储。
```cpp
static void DumpHeader(DumpState* D)
{
 char h[LUAC_HEADERSIZE];
 luaU_header(h);
 DumpBlock(h,LUAC_HEADERSIZE,D);
}
```
**`DumpHeader `**存储了LUAC_HEADERSIZE（12）byte的头格式。
```
static void DumpConstants(const Proto* f, DumpState* D)
{
 int i,n=f->sizek;
 DumpInt(n,D);
 for (i=0; i<n; i++)
 {
  const TValue* o=&f->k[i];
  DumpChar(ttype(o),D);
  switch (ttype(o))
  {
   case LUA_TNIL:
  break;
   case LUA_TBOOLEAN:
  DumpChar(bvalue(o),D);
  break;
   case LUA_TNUMBER:
  DumpNumber(nvalue(o),D);
  break;
   case LUA_TSTRING:
  DumpString(rawtsvalue(o),D);
  break;
   default:
  lua_assert(0);      /* cannot happen */
  break;
  }
 }
 n=f->sizep;
 DumpInt(n,D);
 for (i=0; i<n; i++) DumpFunction(f->p[i],f->source,D);
}

static void DumpDebug(const Proto* f, DumpState* D)
{
 int i,n;
 n= (D->strip) ? 0 : f->sizelineinfo;
 DumpVector(f->lineinfo,n,sizeof(int),D);
 n= (D->strip) ? 0 : f->sizelocvars;
 DumpInt(n,D);
 for (i=0; i<n; i++)
 {
  DumpString(f->locvars[i].varname,D);
  DumpInt(f->locvars[i].startpc,D);
  DumpInt(f->locvars[i].endpc,D);
 }
 n= (D->strip) ? 0 : f->sizeupvalues;
 DumpInt(n,D);
 for (i=0; i<n; i++) DumpString(f->upvalues[i],D);
}

static void DumpFunction(const Proto* f, const TString* p, DumpState* D)
{
 DumpString((f->source==p || D->strip) ? NULL : f->source,D);
 DumpInt(f->linedefined,D);
 DumpInt(f->lastlinedefined,D);
 DumpChar(f->nups,D);
 DumpChar(f->numparams,D);
 DumpChar(f->is_vararg,D);
 DumpChar(f->maxstacksize,D);
 DumpCode(f,D);
 DumpConstants(f,D);
 DumpDebug(f,D);
}
```
递归遍历存储函数体过程。

----------
## luac反编译
### 获取lua51 vm
```bash
  $ git clone https://github.com/mkottman/AndroLua
  $ ndk-build
```
### lua_execute函数
在lua vm中luac解释执行luac opcode的函数是luaV_execute，但是有些so会将lua_execute函数名隐藏，以下是通过lua_resume找到execute函数。
```cpp
  LUA_API int lua_resume (lua_State *L, int nargs) {
    int status;
    lua_lock(L);
    if (L->status != LUA_YIELD && (L->status != 0 || L->ci != L->base_ci))
        return resume_error(L, "cannot resume non-suspended coroutine");
    if (L->nCcalls >= LUAI_MAXCCALLS)
      return resume_error(L, "C stack overflow");
    luai_userstateresume(L, nargs);
    lua_assert(L->errfunc == 0);
    L->baseCcalls = ++L->nCcalls;
    status = luaD_rawrunprotected(L, resume, L->top - nargs);
    if (status != 0) {  /* error? */
      L->status = cast_byte(status);  /* mark thread as `dead' */
      luaD_seterrorobj(L, status, L->top);
      L->ci->top = L->top;
    }
    else {
      lua_assert(L->nCcalls == L->baseCcalls);
      status = L->status;
    }
    --L->nCcalls;
    lua_unlock(L);
    return status;
  }
```
lua_resume中调用luaD_rawrunprotected函数，将resum函数指针作为参数传入
```cpp
  static void resume (lua_State *L, void *ud) {
    StkId firstArg = cast(StkId, ud);
    CallInfo *ci = L->ci;
    if (L->status == 0) {  /* start coroutine? */
      lua_assert(ci == L->base_ci && firstArg > L->base);
      if (luaD_precall(L, firstArg - 1, LUA_MULTRET) != PCRLUA)
        return;
    }
    else {  /* resuming from previous yield */
      lua_assert(L->status == LUA_YIELD);
      L->status = 0;
      if (!f_isLua(ci)) {  /* `common' yield? */
        /* finish interrupted execution of `OP_CALL' */
        lua_assert(GET_OPCODE(*((ci-1)->savedpc - 1)) == OP_CALL ||
                   GET_OPCODE(*((ci-1)->savedpc - 1)) == OP_TAILCALL);
        if (luaD_poscall(L, firstArg))  /* complete it... */
          L->top = L->ci->top;  /* and correct top if not multiple results */
      }
      else  /* yielded inside a hook: just continue its execution */
        L->base = L->ci->base;
    }
    luaV_execute(L, cast_int(L->ci - L->base_ci));
  }

```
resum函数中调用luaV_execute
```
void luaV_execute (lua_State *L, int nexeccalls) {

    ...

    switch (GET_OPCODE(i)) {
      case OP_MOVE: {
        setobjs2s(L, ra, RB(i));
        continue;
      }
      case OP_LOADK: {
        setobj2s(L, ra, KBx(i));
        continue;
      }
      case OP_LOADBOOL: {
        setbvalue(ra, GETARG_B(i));
        if (GETARG_C(i)) pc++;  /* skip next instruction (if C) */
        continue;
      }

    ...

    }
  }
}
```
luaV_execute中存在38个switch case，然后再该游戏的lua vm中找到execute函数同样也是38个switch case，分析得到opcode对照表：
```
game orgin
  0 0
  1 24
  2 c
  3 d
  4 1
  5 2 
  6 3 
  7 4 
  8 5
  9 7 
  a 8
  b 6
  c a
  d b
  e 9
  f e
  10 f
  11 10
  12 11
  13 12 
  14 13
  15 14
  16 15
  17 16
  18 17
  19 18
  1a 19
  1b 1a
  1c 1b
  1d 1c
  1e 1d
  1f 1e
  20 1f
  21 20
  22 21
  23 22
  24 23
  25 25
```
### 编译unluac
  将分析出来的对照表覆盖到unluac的OpcodeMap.java中进行编译，生成unluac.jar
  ```
      } else if(version == 0x51) {
      map = new Op[38];
      map[0] = Op.MOVE;
      map[36] = Op.LOADK;
      map[12] = Op.LOADBOOL;
      map[13] = Op.LOADNIL;
      map[1] = Op.GETUPVAL;
      map[2] = Op.GETGLOBAL;
      map[3] = Op.GETTABLE;
      map[4] = Op.SETGLOBAL;
      map[7] = Op.SETUPVAL;
      map[5] = Op.SETTABLE;
      map[8] = Op.NEWTABLE;
      map[9] = Op.SELF;
      map[10] = Op.ADD;
      map[11] = Op.SUB;
      map[6] = Op.MUL;
      map[14] = Op.DIV;
      map[15] = Op.MOD;
      map[16] = Op.POW;
      map[17] = Op.UNM;
      map[18] = Op.NOT;
      map[19] = Op.LEN;
      map[20] = Op.CONCAT;
      map[21] = Op.JMP;
      map[22] = Op.EQ;
      map[23] = Op.LT;
      map[24] = Op.LE;
      map[25] = Op.TEST;
      map[26] = Op.TESTSET;
      map[27] = Op.CALL;
      map[28] = Op.TAILCALL;
      map[29] = Op.RETURN;
      map[30] = Op.FORLOOP;
      map[31] = Op.FORPREP;
      map[32] = Op.TFORLOOP;
      map[33] = Op.SETLIST;
      map[34] = Op.CLOSE;
      map[35] = Op.CLOSURE;
      map[37] = Op.VARARG; 
    } else if(version == 0x52) {
  ```
  最后尝试反编译luac文件，成功！！！~~~
  ```
  java -jar unluac.jar test.lua
  ```

## luac
> **Note:** 资料参考:
> - **luac反编译工具 **：https://github.com/viruscamp/unluac
> - **lua52反编译分析博客** : https://github.com/feicong/lua_re
> - **lua51源码** :  https://github.com/lua/lua/tree/v5_1
