Lua5.1代码阅读（四）：lparser.h/lparser.c

（未完成，待修改）

一、概述
	lparser.h/lparser.c是Lua的语法分析器。
	用于分析Lua脚本的语法以及把上下文信息传递给代码生成器，
	完成文本代码到二进制代码的转换，以及语法检查。
	在线版：
	http://www.lua.org/source/5.1/lparser.h.html
	http://www.lua.org/source/5.1/lparser.c.html
	lparser内部主要函数的引用图如下：（只含部分）
	
	在Lua参考手册（LRM）中描述了Lua的基本语法，见
	http://www.lua.org/manual/5.1/manual.html#8
	可以画出的关系图大概是这样子：

二、枚举值
	1. typedef enum {
	  VVOID,	/* 无值 */
	  VNIL,
	  VTRUE,
	  VFALSE,
	  VK,		/* info = k内常量的索引 */
	  VKNUM,	/* nval = 数值 */
	  VLOCAL,	/* info = 局部寄存器 */
	  VUPVAL,   /* info = upvalues内的上值索引 */
	  VGLOBAL,	/* info = 表索引; aux = k内全局名称的索引 */
	  VINDEXED,	/* info = 表寄存器; aux = 索引寄存器（或k） */
	  VJMP,		/* info = 指令pc */
	  VRELOCABLE,	/* info = 指令pc */
	  VNONRELOC,	/* info = 结果寄存器 */
	  VCALL,	/* info = 指令pc */
	  VVARARG	/* info = 指令pc */
	} expkind;

三、结构体、联合体
	1. typedef struct expdesc {
	  expkind k;
	  union {
		struct { int info, aux; } s;
		lua_Number nval;
	  } u;
	  int t;  /* 真时退出的补丁列表 */
	  int f;  /* 假时退出的补丁列表 */
	} expdesc;
	2. typedef struct upvaldesc {
	  lu_byte k;
	  lu_byte info;
	} upvaldesc;
	3. typedef struct FuncState {
	  Proto *f;  /* 当前函数头 */
	  Table *h;  /* 用来查找（或重用）k内元素的表*/
	  struct FuncState *prev;  /* 闭合函数 */
	  struct LexState *ls;  /* 词法状态 */
	  struct lua_State *L;  /* Lua状态的复制 */
	  struct BlockCnt *bl;  /* 当前块链 */
	  int pc;  /* 代码的下一个位置（等价于ncode） */
	  int lasttarget;   /* 上一次jump target的pc */
	  int jpc;  /* 即将跳转的pc列表 */
	  int freereg;  /* 第一个空闲的寄存器 */
	  int nk;  /* k内的元素个数 */
	  int np;  /* p内的元素个数 */
	  short nlocvars;  /* locvars内的元素个数 */
	  lu_byte nactvar;  /*激活的局部变量的个数 */
	  upvaldesc upvalues[LUAI_MAXUPVALUES];  /* 上值 */
	  unsigned short actvar[LUAI_MAXVARS];  /* 已声明变量的堆栈 */
	} FuncState;
	4. typedef struct BlockCnt {
	  struct BlockCnt *previous;  /* 链 */
	  int breaklist;  /* 跳出此循环的列表 */
	  lu_byte nactvar;  /* 在可跳出结构外的激活局部变量 */
	  lu_byte upval;  /* 如果块的一些变量为上值的话为真 */
	  lu_byte isbreakable;  /* 如果块为循环的话为真 */
	} BlockCnt;
	5. struct ConsControl {
	  expdesc v;  /* 上一次读入的列表项 */
	  expdesc *t;  /* 表描述符 */
	  int nh;  /* record元素的总数 */
	  int na;  /* 数组元素的总数 */
	  int tostore;  /* 将要被存储的数组元素的个数 */
	};
	6. static const struct {
	  lu_byte left;  /* 每个二元操作符的左优先级 */
	  lu_byte right; /* 右优先级 */
	} priority[] = {  /* ORDER OPR */
	   {6, 6}, {6, 6}, {7, 7}, {7, 7}, {7, 7},  /* `+' `-' `/' `%' */
	   {10, 9}, {5, 4},                 /* 乘幂和拼接 (右结合) */
	   {3, 3}, {3, 3},                  /* 相等和不等 */
	   {3, 3}, {3, 3}, {3, 3}, {3, 3},  /* 排序 */
	   {2, 2}, {1, 1}                   /* 逻辑（与或） */
	};
	7. struct LHS_assign {
	  struct LHS_assign *prev;
	  expdesc v;  /* 变量（全局，局部，上值，或索引） */
	};

四、宏
	1. #define lparser_c
	2. #define LUA_CORE
	3. #define hasmultret(k)		((k) == VCALL || (k) == VVARARG)
	4. #define getlocvar(fs, i)	((fs)->f->locvars[(fs)->actvar[i]])
	5. #define luaY_checklimit(fs,v,l,m)	if ((v)>(l)) errorlimit(fs,l,m)
	6. #define check_condition(ls,c,msg)	{ if (!(c)) luaX_syntaxerror(ls, msg); }
	7. #define new_localvarliteral(ls,v,n) \
		new_localvar(ls, luaX_newstring(ls, "" v, (sizeof(v)/sizeof(char))-1), n)
	8. #define leavelevel(ls)	((ls)->L->nCcalls--)
	9. #define UNARY_PRIORITY	8  /* 一元操作符的优先级 */
	
五、公共全局函数
	1. Proto *luaY_parser (lua_State *L, ZIO *z, Mbuffer *buff, const char *name) {

六、私有静态函数
	1. static void anchor_token (LexState *ls) {
	2. static void error_expected (LexState *ls, int token) {
	3. static void errorlimit (FuncState *fs, int limit, const char *what) {
	4. static int testnext (LexState *ls, int c) {
	5. static void check (LexState *ls, int c) {
	6. static void checknext (LexState *ls, int c) {
	7. static void check_match (LexState *ls, int what, int who, int where) {
	8. static TString *str_checkname (LexState *ls) {
	9. static void init_exp (expdesc *e, expkind k, int i) {
	10. static void codestring (LexState *ls, expdesc *e, TString *s) {
	11. static void checkname(LexState *ls, expdesc *e) {
	12. static int registerlocalvar (LexState *ls, TString *varname) {
	13. static void new_localvar (LexState *ls, TString *name, int n) {
	14. static void adjustlocalvars (LexState *ls, int nvars) {
	15. static void removevars (LexState *ls, int tolevel) {
	16. static int indexupvalue (FuncState *fs, TString *name, expdesc *v) {
	17. static int searchvar (FuncState *fs, TString *n) {
	18. static void markupval (FuncState *fs, int level) {
	19. static int singlevaraux (FuncState *fs, TString *n, expdesc *var, int base) {
	20. static void singlevar (LexState *ls, expdesc *var) {
	21. static void adjust_assign (LexState *ls, int nvars, int nexps, expdesc *e) {
	22. static void enterlevel (LexState *ls) {
	23. static void enterblock (FuncState *fs, BlockCnt *bl, lu_byte isbreakable) {
	24. static void leaveblock (FuncState *fs) {
	25. static void pushclosure (LexState *ls, FuncState *func, expdesc *v) {
	26. static void open_func (LexState *ls, FuncState *fs) {
	27. static void close_func (LexState *ls) {
	28. static void field (LexState *ls, expdesc *v) {
	29. static void yindex (LexState *ls, expdesc *v) {
	30. static void recfield (LexState *ls, struct ConsControl *cc) {
	31. static void closelistfield (FuncState *fs, struct ConsControl *cc) {
	32. static void lastlistfield (FuncState *fs, struct ConsControl *cc) {
	33. static void listfield (LexState *ls, struct ConsControl *cc) {
	34. static void constructor (LexState *ls, expdesc *t) {
	35. static void parlist (LexState *ls) {
	36. static void body (LexState *ls, expdesc *e, int needself, int line) {
	37. static int explist1 (LexState *ls, expdesc *v) {
	38. static void funcargs (LexState *ls, expdesc *f) {
	39. static void prefixexp (LexState *ls, expdesc *v) {
	40. static void primaryexp (LexState *ls, expdesc *v) {
	41. static void simpleexp (LexState *ls, expdesc *v) {
	42. static UnOpr getunopr (int op) {
	43. static BinOpr getbinopr (int op) {
	44. static BinOpr subexpr (LexState *ls, expdesc *v, unsigned int limit) {
	45. static void expr (LexState *ls, expdesc *v) {
	46. static int block_follow (int token) {
	47. static void block (LexState *ls) {
	48. static void check_conflict (LexState *ls, struct LHS_assign *lh, expdesc *v) {
	49. static void assignment (LexState *ls, struct LHS_assign *lh, int nvars) {
	50. static int cond (LexState *ls) {
	51. static void breakstat (LexState *ls) {
	52. static void whilestat (LexState *ls, int line) {
	53. static void repeatstat (LexState *ls, int line) {
	54. static int exp1 (LexState *ls) {
	55. static void forbody (LexState *ls, int base, int line, int nvars, int isnum) {
	56. static void fornum (LexState *ls, TString *varname, int line) {
	57. static void forlist (LexState *ls, TString *indexname) {
	58. static void forstat (LexState *ls, int line) {
	59. static int test_then_block (LexState *ls) {
	60. static void ifstat (LexState *ls, int line) {
	61. static void localfunc (LexState *ls) {
	62. static void localstat (LexState *ls) {
	63. static int funcname (LexState *ls, expdesc *v) {
	64. static void funcstat (LexState *ls, int line) {
	65. static void exprstat (LexState *ls) {
	66. static void retstat (LexState *ls) {
	67. static int statement (LexState *ls) {
	68. static void chunk (LexState *ls) {

七、值得观察的代码
	1. luaY_parser
		Lua语法分析器的入口
	2. chunk/block/expr/subexpr/constructor/explist1
		Lua语法中几个重要元素
	
（未完成，待修改）
