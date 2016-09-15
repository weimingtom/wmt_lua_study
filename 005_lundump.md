Lua5.1代码阅读（五）：lundump.h/lundump.c

（未完成，待修改）

一、概述
	lundump.h和lundump.c是lua预编译二进制代码的加载器。
	不同于llex/lparser/lcode串联起来对文本脚本的解析和编译，
	lundump解析的是二进制脚本文件（由luac编译生成）。
	它的公开接口luaU_undump和luaY_parser的声明原型是相同的，
	所以可以把lundump看成是lparser的另一种实现。
	另外，由于lundump解析的二进制文件的结构是嵌套而且动态变长的，
	所以lundump大多时候是手工解码，而且依赖于ldump的写入格式。

二、头文件
	#include "lobject.h"
	#include "lzio.h"
	#include <string.h>
	#include "lua.h"
	#include "ldebug.h"
	#include "ldo.h"
	#include "lfunc.h"
	#include "lmem.h"
	#include "lobject.h"
	#include "lstring.h"
	#include "lundump.h"
	#include "lzio.h"

三、宏
	1. #define LUAC_VERSION		0x51
		二进制文件的头，指示现在为Lua 5.1
	2. #define LUAC_FORMAT		0
		二进制文件的头，指示这是官方格式
	3. #define LUAC_HEADERSIZE		12
		二进制文件的头长度
	4. #define lundump_c
		表示在lundump.c中
	5. #define LUA_CORE
		表示lundump为Lua的内部实现
	6. #define IF(c,s)
	   #define IF(c,s)		if (c) error(S,s)
		可以在某个条件满足时报告错误并抛出异常。
		可以用LUAC_TRUST_BINARIES宏屏蔽（默认不屏蔽）
	7. #define error(S,s)
		用LUAC_TRUST_BINARIES宏屏蔽错误和避免抛异常（默认不屏蔽）
	8. #define LoadMem(S,b,n,size)	LoadBlock(S,b,(n)*(size))
		从S中读取n * size大小的内存，保存至b。
	9. #define	LoadByte(S)		(lu_byte)LoadChar(S)
		从S中读取一个字节的内容，然后返回。
	10. #define LoadVar(S,x)		LoadMem(S,&x,1,sizeof(x))
		从S中读取变量x的值（字节长度和x相同），保存至x。
	11. #define LoadVector(S,b,n,size)	LoadMem(S,b,n,size)
		从S中读取n * size大小的数组，保存至b。
	
四、结构体
	1. typedef struct {
	 lua_State* L;
	 ZIO* Z;
	 Mbuffer* b;
	 const char* name;
	} LoadState;
		加载状态，lundump的数据中心，携带L状态机。
	2.
	
五、静态私有函数
	1. static void error(LoadState* S, const char* why)
		使用S中的L状态机报告错误并抛出LUA_ERRSYNTAX异常。
	2. static void LoadBlock(LoadState* S, void* b, size_t size)
		从S中读取size大小的字节块，输出到b。
	3. static int LoadChar(LoadState* S)
		从S中读取char变量（实际上返回int）
	4. static int LoadInt(LoadState* S)
		从S中读取int值（实际上返回的是非负整数）
	5. static lua_Number LoadNumber(LoadState* S)
		从S中读取lua_Number值。
	6. static TString* LoadString(LoadState* S)
		从S中读取TString字符串（符号表字符串）。
		过程如下：
		* 读取size_t型的size变量（字符串长度，包括末尾的\0）
		* 如果长度为0，直接返回空指针
		* 确保S->b中的变长内存块足够大，否则就扩大。
		* 读出size长度的字节块
		* 删除末尾的\0，然后转换为TString字符串。
	7. static void LoadCode(LoadState* S, Proto* f)
		从S中读取代码（指令数组），填充原型f。
		过程如下：
		* 读取整型值n（指令数）
		* 创建一个Instruction[n]数组，保存到f->code
		* f->sizecode就是n
		* 读取并填充f->code数组。
	8. static void LoadConstants(LoadState* S, Proto* f)
		从S中加载常数表，填充原型f
		过程如下：
		* 读取整型值n（TValue个数）
		* 创建数组TValue[n]，保存到f->k
		* f->sizek就是n
		* 把f->k[n]填充为nil
		* 填充f->k[n]为实际的值：
			第一个char值表示常数类型，如下：
			LUA_TNIL表示nil，
			LUA_TBOOLEAN表示布尔值，再读一个char得到它的值。
			LUA_TNUMBER表示数值，再读一个Number得到它的值。
			LUA_TSTRING表示字符串，再读一个TString得到它的值。
			如果出现其他类型值，则报错。
		* 读取整型值n（嵌套函数的个数）
		* 创建Proto*[n]，保存至f->p
		* f->sizep就是n
		* 清空f->p[n]数组
		* 加载n个嵌套函数到f->p[n]（函数内嵌套的函数被认为是特殊的常数）
		* 嵌套函数的缺省源文件名为父函数的源文件名。
	9. static void LoadDebug(LoadState* S, Proto* f)
		加载调试信息。
		过程如下：
		* 读取整型值n
		* 在S->L上分配int[n]数组，地址赋值给f->lineinfo
		* f->sizelineinfo就是n
		* 从S中把int[n]数组读到f->lineinfo数组中
		* 读取整型值n
		* 在S->L上分配LocVar[n]数组，地址赋值给f->locvars
		* f->sizelocvars就是ns
		* 清空所有f->locvars[i].varname为NULL
		* 从S中读取并填充每个f->locvars[i]的域
			varname : 字符串
			startpc : 整型
			endpc : 整型
		* 读取整型值n
		* 在S->L上分配TString*[n]数组，地址赋值给f->upvalues
		* f->sizeupvalues就是n
		* 清空f->upvalues[i]为NULL
		* 从S中读取字符串数组并填充f->upvalues[n]
	10. static Proto* LoadFunction(LoadState* S, TString* p)
		加载函数
		过程如下
		* 检查嵌套是否超过LUAI_MAXCCALLS
		（使用L->nCcalls，把嵌套层数看成C调用的个数）
		* 创建新的函数原型f
		* 把f压进S->L栈（通过L栈为以后嵌套的函数提供相关信息）
		* 从S中读取字符串作为f->source（源文件名）。
		* 如果f->source为空（通常嵌套的函数读到的是空字符串），则赋参数p。
		* 从S中读取整型f->linedefined（定义的行号）
		* 从S中读取整型f->lastlinedefined（定义的最后行号）
		* 从S中读取字节值f->nups（上值个数）
		* 从S中读取字节值f->numparams（参数个数）
		* 从S中读取字节值f->is_vararg（判断参数表是否变长）
		* 从S中读取字节值f->maxstacksize（最大堆栈大小）
		* 从S中读取代码数组保存到f
		* 从S中读取常数表（包括嵌套函数）保存到f
		* 从S中读取调试信息保存到f
		* 从头到尾模拟运行一次进行代码检查
		* 还原栈顶和嵌套层数
	11. static void LoadHeader(LoadState* S)
		加载文件头
		过程如下：
		* 创建对于当前平台是正确的文件头h[LUAC_HEADERSIZE]
		* 从S中读取LUAC_HEADERSIZE长度的字节块，保存到s[LUAC_HEADERSIZE]
		* 比较h和s，内容必须完全相同
	
六、公开的导出函数
	1. Proto* luaU_undump (lua_State* L, ZIO* Z, Mbuffer* buff, const char* name)
		加载预编译二进制文件的内容
		（注意，开头的<esc>Lua已经被读取）。
		过程如下：
		* 创建局部变量S（加载状态）
		* 用参数name确定S.name
		（name可能是@开头，=开头，<esc>Lua，一般字符串）
		* 把参数L、Z、buff都保存到S中
		* 加载文件头
		* 加载函数，缺省的源文件名为=?
	2. void luaU_header (char* h)
		制作适合当前程序的正确文件头
		（在lundump中用于校验读取到的文件头内容）
		格式为：
		1 byte : 版本号
		1 byte : 格式号
		1 byte : 数的小端是否在前
		1 byte : int的长度
		1 byte : size_t的长度
		1 byte : 指令类型Instruction的长度
		1 byte : 数lua_Number的长度
		这些值都在编译期决定
		
七、值得观察的代码
	1. LoadFunction / LoadConstants
		这两个函数相互调用（嵌套分析二进制代码）
		lundump认为嵌套的子函数属于常量表，
		而子函数可能包含常量表和它自己的子函数，
		直至没有子函数为止。
	2. luaU_undump
		lundump模块的入口。
		最开始认为整个文件（除头部外）是一个函数。
		然后递归地加载其中的所有常量表和子函数代码。
		
（未完成，待修改）
