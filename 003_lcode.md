Lua5.1代码阅读（三）：lcode.h/lcode.c

（未完成，待修改）

一、概述
	lcode.h/lcode.c是Lua的代码生成器，
	用于优化和生成目标二进制代码。
	lcode.c的所有导出函数只被lparser.c引用。
	lcode内部的函数引用图如下：
	
二、宏
	1. #define NO_JUMP (-1)
	2. #define getcode(fs,e)	((fs)->f->code[(e)->u.s.info])
	3. #define luaK_codeAsBx(fs,o,A,sBx)	luaK_codeABx(fs,o,A,(sBx)+MAXARG_sBx)
	4. #define luaK_setmultret(fs,e)	luaK_setreturns(fs, e, LUA_MULTRET)
	5. #define lcode_c
	6. #define LUA_CORE
	7. #define hasjumps(e)	((e)->t != (e)->f)

三、枚举值
	1. typedef enum BinOpr {
	  OPR_ADD, OPR_SUB, OPR_MUL, OPR_DIV, OPR_MOD, OPR_POW,
	  OPR_CONCAT,
	  OPR_NE, OPR_EQ,
	  OPR_LT, OPR_LE, OPR_GT, OPR_GE,
	  OPR_AND, OPR_OR,
	  OPR_NOBINOPR
	} BinOpr;
	2. typedef enum UnOpr { OPR_MINUS, OPR_NOT, OPR_LEN, OPR_NOUNOPR } UnOpr;

四、私有静态函数
	1. static int isnumeral(expdesc *e) {
	2. static int condjump (FuncState *fs, OpCode op, int A, int B, int C) {
	3. static void fixjump (FuncState *fs, int pc, int dest) {
	4. static int getjump (FuncState *fs, int pc) {
	5. static Instruction *getjumpcontrol (FuncState *fs, int pc) {
	6. static int need_value (FuncState *fs, int list) {
	7. static int patchtestreg (FuncState *fs, int node, int reg) {
	8. static void removevalues (FuncState *fs, int list) {
	9. static void patchlistaux (FuncState *fs, int list, int vtarget, int reg, int dtarget) {
	10. static void dischargejpc (FuncState *fs) {
	11. static void freereg (FuncState *fs, int reg) {
	12. static void freeexp (FuncState *fs, expdesc *e) {
	13. static int addk (FuncState *fs, TValue *k, TValue *v) {
	14. static int boolK (FuncState *fs, int b) {
	15. static int nilK (FuncState *fs) {
	16. static int code_label (FuncState *fs, int A, int b, int jump) {
	17. static void discharge2reg (FuncState *fs, expdesc *e, int reg) {
	18. static void discharge2anyreg (FuncState *fs, expdesc *e) {
	19. static void exp2reg (FuncState *fs, expdesc *e, int reg) {
	20. static void invertjump (FuncState *fs, expdesc *e) {
	21. static int jumponcond (FuncState *fs, expdesc *e, int cond) {
	22. static void luaK_goiffalse (FuncState *fs, expdesc *e) {
	23. static void codenot (FuncState *fs, expdesc *e) {
	24. static int constfolding (OpCode op, expdesc *e1, expdesc *e2) {
	25. static void codearith (FuncState *fs, OpCode op, expdesc *e1, expdesc *e2) {
	26. static void codecomp (FuncState *fs, OpCode op, int cond, expdesc *e1, expdesc *e2) {
	27. static int luaK_code (FuncState *fs, Instruction i, int line) {

五、公共全局函数
	1. void luaK_nil (FuncState *fs, int from, int n) {
	2. int luaK_jump (FuncState *fs) {
	3. void luaK_ret (FuncState *fs, int first, int nret) {
	4. int luaK_getlabel (FuncState *fs) {
	5. void luaK_patchlist (FuncState *fs, int list, int target) {
	6. void luaK_patchtohere (FuncState *fs, int list) {
	7. void luaK_concat (FuncState *fs, int *l1, int l2) {
	8. void luaK_checkstack (FuncState *fs, int n) {
	9. void luaK_reserveregs (FuncState *fs, int n) {
	10. int luaK_stringK (FuncState *fs, TString *s) {
	11. int luaK_numberK (FuncState *fs, lua_Number r) {
	12. void luaK_setreturns (FuncState *fs, expdesc *e, int nresults) {
	13. void luaK_setoneret (FuncState *fs, expdesc *e) {
	14. void luaK_dischargevars (FuncState *fs, expdesc *e) {
	15. void luaK_exp2nextreg (FuncState *fs, expdesc *e) {
	16. int luaK_exp2anyreg (FuncState *fs, expdesc *e) {
	17. void luaK_exp2val (FuncState *fs, expdesc *e) {
	18. int luaK_exp2RK (FuncState *fs, expdesc *e) {
	19. void luaK_storevar (FuncState *fs, expdesc *var, expdesc *ex) {
	20. void luaK_self (FuncState *fs, expdesc *e, expdesc *key) {
	21. void luaK_goiftrue (FuncState *fs, expdesc *e) {
	22. void luaK_indexed (FuncState *fs, expdesc *t, expdesc *k) {
	23. void luaK_prefix (FuncState *fs, UnOpr op, expdesc *e) {
	24. void luaK_infix (FuncState *fs, BinOpr op, expdesc *v) {
	25. void luaK_posfix (FuncState *fs, BinOpr op, expdesc *e1, expdesc *e2) {
	26. void luaK_fixline (FuncState *fs, int line) {
	27. int luaK_codeABC (FuncState *fs, OpCode o, int a, int b, int c) {
	28. int luaK_codeABx (FuncState *fs, OpCode o, int a, unsigned int bc) {
	29. void luaK_setlist (FuncState *fs, int base, int nelems, int tostore) {

六、值得观察的代码
	1. luaK_code
		可以在此处设置断点，
		观察Lua编译器是何时生成Instruction代码，
		以及如何将其保存到Proto（函数原型）结构体的code数组中。
	2. TODO:
		大多数公共全局函数或全局宏（luaK_*）都有生成OP_*指令的能力。

（未完成，待修改）
