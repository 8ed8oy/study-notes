## Whitespace 空白符

1. Each file must end with a newline sequence.每个文件都必须以换行序列结尾。
    
2. Files may not contain horizontal tab characters. Use blanks only for indentation.文件不得包含水平制表符。仅使用空格进行缩进。
    
3. No line may contain trailing blanks.任何一行都不得包含尾随空格。
    
4. Do NOT put whitespace: 不要添加空格：
    1. Around the “<” and “>” within a generic type designation (“List", not "List ", or "List< Integer >").在泛型类型指定中的“<”和“>”周围（“List”，而非“List ”或“List< Integer >”）。
    2. After the prefix operators “!”, “–”, “++”, unary “-“, or unary “+”.在前缀运算符“!”、“–”、“++”、一元“-”或一元“+”之后。
    3. Before the tokens “;” or the suffix operators “–” and “++”.在标记“;”或后缀运算符“–”和“++”之前。
    4. After “(“ or before “)”. 在“(”之后或“)”之前。
    5. After “.” 在“.”之后
    6. When you are lonely. 当你独处时。
5. DO put whitespace: 请添加空格：
    1. After “;”, “,”, or type casts (e.g., “(String) x”, not “(String)x”).在“;”、“,”之后，或类型转换之后（例如，“(String) x”，而不是“(String)x”）。
    2. Around binary operators (e.g., “*”, “+”) and comparison operators.在二元运算符（例如“*”、“+”）和比较运算符周围。
    3. Around assignment operators (e.g., “=”, “+=”).在赋值运算符周围（例如，“=”、“+=”）。
    4. Around “?” and “:” in the ternary conditional operator (“x>0 ? x : -x”).在三元条件运算符（“x>0 ? x : -x”）中的“?”和“:”前后。
    5. Around the keywords “assert”, “catch”, “do”, “else”, “finally”, “for”, “if”, “return”, “synchronized”, “try”, and “while”.围绕“assert”“catch”“do”“else”“finally”“for”“if”“return”“synchronized”“try”和“while”这些关键词。
6. In general, break (insert newlines in) lines before an operator, as in通常，在运算符前换行（插入新行），如下所示
    
    ```
     ... + 20 * X 
         + Y;
    ```
    
7. Do not separate a method name from the “(“ in a method call with blanks. However, you may separate them with a newline followed by blanks (for indentation) on long lines.在方法调用中，不要用空格将方法名与“(”分隔开。不过，在长行中，你可以用一个换行符加空格（用于缩进）来分隔它们。

## Indentation 缩进

1. The basic indentation step is 4 spaces.基本缩进步骤为4个空格。
    
2. Indent code by the basic indentation step for each block level (blocks are generally enclosed in “{“ and “}”), as in对于每个块级（块通常用“{”和“}”括起来），按基本缩进步骤对代码进行缩进，如下所示
    
    ```
     if (x > 0) {
         r = -x;
     } else {
         r = x;
     }
    ```
    
3. Indent ‘case’ labels an indent past their enclosing ‘switch’, as in将“case”标签缩进至其包含的“switch”之外一个缩进单位，如下所示
    
    ```
     switch (op) {
         case '+':
             addOpnds(x, y);
             break;
         default:
             ERROR();
     }
    ```
    
4. Indent continued lines by the basic indentation step.用基本缩进步骤缩进续行。
    

## Braces 大括号

1. Use { } braces around the statements of all ‘if’, ‘while’, ‘do’, and ‘for’ statements.在所有“if”、“while”、“do”和“for”语句的表达式周围使用{ }花括号。
    
2. Place a “}” brace on the same line as a following “else”, “finally”, or “catch”, as in将“}”括号与后面的“else”、“finally”或“catch”放在同一行，如下所示
    
    ```
     if (x > 0) {
         y = -x;
     } else {
         y = x;
     }
    ```
    
3. Put the “{“ that opens a block at the end of a line. Generally, it goes at the end of the “if”, “for”, “while”, “switch”, “do”, method header, or class header that contains it. If line length forces it to the next line, do not indent it, and put it alone on the line.将开启代码块的“{”放在行尾。通常，它应放在包含它的“if”“for”“while”“switch”“do”、方法头或类头的末尾。如果行长度迫使它移到下一行，则不要缩进，且让它单独占一行。
    

## Comments 注释

1. Methods should have javadoc comments explaining the behavior, parameters (using @param tags or otherwise), and return type.方法应该有Java文档注释，用于解释其行为、参数（使用@param标签或其他方式）以及返回类型。
    
2. Methods that return non-void values must describe them in their Javadoc comment either with a “@return” tag or in a phrase in running text that contains the word “return”, “returning”, or “returns”.返回非void值的方法必须在其Javadoc注释中通过“@return”标签，或者在包含“return”“returning”或“returns”一词的连贯文本短语中对这些值进行描述。
    
3. Each Javadoc comment must start with a properly formed sentence, starting with a capital letter and ending with a period.每个Javadoc注释都必须以一个格式正确的句子开头，该句子首字母大写且以句号结尾。
    

## Names 名称

1. Names of static final constants must be in all capitals (e.g., RED, DEFAULT_NAME).静态 final 常量的名称必须全部大写（例如，RED、DEFAULT_NAME）。
    
2. Names of parameters, local variables, and methods must start with a lower-case letter, or consist of a single, upper-case letter.参数、局部变量和方法的名称必须以小写字母开头，或者由单个大写字母组成。
    
3. Names of types (classes), including type parameters, must start with a capital letter.类型（类）的名称，包括类型参数，必须以大写字母开头。
    
4. Names of packages must start with a lower-case letter.包名必须以小写字母开头。
    
5. Names of instance variables and non-final class (static) variables must start with either a lower-case letter or “_”.实例变量和非最终类（静态）变量的名称必须以小写字母或“_”开头。
    

## Imports 导入

1. Do not use ‘import PACKAGE._’, unless the package is java.lang.Math, java.lang.Double, or org.junit.Assert. ‘import static CLASS._’ is OK.除非包是java.lang.Math、java.lang.Double或org.junit.Assert，否则不要使用“import PACKAGE._”。“import static CLASS._”是可以的。
    
2. Do not import the same class or static member twice.不要重复导入同一个类或静态成员。
    
3. Do not import classes or members that you do not use.不要导入不使用的类或成员。
    

## Assorted Java Style Conventions 各种Java风格约定

1. Write array types with the “[]” after the element-type name, not after the declarator. Write “String[] names”, not “String names[]”.数组类型应在元素类型名称后加上“[]”来书写，而不是在声明符后。应写成“String[] names”，而不是“String names[]”。
    
2. Write any modifiers for methods, classes, or fields in the following order:按以下顺序编写方法、类或字段的任何修饰符：
    1. public, protected, or private. public、protected或private。
    2. abstract or static. 抽象的或静态的。
    3. final, transient, or volatile. final、transient或volatile。
    4. synchronized. 同步的
    5. native. 本地方法的
    6. strictfp. 严格浮点
3. Do not explicitly modify methods, fields, or classes where the modification is redundant:不要对修改是多余的方法、字段或类进行显式修改：
    1. Do not label methods in interfaces or annotations as “public” or “abstract”.不要将接口或注解中的方法标记为“public”或“abstract”。
    2. Do not label fields in interfaces or annotations as “static”, “public”, or “final”.不要将接口或注解中的字段标记为“static”“public”或“final”。
    3. Do not label methods in final classes as “final”.不要将最终类中的方法标记为“final”。
    4. Do not label nested interfaces “static”.不要将嵌套接口标记为“static”。
4. Do not use empty blocks (‘{ }’ with only whitespace or comments inside) for control statements. There is one exception: a catch block may consist solely of comments having the form不要为控制语句使用空块（仅包含空白或注释的“{ }”）。有一个例外情况：catch块可以仅包含以下形式的注释
    
    ```
      /* Ignore EXCEPTIONNAME. */
    ```
    
5. Avoid “magic numbers” in code by giving them symbolic names, as in通过为代码中的“魔术数字”赋予符号名称来避免使用它们，例如
    
    ```
     public static final MAX_SIZE = 100;
    ```
    
    Exceptions are the numerals -10 through 10, 0.5, -0.5, 0.25, -0.25.例外情况是数字-10到10、0.5、-0.5、0.25、-0.25。
    
6. Do not try to catch the exceptions Exception, RuntimeError, or Error.不要尝试捕获Exception、RuntimeError或Error异常。
    
7. Write “b” rather than “b == true” and “!b” rather than “b == false”.使用“b”而非“b == true”，使用“!b”而非“b == false”。
    
8. Replace 替换
    
    ```
     if (condition) {
         return true;
     } else {
         return false;
     }
    ```
    
    with just 仅用
    
    ```
     return condition;
    ```
    
9. Only static final fields of classes may be public. Other fields must be private or protected.类中只有静态最终字段可以是公共的。其他字段必须是私有的或受保护的。
    
10. Classes that have only static methods and fields must not have a public (or defaulted) constructor.只有静态方法和字段的类不得拥有公共（或默认的）构造函数。
    
11. Classes that have only private constructors must be declared “final”.只有私有构造函数的类必须声明为“final”。

## Avoiding Error-Prone Constructs 避免易出错的结构

1. If a class overrides “equals”, it must also override “hashCode”.如果一个类重写了“equals”方法，那么它也必须重写“hashCode”方法。
    
2. Local variables and parameters must not shadow field names. The preferred way to handle, e.g., getter/setter methods that simply control a field is to prefix the field name with “_”, as in局部变量和参数不得遮蔽字段名。处理（例如）仅用于控制某个字段的 getter/setter 方法时，首选方式是在字段名前加上“_”，如下所示：
    
    ```
     public double getWidth() {
         return _width;
     }
    
     public void setWidth(double width) {
         _width = width;
     }
    ```
    
3. Do not use nested assignments, such as “if ((x = next()) != null) …”. Although this can be useful in C, it is almost never necessary in Java.不要使用嵌套赋值，例如“if ((x = next()) != null) …”。虽然这在C语言中可能有用，但在Java中几乎毫无必要。
    
4. Include a “default” case in every “switch” statement.在每个“switch”语句中包含一个“default”情况。
    
5. End every arm of a “switch” statement either with a “break” statement or a comment of the form用“break”语句或以下形式的注释结束“switch”语句的每个分支
    
    ```
     /* fall through */
    ```
    
1. Do not compare String literals with “\==”. Write不要用“\==”比较字符串字面量。应写成
    
    ```
     if (x.equals("something"))
    ```
    
    and not 而不是
    
    ```
     if (x == "something")
    ```
    
    There are cases where you really want to use “\==”, but you are unlikely to encounter them in this class.在某些情况下，你确实会想要使用“\==”，但在这门课中你不太可能遇到这些情况。
    

## Limits 限制

1. No file may be longer than 2000 lines.任何文件的长度不得超过2000行。
    
2. No line may be longer than 100 characters.每行长度不得超过100个字符。
    
3. No method may be longer than 80 lines.任何方法的长度不得超过80行。
    
4. No method may have more than 8 parameters.任何方法都不能有超过8个参数。
    
5. Every file must contain exactly one outer class (nested classes are OK).每个文件必须恰好包含一个外部类（嵌套类是允许的）。