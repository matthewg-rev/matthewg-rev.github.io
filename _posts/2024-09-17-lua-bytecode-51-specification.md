## Lua Bytecode 5.1 Specification

This blog post will be dedicated to my self-documentation about the Lua 5.1 bytecode specification. This post is written with accordance to the Lua 5.1 source code for the files:
- ldump.c
- lundump.c
- lvm.c
- lopcodes.h

## The Lua Header

In ldump.c we have a C function titled `luaU_dump(lua_State* L, const Proto* f, lua_Writer w, void* data, int strip)`, this function is responsible for handling the calls to write all the compiled data to a given file writer. Following this function's process we can start disecting the first function call `DumpHeader(&D);`. This function creates a `char[LUAC_HEADERSIZE]`which is a byte array to store the compiled info for the lua file header. This character array is handled by the luaU_header function and populates the header with information about the lua file. The DumpBlock function then handles the header array along with the `DumpState*`. 
```C
static void DumpHeader(DumpState* D)
{
 char h[LUAC_HEADERSIZE];
 luaU_header(h);
 DumpBlock(h,LUAC_HEADERSIZE,D);
}

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

Inside lundump.c the function `luaU_header (char* h)` allows us to see the encoding of a lua header. Walking through this function we find ourselves memory copying the `LUA_SIGNATURE` which in our case is `"\033Lua"` to the `char* h` array. Now following this operation we store `LUAC_VERSION` as a `char`(`byte`) into the header byte array. Proceeding we do the same thing with `LUAC_FORMAT`. Now unless this has changed the endianness is also stored and written as a singular byte value `1` into the header byte array. Afterwards we encode the encoding size for `instructions`, `constants`, and `prototypes`. Now this encoded byte has two possible values for our use case: `4` or `8`. Depending on this size the arrays for `instructions`, `constants`, and `prototypes`. Following this we encode the size of our `size_t` which is also `4` or `8`. The only use case as far as I am aware is the size of encoded `string` types. We also encode the size of our `Instruction` and `lua_Number` both of these types will only encode either as `4` or `8`. In the case of `Instruction` depending on whether it's `4` or `8` we will read instructions as either `uint32_t` or `uint64_t`. In the case of `lua_Number` depending on whether it's `4` or `8` we will read numbers as either 32-bit or 64-bit floats. The last byte for the header represents whether `lua_Number`'s that are decimals will round down to the integer value. 
```C
void luaU_header (char* h)
{
 int x=1;
 memcpy(h,LUA_SIGNATURE,sizeof(LUA_SIGNATURE)-1);
 h+=sizeof(LUA_SIGNATURE)-1;
 *h++=(char)LUAC_VERSION;
 *h++=(char)LUAC_FORMAT;
 *h++=(char)*(char*)&x;                         /* endianness */
 *h++=(char)sizeof(int);
 *h++=(char)sizeof(size_t);
 *h++=(char)sizeof(Instruction);
 *h++=(char)sizeof(lua_Number);
 *h++=(char)(((lua_Number)0.5)==0);             /* is lua_Number integral? */
}
```