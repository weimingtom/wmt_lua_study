# Lua5.1代码阅读（二）：llex.h/llex.c

## 一、作用和参考资料
llex.c是Lua的词法分析器（把单个输入字符串切割为多个输出符号）  
参考：  
* Lua 5.1.3源代码分析之词法分析 By 天地沙鸥  
http://xenyinzen.wordpress.com/2009/12/09/lua-5-1-3%E6%BA%90%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%E8%AF%8D%E6%B3%95%E5%88%86%E6%9E%90/  
	
## 二、包含头文件  
* llex.h
`    #include "lobject.h"
`    #include "lzio.h"

* llex.c
`    #include <ctype.h>
`    #include <locale.h>
`    #include <string.h>
`    #include "lua.h"
`    #include "ldo.h"
`    #include "llex.h"
`    #include "lobject.h"
`    #include "lparser.h"
`    #include "lstate.h"
`    #include "lstring.h"
`    #include "ltable.h"
`    #include "lzio.h"

## 三、宏
* #define FIRST_RESERVED	257
    RESERVED枚举值第一个枚举的整数值。
    大于256的目的可能是想避开ASCII值的范围（见luaX_token2str）。

* #define TOKEN_LEN	(sizeof("function")/sizeof(char))
    保留字的最大内存长度（包括结尾的'\0'）。

* #define NUM_RESERVED	(cast(int, TK_WHILE-FIRST_RESERVED+1))
    保留字的个数（cast是定义在llimit.h的宏，用于C强制转换）。

* #define llex_c
    llex.c的宏缩写（象征形式）

* #define LUA_CORE
    表示llex.c位于底层实现中（象征形式）

* #define next(ls) (ls->current = zgetc(ls->z))
    从输入流中读取一个字符值，保存为ls（词法分析器）的当前字符值

* #define currIsNewline(ls)	(ls->current == '\n' || ls->current == '\r')
    判断当前是否回车。

* #define save_and_next(ls) (save(ls, ls->current), next(ls))
    把当前字符保存到变长缓冲区，然后读下一个符号。

* #define MAXSRC          80
    用于处理源文件名ls->source的临时缓冲区的最大长度（？）
		
## 四、枚举
	1. 
	enum RESERVED {
	  TK_AND = FIRST_RESERVED, TK_BREAK,
	  TK_DO, TK_ELSE, TK_ELSEIF, TK_END, TK_FALSE, TK_FOR, TK_FUNCTION,
	  TK_IF, TK_IN, TK_LOCAL, TK_NIL, TK_NOT, TK_OR, TK_REPEAT,
	  TK_RETURN, TK_THEN, TK_TRUE, TK_UNTIL, TK_WHILE,
	  TK_CONCAT, TK_DOTS, TK_EQ, TK_GE, TK_LE, TK_NE, TK_NUMBER,
	  TK_NAME, TK_STRING, TK_EOS
	};
		保留字枚举量（和luaX_tokens对应），表示特殊的token范围。
		如果要修改它的顺序，需要修改注释有ORDER RESERVED的代码。
		* TK_AND到TK_WHILE是表示保留字的终结符（终结符是指语法中不需要继续推导的符号）
		* TK_CONCAT到TK_EOS是非保留字的其它终结符（运算符、字面值和文件结束）。
		
## 五、公开的全局变量
	1. LUAI_DATA const char *const luaX_tokens [];
		const char *const luaX_tokens [] = {
			"and", "break", "do", "else", "elseif",
			"end", "false", "for", "function", "if",
			"in", "local", "nil", "not", "or", "repeat",
			"return", "then", "true", "until", "while",
			"..", "...", "==", ">=", "<=", "~=",
			"<number>", "<name>", "<string>", "<eof>",
			NULL
		};
		names符号（终结符）的常量字符串数组（对应RESERVED枚举的顺序）。
		用于把特殊符号转换为字符串。
		
## 六、结构体、联合体
	1. 
	typedef union {
	  lua_Number r;
	  TString *ts;
	} SemInfo;
	单个语义信息（联合体）
	2. 
	typedef struct Token {
	  int token;
	  SemInfo seminfo;
	} Token;
	单个符号信息（结构体）
	3. 
	typedef struct LexState {
	  int current;  
	  int linenumber;  
	  int lastline;  
	  Token t;  
	  Token lookahead;  
	  struct FuncState *fs;  
	  struct lua_State *L;
	  ZIO *z;  
	  Mbuffer *buff;  
	  TString *source;  
	  char decpoint;  
	} LexState;
	词法状态：
	* current：当前int型字符值
	* linenumber：输入行号
	* lastline：最后一个被消费的符号的行号。
	* Token：当前符号信息
	* lookahead：向前看的符号信息
	* fs：函数状态，对于parser是私有的
	* L：Lua状态机
	* z：输入流
	* buff：符号缓冲
	* source：当前源码文件名
	* decpoint：本地区域的十进制浮点
	
## 七、全局私有的静态方法
	1. static void save (LexState *ls, int c) {
		把c接到ls->buff变长缓冲区的结尾（确保不会超出缓冲区范围）
		
	2. static const char *txtToken (LexState *ls, int token) {
		根据token取出字面值。
		如果token是名称、数、字符串字面值，把ls->buff转成'\0'结束的C字符串。
		如果token是特殊token（保留字、运算符），直接返回其字符串形式。
		
	3. static void inclinenumber (LexState *ls) {
		增加行号（检查溢出）。
		然后跳过附近的'\n'和'\r'。
		
	4. static int check_next (LexState *ls, const char *set) {
		检查词法状态机的当前字符值是否在字符集set内。
		如果是，则执行save_and_next
	
	5. static void buffreplace (LexState *ls, char from, char to) {
		把词法状态机的缓冲区中的字符值from全部替换为字符值to。
	
	6. static void trydecpoint (LexState *ls, SemInfo *seminfo) {
		把'.'改为本地（locale）的十进制小数点分割符
		（使用<locale.h>的函数localeconv获取本地数字及货币信息格式），
		然后继续尝试字符串到数字的转换。
		
	7. static void read_numeral (LexState *ls, SemInfo *seminfo) {
		循环（逐字符）读取LUA_NUMBER（内部实现是double型）字面值字符串
		
	8. static int skip_sep (LexState *ls) {
		循环读取[=或]=分割符，
		然后返回其层数（'='的个数，如果是右面则为负数）
		
	9. static void read_long_string (LexState *ls, SemInfo *seminfo, int sep) {
		读带分隔符的字符串字面值（如[[abc]]）
		
	10. static void read_string (LexState *ls, int del, SemInfo *seminfo) {
		读一般的字符串字面值（如"abc"，'abc'）
		
	11. static int llex (LexState *ls, SemInfo *seminfo) {
		词法分析器循环，使用switch进行跳转。
		
	
## 八、全局公开的方法
	1. void luaX_init (lua_State *L) {
		LUAI_FUNC void luaX_init (lua_State *L);
		初始化词法分析器：
		* 把所有保留字加入符号表。
		* 诊断TOKEN_LEN是最大的保留字长度。
		
	2. const char *luaX_token2str (LexState *ls, int token) {
		LUAI_FUNC const char *luaX_token2str (LexState *ls, int token);
		把int型的token（可印刷字符、控制字符、保留字符号）转为可读的字符串。
		
	3. void luaX_lexerror (LexState *ls, const char *msg, int token) {
		LUAI_FUNC void luaX_lexerror (LexState *ls, const char *msg, int token);
		报告词法错误，然后抛出LUA_ERRSYNTAX异常。
		
	4. void luaX_syntaxerror (LexState *ls, const char *msg) {
		LUAI_FUNC void luaX_syntaxerror (LexState *ls, const char *s);
		报告语法错误，然后抛出LUA_ERRSYNTAX异常。
		和luaX_lexerror相同，只是报告的token是当前词法状态机的符号。
		
	5. TString *luaX_newstring (LexState *ls, const char *str, size_t l) {
		LUAI_FUNC TString *luaX_newstring (LexState *ls, const char *str, size_t l);
		在符号表中创建字符串（确保不会重复收集），返回TString指针（符号表字符串）。
		
	6. void luaX_setinput (lua_State *L, LexState *ls, ZIO *z, TString *source) {
		LUAI_FUNC void luaX_setinput (lua_State *L, LexState *ls, ZIO *z, TString *source);
		初始化词法状态机的输入流ls，然后读取第一个字符值
		
	7. void luaX_next (LexState *ls) {
		LUAI_FUNC void luaX_next (LexState *ls);
		读下一个词法符号。
		如果已经有被lookahead的词法符号，就直接取它的值避免遗漏。
		
	8. void luaX_lookahead (LexState *ls) {
		LUAI_FUNC void luaX_lookahead (LexState *ls);
		向前看（lookahead，即提前读）后一个词法符号。用于语法预测。
		
## 九、值得观察的关键代码
	1. llex
		它是luaX_next和luaX_lookahead的底层实现
		