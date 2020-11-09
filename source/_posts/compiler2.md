---
title: 编译器 - 词法分析
date: 2020-09-08 09:27:33
tags: compile
categories: compile
---

## 定义
词法分析器的功能：输入源程序，按照构词规则分解成一系列单词符号。单词是语言中具有独立意义的最小单位，包括关键字、标识符、运算符、界符和常量等

+ 关键字 是由程序语言定义的具有固定意义的标识符。例如，Java 中的if，while，for都是保留字
+ 标识符 用来表示各种名字，如变量名，数组名，过程名等等
+ 常数  常数的类型一般有整型、实型、布尔型、文字型等
+ 运算符 如+、-、*、/等等
+ 界符  如逗号、分号、括号、等等

理论的知识还是比较复杂的，有状态转换图(FA)、正规式与正规集等，我的编译器系列文章是从开发人员的角度倒着来学习，先把算法过程捋一遍，然后再回过头去看理论知识，这样更容易理解理论说的是什么。

## 思路

```
这边的算法是从go中text/scanner包中的Scanner移植过来的，去掉了unicode的支持，做了一点简化。
我们主要是为了理解整个过程，所以有些严禁的东西都去掉了。
```

一. 简介

+ 整个算法用了两个缓存区（源数据缓冲区和词法单元临时缓冲区）
+ 三个指针，分别指向当前源数据缓存区位置和词法单元的开始位置及结束位置
+ 当前读取的字符的变量
+ 当前读取的行和列变量
+ 判断数字、字符、标识符的方法
+ 处理数字、字符串、标识符、注释的方法

二. 文法

1. 字符
```
正则表达式: a-z | A-Z
```

2. 数字
```
正则表达式: 0-9
```

3. 标识符
```
以字符(a-zA-Z)或下划线(_)开头，后面只包含任意个字符、数字、下划线(a-zA-Z0-9_)的字符串
正则表达式: [a-zA-Z_]+[a-zA-Z0-9_]*
```

4. 数值

	+ 整数或浮点数
	```
	以数字(0-9)开头，后面只包含一个小数点(.)和其他数字(0-9)的字符串
	正则表达式: [0-9]+\.?[0-9]*
	```

	+ 浮点数
	```
	以小数点(.)开头，后面只包含其他数字的字符串
	正则表达式: \.[0-9]+
	```

4. 字符串
	+ 单引号
	```
	以单引号(')开头,直到找到下一个单引号结束
	正则表达式: '.*'
	```
	+ 双引号
	```
	以双引号(")开头,直到找到下一个双引号结束
	正则表达式: ".*"
	```

5. 注释
	+ 单行注释
	```
	以双反斜杠(//)开头,直到当前行结束
	正则表达式: //.*
	```

	+ 多行注释或局部注释
	```
	以反斜杠(/)和星号(*)开头，直到找到下一个反斜杠和星号结束
	正则表达式: /\*.*\*/
	```

三. 算法

1. 大体思路
首先获取整个字符流实例，读取到源数据缓存中，然后从源数据缓存中读取一个字符，判断是否符合文法规则和是否继续读取下一个字符，还是退出当前规则，重新开始。反复以上的过程直到字符流中的数据读完，算法很简单。

2. 详细过程
先用文字给说明一下，再对照代码就明白了
```golang
定义变量： 
	字符流实例
	//字符数组，长度为定义长度+1，为什么要+1后面解释
	//这里我们默认长度可以是1024，那数组的长度就是1024+1
	//当然也可以是其他长度2048+1等
	源数据缓存
	源数据缓存指针
	//动态字符数组，不确定长度
	词法单元暂存缓存
	词法单元指针开始，词法单元指针结束
	当前行，列
	当前读取的字符

初始化：
	1.  打开文件或字符串到字符流实例
	//读取字符的时候如果读到0x80表示缓冲区没有可读字符或已经读完
	//需要从字符流中读取未读的字符
	2.  源数据缓存第一个字符为0x80
	3.  源数据缓存指针为0
	//只要有数据开始行肯定是1，当前列为0
	4.  当前行为1
	5.  当前列为0
	//默认没有读取任何字符，用-2来标记，后面用来判断
	6.  当前读取的字符为-2
	//词法单元暂存缓存也没有数据，指针开始置位-1，结束位置为0
	7.  词法单元指针开始为-1
	8.  词法单元指针结束0

读取字符方法：
	1.  根据源数据缓存指针从源数据缓存中读取一个字符
	2.  如果读取的字符是0x80
		//这边有种情况
		//如果前面的处理正好符合某个规则，但是缓存中由于
		//长度的关系没有把完整的字符流中的数据读取进来，
		//所以必须先把之前处理的正确数据存起来
		//这也是这个词法单元暂存缓存的作用
		3.  如果词法单元指针开始大于等于0
			4.  把源数据缓存中词法单元指针开始到当前源数据缓存指针的数据存到词法单元暂存缓存
			//后面要重新读，长度位置都会变，所以重置
			5.  重置词法单元指针为0
		//表示还没有从字符流读取数据或源数据缓存已经读完
		6.  读取最多源数据缓存定义长度(1024)个字符到源数据缓存
		//因为初始化的时候预留的1个字节
		//所以即使读取的数据长度为1024我们也可以在1025的位置定义标记0x80
		//为什么要标记0x80原因上面解释
		//如果读取的长度小于1024则只要在读取数据的后面加上0x80
		7.  源数据缓存中数据的后面一个字节设置为0x80
		//重新读取过数据所以源数据缓存指针必须重置
		8.  重置源数据缓存指针为0
		9.  如果字符流读取错误
			10. 如果数据读完出错返回EOF
			11. 否则输出错误信息
		12.  重新根据源数据缓存指针从源数据缓存的位置读取一个字符，也就是第0个字符
	//当前位置已经读过，指针后移，当前列增加
	13.  缓存指针的位置+1
	14.  当前列+1
	15.  如果读取的字符为换行
		//表示新的一行开始
		16. 当前行+1
		17. 当前列0
	18. 返回读取的字符

开始扫描：
	1.  获取当前读取的字符
	//重新扫描开始，词法单元指针为-1
	//保证在调用(读取字符方法)不会从词法单元暂存缓存获取数据
	2.  重置词法单元指针开始为-1
	3.  如果获取的字符是-2
		//表示刚开始读
		4. 调用(读取字符方法)重新获取
	5.  如果获取的字符是制表符(\t)、回车(\r)、换行(\n)、空字符串(' ')
		6. 调用(读取字符方法)重新获取，直到不满足规则
	//读到了字符，因为获取字符以后指针往后移了，那当前字符的位置就要-1
	//最后我们会根据开始位置和结束位置来获取当前获取的词法单元内容
	7.  重置词法单元指针开始为源数据缓存指针-1
	8.  清空词法单元暂存缓存
	//因为有可能我们分析出来的字符在我们的规则中不存在
	//但是也必须要返回，所以这里暂时先这么设置
	9.  定义词法单元类型为获取的获取的字符
	10. 判断获取的字符
		11. 如果是EOF
			//源数据缓存已经处理完，同时字符流也没有数据，直接返回
			12. 返回
		13.  如果是字符
			14. 词法单元类型为标识符类型
			15. 调用(标识符处理方法)开始处理
			16. 更新获取的字符为返回处理完后返回的字符
		17. 如果是数字
			18. 调用(数值处理方法)开始处理
			19. 更新获取的字符为返回处理完后返回的字符
			20. 根据返回更新词法单元类型为整型或浮点型
		21. 如果是单引号(‘)或双引号(“)
			22. 词法单元类型为字符串类型
			23. 字符串处理方法开始处理
			24. 调用(读取字符方法)再次获取
		25. 如果是小数点(.)
			//因为不确定后面是不是数字，所以再获取一个
			26. 调用(读取字符方法)再次获取
			27. 如果是数字
				//因为有小数点一定是浮点数
				28. 词法单元类型为浮点型
				29. 调用(浮点数处理方法)开始处理
				30. 更新获取的字符为返回处理完后返回的字符
		31. 如果是反斜杠(/)
			// 因为不确定是不是注释，所以再获取一个
			32. 调用(读取字符方法)再次获取
			33. 如果是/或*
				34. 词法单元类型为注释
				35. 调用(注释处理方法)开始处理
				36. 更新获取的字符为返回处理完后返回的字符
		37. 如果都没有匹配到
			38. 调用(读取字符方法)更新获取的字符
	//原因通词法单元指针开始
	39. 词法单元指针结束为源数据缓存指针-1
	40. 更新当前读取的字符为获取的字符
	41. 返回词法单元类型

文法处理方法：这边就不一一写出来了，后面代码中会有，比较简单

```

## 图解

## 代码
代码用的是GO

1. 代码
		
	```golang
	package lexer

	import (
		"bytes"
		"fmt"
		"io"
		"os"
	)

	// 缓存长度
	const bufLen = 1024

	// 标识符
	const readFlag = 0x80

	// 词法单元类型
	const (
		EOF = 1 << iota
		Ident
		Int
		Float
		String
		Comment
	)

	var tokenString = map[rune]string{
		EOF:     "EOF",
		Ident:   "Ident",
		Int:     "Int",
		Float:   "Float",
		String:  "String",
		Comment: "Comment",
	}

	// TokenString 词法单元类型
	func TokenString(token rune) string {
		if s, found := tokenString[token]; found {
			return s
		}
		return fmt.Sprintf("%q", string(token))
	}

	// Lexer 词法分析器
	type Lexer struct {

		// 数据源
		src io.Reader
		// 数据缓存区(bufLen字符),多一位为了下一次读取判断
		srcBuf [bufLen + 1]byte
		// 当前读取位置
		srcPos int

		// 词法临时缓存区
		tokenBuf bytes.Buffer
		// 词法单元开始位置
		tokenPos int
		// 词法单元结束位置
		tokenEnd int

		// 当前读取的字符
		ch rune

		// 当前行
		line int
		// 当前列
		column int

		Filename string
	}


	// 判断字符
	func isLetter(ch rune) bool {
		return (ch >= 'a' && ch <= 'z') || (ch >= 'A' && ch <= 'Z')
	}

	// 判断数字
	func isDecimal(ch rune) bool {
		return ch >= '0' && ch <= '9'
	}

	// 判断标识符
	func isIdentRune(ch rune, i int) bool {
		return ch == '_' || isLetter(ch) || (isDecimal(ch) && i > 0)
	}

	// 扫描标识符
	func (lexer *Lexer) scanIdentifier() rune {
		ch := lexer.next()
		for i := 1; isIdentRune(ch, i); i++ {
			ch = lexer.next()
		}
		return ch
	}

	// 扫描字符串
	func (lexer *Lexer) scanString(quote rune) {
		ch := lexer.next()
		for ch != quote {
			if ch == '\n' || ch < 0 {
				lexer.error("literal not terminated")
				return
			}
			ch = lexer.next()
		}
	}


	// 扫描数字
	func (lexer *Lexer) scanNumber(ch rune, seenDot bool) (rune, rune) {
		var tk rune
		if !seenDot {
			tk = Int
			for isDecimal(ch) {
				ch = lexer.next()
			}
			if ch == '.' {
				ch = lexer.next()
				seenDot = true
			}
		}

		// 处理小数部分
		if seenDot {
			tk = Float
			for isDecimal(ch) {
				ch = lexer.next()
			}
		}
		return tk, ch
	}

	// 扫描注释
	func (lexer *Lexer) scanComment(ch rune) rune {
		if ch == '/' {
			// 单行注释
			ch = lexer.next()
			for ch != '\n' && ch >= 0 {
				ch = lexer.next()
			}
			return ch
		}

		// 非单行注释
		ch = lexer.next()
		for {
			if ch < 0 {
				lexer.error("comment not terminated")
				break
			}
			ch0 := ch
			ch = lexer.next()
			if ch0 == '*' && ch == '/' {
				ch = lexer.next()
				break
			}
		}
		return ch
	}

	func (lexer *Lexer) peek() rune {
		if lexer.ch == -2 {
			lexer.ch = lexer.next()
			if lexer.ch == '\uFEFF' {
				lexer.ch = lexer.next() // ignore BOM
			}
		}
		return lexer.ch
	}

	// 读取下一个字符
	func (lexer *Lexer) next() rune {
		ch := rune(lexer.srcBuf[lexer.srcPos])
		// 如果是读取标记,则进行读取
		if ch >= readFlag {

			// 字符处理一半,缓存区已满，使用词法缓存暂存
			if lexer.tokenPos >= 0 {
				lexer.tokenBuf.Write(
					lexer.srcBuf[lexer.tokenPos:lexer.srcPos]
				)
				lexer.tokenPos = 0
			}

			// 从缓冲区中继续读取数据
			n, err := lexer.src.Read(lexer.srcBuf[0:bufLen])
			// 最后一位设置为下一次继续读取的标记
			lexer.srcBuf[n] = readFlag
			// 重置当前读取位置
			lexer.srcPos = 0
			if err != nil {
				// 其他错误则打印
				if err != io.EOF {
					lexer.error(err.Error())
				}
				// 如果已经读完返回End Of File
				return EOF
			}
			// 重新读取第一次字符
			ch = rune(lexer.srcBuf[lexer.srcPos])
		}
		lexer.srcPos++
		lexer.column++
		switch {
		case ch == '\n':
			lexer.line++
			lexer.column = 0
		}
		return ch
	}

	// 错误
	func (lexer *Lexer) error(msg string) {
		fmt.Fprintf(os.Stderr, "%s: %s\n", lexer.Position(), msg)
	}

	// Init 初始化
	func (lexer *Lexer) Init(src io.Reader) {
		lexer.src = src

		lexer.line = 1
		lexer.column = 0

		lexer.srcBuf[0] = readFlag
		lexer.srcPos = 0

		lexer.tokenPos = -1
		lexer.tokenEnd = 0

		lexer.ch = -2

		lexer.Filename = "nofile"
	}

	// Scan 扫描
	func (lexer *Lexer) Scan() rune {

		// 获取字符
		ch := lexer.peek()

		lexer.tokenPos = -1

		// 忽略 制表符 回车换行 空字符串
		for ch == '\t' || ch == '\n' || ch == '\r' || ch == ' ' {
			ch = lexer.next()
		}

		// 清空词法缓存
		lexer.tokenBuf.Reset()

		// 设置词法单元开始位置,因为next()中pos++，
		// 所以开始位置在缓存区中的位置往前移一格
		lexer.tokenPos = lexer.srcPos - 1

		tk := ch
		switch {
		case isIdentRune(ch, 0):
			// 标识符
			tk = Ident
			ch = lexer.scanIdentifier()
		case isDecimal(ch):
			// 数值型,要判断是整数还是浮点数
			tk, ch = lexer.scanNumber(ch, false)
		default:
			switch ch {
			case EOF:
				break
			case '"', '\'':
				lexer.scanString(ch)
				tk = String
				ch = lexer.next()
			case '/':
				ch = lexer.next()
				if ch == '/' || ch == '*' {
					ch = lexer.scanComment(ch)
					tk = Comment
				}
			case '.':
				ch = lexer.next()
				if isDecimal(ch) {
					tk, ch = lexer.scanNumber(ch, true)
				}
			default:
				ch = lexer.next()
			}
		}

		// 同样多读取了一个，所以把词法单位结束位置在缓存区中的位置往前移一格
		lexer.tokenEnd = lexer.srcPos - 1
		lexer.ch = ch
		return tk
	}

	// Position 当前文件位置
	func (lexer *Lexer) Position() string {
		var line int
		if lexer.column > 0 {
			line = lexer.line
		} else {
			// 如果读到\n，因为读取的时候已经加1了，所以这边减去1
			line = lexer.line - 1
		}
		return fmt.Sprintf("%s:%d:%d", lexer.Filename, line, lexer.column)
	}

	// Token 词法单元
	func (lexer *Lexer) Token() string {
		if lexer.tokenPos < 0 {
			return ""
		}

		if lexer.tokenEnd < lexer.tokenPos {
			lexer.tokenEnd = lexer.tokenPos
		}

		// 词法单元缓存区为空，直接返回当前词法开始位置和结束位置的字符
		if lexer.tokenBuf.Len() == 0 {
			return string(lexer.srcBuf[lexer.tokenPos:lexer.tokenEnd])
		}

		// 把后面读取的缓冲区中的数据加入到词法单元的缓存区形成完整的字符
		lexer.tokenBuf.Write(lexer.srcBuf[lexer.tokenPos:lexer.tokenEnd])
		return lexer.tokenBuf.String()
	}

	```
代码基本和上面的文字描述一致

2. 测试
	```golang

	func Test1() {
		const src = `
		// Comment begins at column 5.

	This line should not be included in the output.

	/*
	This multiline comment
	should be extracted in
	its entirety.
	*/
	`

		var lexer Lexer
		lexer.Init(strings.NewReader(src))
		for tok := lexer.Scan(); tok != EOF; tok = lexer.Scan() {
			fmt.Printf("%s: %s %s\n", 
				lexer.Position(), lexer.Token(), TokenString(tok)
			)
		}

		// nofile:2:0: // Comment begins at column 5. Comment
		// nofile:4:6: This Ident
		// nofile:4:11: line Ident
		// nofile:4:18: should Ident
		// nofile:4:22: not Ident
		// nofile:4:25: be Ident
		// nofile:4:34: included Ident
		// nofile:4:37: in Ident
		// nofile:4:41: the Ident
		// nofile:4:48: output Ident
		// nofile:4:0: . "."
		// nofile:10:0: /*
		// 	This multiline comment
		// 	should be extracted in
		// 	its entirety.
		// 	*/ Comment
		// 

	```



## 再想想