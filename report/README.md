2020-2021学年第2学期 实 验 报 告

![zucc](assets/zucc.png)

- 课程名称：编程语言原理与编译
- 实验项目：<u>MicroC</u>
- 专业班级：计算机1901
- 学生学号：31901031
- 学生姓名：章文韬
- 实验指导教师: 郭敏
- [编译原理期末大作业 (github)](https://github.com/Wesily-G/FreakingF-Microc)



## 简介

这是一个名为MicroC的编译原理大作业，主要基于microC和Yuby完成，通过对解释器和编译器的代码改进和开发，实现了部分C语言的语法(并改进了部分)。主要完成功能如下：



## 项目说明

### 结构

- `CLex.fsl`生成的`CLex.fs`词法分析器。

  + CLex 中定义了基本的关键字、标识符、常量、使用大写字母表示

    程序读到这个符号就会转换为我们定义的大写字母，然后就给 CPar 处理

- `CPar.fsy`生成的`CPar.fs`语法分析器。

  + CPar 文件分为两部分，每个部分之间通过 %% 分隔

  + 第一部分声明需要使用的变量(词元)，声明变量后还需要声明优先级

  + 第二部分定义语法规则(文法)

    包括 : statement ,expression ,function  ,main ,vardeclare   variabledescirbe ,type ,const这些基本元素

    表示识别到前面定义的这些大写字母组成的符号串后,怎么处理这些规则

- `Absyn.fs` 定义了抽象语法树

  定义了变量描述、函数和类型的构造方法

- `Comp.fs`将抽象语法树转化为栈式虚拟机

- `interp.fs`对抽象语法树进行语义分析

- `Machine.fs` 虚拟机指令定义

- `Machine.java` 执行虚拟机指令

+ `Machine.java`生成`Machine.class`虚拟机与`Machinetrace.class`堆栈追踪

测试集：测试程序放在test文件夹内



### 使用方法

#### 解释器

**运行编译解释器  interpc.exe **

```js
dotnet restore  interpc.fsproj   // 可选
dotnet clean  interpc.fsproj     // 可选
dotnet build -v n interpc.fsproj // 构建 ./bin/Debug/net5.0/interpc.exe
```

**执行解释器**

```
./bin/Debug/net5.0/interpc.exe test/test.c
dotnet run -p interpc.fsproj test/test.c
dotnet run -p interpc.fsproj test/test.c  //-g 显示token AST 等调试信息
```

#### 编译器

**生成汇编指令数字文件 .out (任选以下两文件之一)**

```js
dotnet restore  microc.fsproj // 可选
dotnet clean  microc.fsproj   // 可选
dotnet build  microc.fsproj   // 构建 ./bin/Debug/net5.0/microc.exe

dotnet run -p microc.fsproj example/ex1.c    // 执行编译器，编译 ex1.c，并输出  ex1.out 文件
dotnet run -p microc.fsproj -g example/ex1.c  // -g 查看调试信息
```

**虚拟机的构建与运行**

```java
javac -encoding UTF-8 Machine.java
java Machine ex9.out 3//直接显示结果    
java Machinetrace ex9.out 0//查看栈式虚拟机每一步的细节
```

需要添加-encoding UTF-8，否则会造成编码错误提示。



## 功能实现(解释器部分)

#### 1.2.变量初始化,运算符号前置+= -= *= /= %=

- 抽象语法树:

  ```F#
  | DecAndAssign of typ * string * expr
  | AssignPrim of string *access * expr      (* x+=e or  *p+=e or  a[e]+=e  *)
  ```

- 词法分析:

  ```F#
  | '='             { ASSIGN } 
  | "+="            { PLUSASSIGN }
  | "-="            { MINUSASSIGN }
  | "*="            { TIMESASSIGN }
  | "/="            { DIVASSIGN }
  | "%="            { MODASSIGN }
  ```

- 语法分析:

  ```F#
  | Vardec ASSIGN Expr SEMI StmtOrDecSeq { DecAndAssign (fst $1, snd $1, $3) :: $5}
  | Access PLUSASSIGN Expr              { AssignPrim("+=", $1, $3) }
  | Access MINUSASSIGN Expr             { AssignPrim("-=", $1, $3) }
  | Access TIMESASSIGN Expr             { AssignPrim("*=", $1, $3) }
  | Access DIVASSIGN Expr               { AssignPrim("/=", $1, $3) }
  | Access MODASSIGN Expr               { AssignPrim("%=", $1, $3) }
  ```

- 语义分析(解释器):

  ```F#
  and stmtordec stmtordec locEnv gloEnv store =
      match stmtordec with
      | Stmt stmt -> (locEnv, exec stmt locEnv gloEnv store)
      | Dec (typ, x) -> allocate (typ, x, None) locEnv store
      | DecAndAssign (typ, name, expr) -> allocate (typ, name, Some(fst (eval expr locEnv gloEnv store))) locEnv store

  | AssignPrim(ope, acc, e) ->
      //tmp左值，res右值，以tmp类型为准
      let (loc,store1) = access acc locEnv gloEnv store
      let tmp = getSto store1 loc.pointer
      let (res,store2) = eval e locEnv gloEnv store1
      let num = 
          match ope with
          | "+=" -> 
              match (tmp) with
              | INT i -> INT(tmp.int + res.int)
              | FLOAT i -> FLOAT(tmp.float + res.float)
              | DOUBLE i -> DOUBLE(tmp.double + res.double)
              | _ -> failwith ("wrong calu:+=")
          | "-=" -> 
              match (tmp) with
              | INT i -> INT(tmp.int - res.int)
              | FLOAT i -> FLOAT(tmp.float - res.float)
              | DOUBLE i -> DOUBLE(tmp.double - res.double)
              | _ -> failwith ("wrong calu:-=")
          | "*=" -> 
              match (tmp) with
              | INT i -> INT(tmp.int * res.int)
              | FLOAT i -> FLOAT(tmp.float * res.float)
              | DOUBLE i -> DOUBLE(tmp.double * res.double)
              | _ -> failwith ("wrong calu:*=")
          | "/=" -> 
              match (tmp) with
              | INT i -> INT(tmp.int / res.int)
              | FLOAT i -> FLOAT(tmp.float / res.float)
              | DOUBLE i -> DOUBLE(tmp.double / res.double)
              | _ -> failwith ("wrong calu:/=")
          | "%=" -> 
              match (tmp) with
              | INT i -> INT(tmp.int % res.int)
              | FLOAT i -> FLOAT(tmp.float % res.float)
              | DOUBLE i -> DOUBLE(tmp.double % res.double)
              | _ -> failwith ("wrong calu:%=")
          | _  -> failwith("unkown primitive " + ope)
      (num, setSto store2 loc.pointer num)      
  ```
  
- 测试样例test/test1_assignPlus.c:

  ```c
  void main() {
    int i = 3;
    i += 1;
    print(i);

    i -= 1;
    print(i);

    i *= 2;
    print(i);

    i = i/2;
    print(i);
  }

  ```
- 运行结果:

![test1_interp](image\test1_interp.png)

#### 3.++ --

- 抽象语法树:

  ```F#
  | PreInc of access                 (* ++i or ++a[e]               *)
  | PreDec of access                 (* --i or --a[e]               *) 
  ```

- 词法分析:

  ```F#
  | "++"            { PREINC }
  | "--"            { PREDEC }
  ```

- 语法分析:

  ```F#
  | PREINC Access                       { PreInc $2           }
  | PREDEC Access                       { PreDec $2           }
  | Access PREINC                       { PreInc $1           } 
  | Access PREDEC                       { PreDec $1           }
  ```

- 语义分析(解释器):

  ```F#
  | PreInc acc    -> 
          let (loc, store1) = access acc locEnv gloEnv store
          let (i1) = getSto store1 loc.pointer
          match (i1) with
                  | INT i -> 
                      let res = INT(i1.int + 1)
                      (i1, setSto store1 loc.pointer res)
                  | FLOAT i -> 
                      let res = FLOAT(i1.float + 1.0)
                      (i1, setSto store1 loc.pointer res)
                  | _ -> failwith ("wrong calu:PreInc")
  | PreDec acc    -> 
      let (loc, store1) = access acc locEnv gloEnv store
      let (i1) = getSto store1 loc.pointer
      match (i1) with
              | INT i -> 
                  let res = INT(i1.int - 1)
                  (i1, setSto store1 loc.pointer res)
              | FLOAT i -> 
                  let res = FLOAT(i1.float - 1.0)
                  (i1, setSto store1 loc.pointer res)
              | _ -> failwith ("wrong caculate:PreDec")
  ```
  
- 测试样例test/test2_preinc.c:

  ```c
  void main() {
      int n;
      n = 3;
      ++n;
      n++;
      print(n);
  }
  ```
- 运行结果:

![test2_interp](image\test2_interp.png)

#### 4.int,float,double,boolean,char;  string(未能运行)

- 抽象语法树:

  ```F#
  | CstI of int
  | CstF of float
  | CstB of bool
  | CstD of double
  | CstS of string
  | CstC of char
  ```

- 词法分析:

  ```F#
  | "int"     -> INT
  | "float" -> FLOAT
  | "double" -> DOUBLE
  | "string" -> STRING
  | "boolean" -> BOOLEAN
  | "true"    -> CSTBOOL 1
  | "false"   -> CSTBOOL 0
  //匹配规则
  | ['0'-'9']+      { CSTINT (System.Int32.Parse (lexemeAsString lexbuf)) }
  | ['0'-'9']+'.'['0'-'9']?       { CSTFLOAT (float (lexemeAsString lexbuf)) }
  | ['0'-'9']+'.'['0'-'9']*      { CSTDOUBLE (System.Double.Parse (lexemeAsString lexbuf)) }   //=
  | ['\'']['a'-'z''A'-'Z''0'-'9']['\'']            
                    { try let single = lexemeAsString lexbuf in CSTCHAR (System.Char.Parse(single.Substring(1, 1))) with ex -> failwith "Char literal error." }
  | ['a'-'z''A'-'Z']['a'-'z''A'-'Z''0'-'9']*
                    { keyword (lexemeAsString lexbuf) }
  ```

- 语法分析:

  ```F#
  ....(skip)
  //类型定义
  Type:
      INT                                 { TypI     }
    | CHAR                                { TypC     }
    | BOOLEAN                             { TypB     }  
    | STRING                              { TypS     }
    | FLOAT                               { TypF     }
    | DOUBLE                              { TypD     }
  ;
  ```

- 语义分析(解释器):

  ```F#
  type memData = 
    | INT of int
    | CHAR of char
    | POINTER of int
    | FLOAT of float
    | DOUBLE of double
    | STRING of string
    ....(skip)
  
  ```
  
- 测试样例test/test1_assignPlus.c:

  ```c
  void main() {

      int a = 3;
      print("%i",a);

      double b = 233.2234;
      print("%d",b);
      
      char c = 's';
      print("%c",c);
      
      float d = 13.6;
      print("%f",d);

      // string e = "sdfgw";
      // print("%s",e);

      boolean flag=true;
      if (flag) {
          print(1);
      }
      if (!flag) {
          print(0);
      }
    
  }

  ```
- 运行结果:

![test3](image\test3_interp.png)

#### 5.print格式化输出(上面比较多，就拆开了)

- 抽象语法树:

  ```F#
  | Print of string * expr
  ```

- 词法分析:

  ```F#
  | "print"   -> PRINT
  ```

- 语法分析:

  ```F#
  | PRINT LPAR CSTSTRING COMMA Expr RPAR{ Print($3, $5)       }
  ```

- 语义分析(解释器):

  ```F#
  | Print(op,e1)   -> 
      let (i1, store1) = eval e1 locEnv gloEnv store
      let res = 
          match op with
              | "%c"   -> (printf "%c " i1.char; i1)
              | "%i"   -> (printf "%d " i1.int ; i1)  
              | "%f"   -> (printf "%f " i1.float ;i1)
              | "%s"   -> (printf "%s " i1.string ;i1)
              | "%d"   -> (printf "%f " i1.double ;i1)
      (res, store1)  
  ```
  
- 测试样例test/test3_newType.c:

- 运行结果:同上

#### 6.三目运算

- 抽象语法树:

  ```F#
  | Prim3 of expr * expr * expr      (*  ? ：                        *)
  ```

- 词法分析:

  ```F#
  | ':'             { COLON }
  | '?'             { QUEST }
  ```

- 语法分析:

  ```F#
  | Expr QUEST Expr COLON Expr          { Prim3($1,$3,$5)     }
  ```

- 语义分析(解释器):

  ```F#
  | Prim3(e1, e2, e3) ->
      let (v, store1) = eval e1 locEnv gloEnv store
      if v <> INT(0) then eval e2 locEnv gloEnv store1
      else eval e3 locEnv gloEnv store1
  ```
  
- 测试样例test/test4_threeUnary.c:

  ```c
  void main(){
      int i=2;
      (i > 1) ? print(1):print(2);
  }
  ```
- 运行结果:

![test4_interp](image\test4_interp.png)

#### 7.switch

- 抽象语法树:

  ```F#
  | Switch of expr * stmt list
  | Case of expr * stmt
  | Default of stmt 
  ```

- 词法分析:

  ```F#
  | "switch"  -> SWITCH
  | "case"    -> CASE
  | "default" -> DEFAULT 
  ```

- 语法分析:

  ```F#
  | SWITCH LPAR Expr RPAR LBRACE CaseList RBRACE { Switch($3, $6) }

  CaseList:
                                          { [] }
    | CaseDec                             { [$1]                 }
    | CaseDec CaseList                    { $1 :: $2             }
    | DEFAULT COLON Stmt                  { [Default($3)]        }
  CaseDec:
    CASE Expr COLON Stmt                { Case($2,$4)          }
  ```

- 语义分析(解释器):

  ```F#
  | Switch(e, body) ->
      let (v, store0) = eval e locEnv gloEnv store
      //递归调用carry列表（case?: 或 default:）
      let rec carry list = 
          match list with
          | Case(e1, body1) :: next -> 
              let (v1, store1) = eval e1 locEnv gloEnv store0
              if v1 = v then exec body1 locEnv gloEnv store1
              else carry next
          | Default(body) :: over ->
              exec body locEnv gloEnv store0
          | [] -> store0
          | _ -> store0
      (carry body)
  //两个switch的辅助函数,实际上是差不多的,只不过case有个判断
  | Case (e, body) -> exec body locEnv gloEnv store
  | Default(body) -> exec body locEnv gloEnv store
  ```
  
- 测试样例test/test5_switch.c:

  ```c
  void main(){
      int i=2;
      (i > 1) ? print(1):print(2);
  }
  ```
- 运行结果:

![test5_interp](image\test5_interp.png)

#### 8.for,for in range

- 抽象语法树:

  ```F#
  | ForIn of access * expr * expr * expr * stmt 
  | For of expr * expr * expr * stmt
  ```

- 词法分析:

  ```F#
  | "for"     -> FOR
  | "in"      -> IN
  | "range"   -> RANGE  
  ```

- 语法分析:

  ```F#
  | FOR LPAR Expr SEMI Expr SEMI Expr RPAR StmtM { For($3, $5, $7, $9) }
  | FOR Access IN RANGE LPAR Expr COMMA Expr COMMA Expr RPAR StmtM {ForIn($2, $6, $8, $10, $12)}
  ```

- 语义分析(解释器):

  ```F#
  | For(e1, e2, e3, body) -> 
    //初值和变量环境                   
    let (v, store1) = eval e1 locEnv gloEnv store
    //定义循环循环判断，e2是判断数
    let rec loop store1 =
        let (v, store2) = eval e2 locEnv gloEnv store1
        //定义循环动作，e3是循环的加数
        if v<>INT(0) then loop (snd (eval e3 locEnv gloEnv (exec body locEnv gloEnv store2)))
        else store2
    loop store1
      // for var in range(e1,e2,e3){body}类似于python的语法
  | ForIn (var, e1, e2, e3, body) ->
    //获取上述的四个值
    let (local_var, store1) = access var locEnv gloEnv store
    let (start_num, store2) = eval e1 locEnv gloEnv store1
    let (end_num, store3) = eval e2 locEnv gloEnv store2
    let (step, store4) = eval e3 locEnv gloEnv store3
    //定义回调循环函数
    let rec loop temp store5 =
        let store_local =
            exec body locEnv gloEnv (setSto store5 local_var.pointer temp)
        //循环条件
        if temp.int + step.int < end_num.int then
            let nextValue = INT(temp.int + step.int)
            loop nextValue store_local
        else
            store_local
    //校验一下e1,e2
    if start_num.int < end_num.int then
        let intValue = INT(start_num.int)
        loop intValue store4
    else
        store4
  ```
  
- 测试样例test/test6_forin.c:

  ```c
  void main() {
      int i;
      for (i = 0; i < 5; i = i + 1)
      {
          print(i);
      }
      println;
      int t;
      for t in range (2,7,2)
      {
          print(t);
      }
  }
  ```
- 运行结果(输出较多，所以给出不带调试运行的结果):
![test6_interp_2](image\test6_interp_2.png)
![test6_interp](image\test6_interp.png)



#### 9. do while,do until

- 抽象语法树:

  ```F#
  | DoUntil of stmt * expr
  | DoWhile of stmt * expr
  ```

- 词法分析:

  ```F#
  | "do"      -> DO 
  | "until"   -> UNTIL
  | "while"   -> WHILE
  ```

- 语法分析:

  ```F#
  | DO StmtM WHILE LPAR Expr RPAR SEMI  { DoWhile($2, $5)      }
  | DO StmtM UNTIL LPAR Expr RPAR SEMI  { DoUntil($2, $5)      }
  ```

- 语义分析(解释器):

  ```F#
  | DoWhile (body, e) ->
      //与while类似，递归调用,实际上就是body和e换了个位置
      let rec loop store1 =
          let (v, store2) = eval e locEnv gloEnv store1
          if v <> INT(0) then
              loop (exec body locEnv gloEnv store2)
          else
              store2
      loop (exec body locEnv gloEnv store)
  | DoUntil (body, e) -> 
      //与dowhile类似但终止条件不一样
      let rec loop store1 =
          let (v, store2) = eval e locEnv gloEnv store1
          if v = INT(0) then 
              loop (exec body locEnv gloEnv store2)
          else 
              store2    
      loop (exec body locEnv gloEnv store)
  ```
  
- 测试样例test/test7_doWhileUntil.c:

  ```c
  void main() {
      int i=2;
      do{
          i+=1;
      }while(i<3);
      print(i);
      do{
          i-=1;
      }until(i<3);
      print(i);
  }
  ```
- 运行结果:
![test7_interp_2](image\test7_interp_2.png)
![test7_interp](image\test7_interp.png)

## 功能实现(编译器部分 测试样例相同,只给出文件名)

#### 1.++--

- 编译器
  ```F#
  and cStmtOrDec stmtOrDec (varEnv: VarEnv) (funEnv: FunEnv) : VarEnv * instr list =
    match stmtOrDec with
    | Stmt stmt -> (varEnv, cStmt stmt varEnv funEnv)
    | Dec (typ, x) -> allocateWithMsg Locvar (typ, x) varEnv
    | DecAndAssign (typ, x, e) -> //定义时赋值
        let (varEnv, code) = allocate Locvar (typ, x) varEnv

        (varEnv,
         code
         @ (cExpr (Assign((AccVar x), e)) varEnv funEnv)
           @ [ INCSP -1 ])
  ```
- 堆栈图(test2_preinc.c)
![image-20210627235747819](assets/image-20210627235747819.png)



**代码提交日志**

![image-20210627235747819](assets/image-20210627235747819.png)



## 技术评价

+ example

  |                  功能                   |  对应测试文件  |  优  |  良  |  中  |
  | :-------------------------------------: | :------------: | :--: | :--: | :--: |
  |               变量初始化                |    Assign.c    |      |  √   |      |
  |               自增、自减                |     Pre.c      |  √   |      |      |
  |                 for循环                 |     for.c      |  √   |      |      |
  |               三目运算符                |    prim3.c     |  √   |      |      |
  |               switch-case               |    switch.c    |  √   |      |      |
  | float、double、char、long类型（编译器） |     type.c     |  √   |      |      |
  |            dowhile、dountil             | dowhileuntil.c |  √   |      |      |
  |            +=  -=  *=  /= %=            |     Add.c      |  √   |      |      |
  |                 boolean                 |   boolean.c    |  √   |      |      |




## 心得体会




