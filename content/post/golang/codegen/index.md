---
title: golang代码生成
description: 
slug: 
date: 2024-01-29
categories:
    - golang
tags:
    - 代码生成
weight: 1       
---

## 动机
我们经常会看到很多的go语言库会使用`go:generate xxx`命令来减少一些重复性代码的编写（boilerplate codes）， 但是go:generate命令本身不是达到减轻重复性工作的关键， 关键是上面的xxx(指代任何一种代码生成工具)是如何帮组我们实现代码生成的呢？  
比如, 我们在写gin的controller的时候(MVC架构)， 我们通常会通过如下方式提取http请求对象的参数：
```golang
// controller/controller.go
type GetUserParmas struct {
	Name     string `form:"name"`
	Email    string `form:"email"`
	IsAdmin  int    `form:"is_admin"`
}

func (con *Controller) GetUser() gin.HandlerFunc {
	return func(c *gin.Context) {
		var rq GetUserParmas

		if err := c.ShouldBindQuery(&rq); err != nil {
			response.BadRequest(c, err)
			return
		}
        
        // 

	}
}
```
对Get请求我们我们会像上面那样调用`ShouldBindQuery`来解析参数， 问题的关键在于`GetUserParams`这个表示请求的结构体， 它在结构上其实和`User`结构体其实是类似的。  
我们在应用MVC的时候， 肯定会定义一些数据的模型， 如果使用gorm这类的ORM工具的话， 会直接借用这些结构体来进行对数据库的操作。但是在定义数据模型的时候， 用的tag可能会类似于:
```golang
//models/users.go
type User struct {
	Name     string `gorm:"column:name"`
	Email    string `gorm:"column:email"`
	IsAdmin  int    `gorm:"column:is_admin"`
}
```
可以看到`User`和`GetGetUserParams`唯一的差别只是它们的tag是不同的而已， 上面演示的只是Get请求解析参数的场景， 如果还有类似的其他场景呢， 比如使用`ShouldBindJson`, 那么在我们的tag里面就需要加上json标签了， 很显然这个过程是很繁杂的。  
我们希望我们有一个聪明的工具， 可以帮助我们基于`User`以及其他的结构体， 自动生成生成`GetUserParmas`之类的代码， 减少重复的工作量.



## 实现
实现的步骤大概分为：
-  收集结构体  
-  生成代码


### 收集结构体
首先我们需要收集到所有的结构体的相关信息，意味着需要知道结构体的名称， 结构体的字段， 字段类型， 字段标签等等， 譬如上面的`User`结构体中个字段信息。实现这个目标的方法大体上会有两种：
- 反射
- 将代码解析成抽象语法树然后遍历

我们这里会使用第二种方法达到我们的目的。因为几乎所有的语言都会提供将源代码解析为目标语言AST树（抽象语法树）的工具， golang也不例外， golang的`go/parser`包中的`ParseFile`方法会将源码解析成AST树

> 严格来讲， 使用正则表达式之类的方法应该也是可以的， 因为Parse代码的过程本身，通常也会包含某种形式的正则表达式来实现所谓的词法分析（lexical analyze）

```golang
fset := token.NewFileSet()

var contents interface{}

node, err := parser.ParseFile(fset, file, contents, parser.ParseComments)
```

上面的file是一个源码文件名称，ParseFile会读取并把代码解析为ast.File对象， 也就是上面的node。

接着我们会沿着ast.File这棵语法树进行遍历， 找到我们需要的结构体对象：

``` golang
structs := collectStructs(node)
```

``` golang

type structType struct {
	name string
	node *ast.StructType
}

func collectStructs(node ast.Node) []*structType {
	structs := make([]*structType, 0)

	collectStructs := func(n ast.Node) bool {
		var t ast.Expr
		var structName string

		switch x := n.(type) {
		case *ast.TypeSpec:
			if x.Type == nil {
				return true
			}

			structName = x.Name.Name
			t = x.Type
		case *ast.CompositeLit:
			t = x.Type
		case *ast.ValueSpec:
			structName = x.Names[0].Name
			t = x.Type
		case *ast.Field:
			if len(x.Names) != 0 {
				structName = x.Names[0].Name
			}
			t = x.Type
		}

		t = deref(t)

		x, ok := t.(*ast.StructType)
		if !ok {
			return true
		}

		structs = append(structs, &structType{
			name: structName,
			node: x,
		})
		return true
	}

	ast.Inspect(node, collectStructs)
	return structs
}
```
collectStructs的核心其实是通过ast.Inspect方法进行遍历AST树， 它的第二个参数是一个参数为ast.Node的回调函数， 回调函数如果返回true， 会继续遍历。  
上面的代码会遍历AST树的所有节点Node， 然后将目标对象ast.StructType收集起来返回; 注意, 我们这里其实自定义了一个structType对象， 原因在于ast.StructType没有包含结构体名称的字段， 而结构体名称后续是需要用到的
> collectStructs函数参考了https://github.com/fatih/gomodifytags中的实现

这里稍微插播一下ast.Inspect的实现：
```golang 
func Inspect(node Node, f func(Node) bool) {
	Walk(inspector(f), node)
}

func Walk(v Visitor, node Node) {
	if v = v.Visit(node); v == nil {
		return
	}

	// walk children
	// (the order of the cases matches the order
	// of the corresponding node types in ast.go)
	switch n := node.(type) {
	// Comments and fields
	case *Comment:
		// nothing to do

	case *CommentGroup:
		for _, c := range n.List {
			Walk(v, c)
		}

	case *Field:
		if n.Doc != nil {
			Walk(v, n.Doc)
		}
		walkIdentList(v, n.Names)
		if n.Type != nil {
			Walk(v, n.Type)
		}
		if n.Tag != nil {
			Walk(v, n.Tag)
		}
		if n.Comment != nil {
			Walk(v, n.Comment)
		}

	case *FieldList:
		for _, f := range n.List {
			Walk(v, f)
		}

    （中间略）

    	// Files and packages
	case *File:
		if n.Doc != nil {
			Walk(v, n.Doc)
		}
		Walk(v, n.Name)
		walkDeclList(v, n.Decls)
		// don't walk n.Comments - they have been
		// visited already through the individual
		// nodes

	case *Package:
		for _, f := range n.Files {
			Walk(v, f)
		}

	default:
		panic(fmt.Sprintf("ast.Walk: unexpected node type %T", n))
	}
```
这里使用了递归下降的方法去遍历整棵树，通过对节点类型的判断， 使用不同的方法(Walk, walkDeclList等)来更深一步地进行递归调用。

当我们收集完了所有结构体之后， 为了方便后续的代码生成环节， 我们这里对收集的结构体进行进一步的挖掘， 获取结构体中的字段名称、类型、Tag等信息：
```golang
components, err := collectStructComponents(structs)

func collectStructComponents(structs []*structType) (map[string]structComponets, error) {
	result := make(map[string]structComponets, len(structs))

	for _, s := range structs {
		sc := structComponets{
			Name: s.name,
		}

		fieldNames := make([]string, 0)
		fieldTypes := make([]string, 0)
		fieldTags := make([]string, 0)

		for _, f := range s.node.Fields.List {
			fieldName := ""

			if len(f.Names) != 0 {
				for _, field := range f.Names {
					fieldName = field.Name

				}
			}

			if f.Names == nil {
				ident, ok := f.Type.(*ast.Ident)
				if !ok {
					continue
				}
				fieldName = ident.Name
			}

			if fieldName == "" {
				continue
			}

			if f.Tag == nil {
				f.Tag = &ast.BasicLit{}
			}

			fieldNames = append(fieldNames, fieldName)

			// add Types
			typeExpr := f.Type
			typeString := types.ExprString(typeExpr)
			fieldTypes = append(fieldTypes, typeString)

			// add Tags
			tagExpr := f.Tag
			tagString := types.ExprString(tagExpr)
			fieldTags = append(fieldTags, tagString)

		}
		sc.FieldNames = fieldNames
		sc.FieldTypes = fieldTypes
		sc.FieldTags = fieldTags
		result[sc.Name] = sc
	}

	return result, nil
}
```
代码整体比较简单， 就是遍历StructType的Fields.List来获取每个结构体字段的详细信息， 最后汇总成一个字典。

### 生成代码
当我们有了所需的结构体信息之后， 生成代码就是一件非常水到渠成的事情; 这里我们会使用golang自带template包来实现代码生成
```golang
genCode(file, components)


func genCode(file string, scm map[string]structComponets) {
	fileName := strings.TrimRight(file, ".go")
	src := fmt.Sprintf("package %s", pkgName)

	for _, v := range scm {
		forms, err := genForm(v)
		if err != nil {
			log.Fatal(err)
		}
		src += "\n" + forms

		jsons, err := genJson(v)
		if err != nil {
			log.Fatal(err)
		}
		src += "\n" + jsons
	}
	finalSrc, err := format.Source([]byte(src))
	if err != nil {
		log.Fatal(err)
	}

	// fmt.Println(string(finalSrc))
	outFile := fmt.Sprintf("%s_param.go", fileName)
	os.WriteFile(outFile, finalSrc, 0644)
}
```
`genCode`会遍历我们上一步获取的字典， 然后调用`genForm`和`genJson`生成我们想要得到的目标代码:
```golang
func ToSnakeCase(str string) string {
	snake := matchFirstCap.ReplaceAllString(str, "${1}_${2}")
	snake = matchAllCap.ReplaceAllString(snake, "${1}_${2}")
	return strings.ToLower(snake)
}

var (
	matchFirstCap = regexp.MustCompile("(.)([A-Z][a-z]+)")
	matchAllCap   = regexp.MustCompile("([a-z0-9])([A-Z])")
	funcMap       = template.FuncMap{
		"ToLower": strings.ToLower,
		"ToSnake": ToSnakeCase,
	}
)

func genForm(sc structComponets) (string, error) {
	tempString := `
	type {{.Name}}Form struct {
		 {{range $i, $e := .FieldNames}} 
		    {{$e}}	{{index $.FieldTypes $i}}` + "   `form:\"{{$e|ToSnake}}\"`" + `{{end}}` + `
			}`

	temp, err := template.New("controller").Funcs(funcMap).Parse(tempString)
	if err != nil {
		return "", err
	}
	buff := bytes.NewBufferString("")

	err = temp.Execute(buff, sc)

	if err != nil {
		return "", err
	}
	return buff.String(), nil
}
```
上面以genForm为例， 我们使用template的Excute方法将Struct中的诸多信息注入到了代码模板中

## 牛刀小试
上面的代码我已上传至[github](https://github.com/superjcd/paramgen)， 我们可以下载下来使用一下：
```shell
go install github.com/superjcd/paramgen
``` 
然后我们在目标文件的最上方(譬如下面这样)， 添加`//go:generate paramgen`
```golang
//go:generate  paramgen
package model

type User struct {
	gorm.Model
	Name     string
	Password string
	Email    string
	IsAdmin  int
}
```
然后再根目录的终端运行：
```shell
go generate ./...
```
然后就会看到上面文件的旁边新生成了后缀`_param`的go文件.


## Reference
- [the-ultimate-guide-to-writing-a-go-tool](https://arslan.io/2017/09/14/the-ultimate-guide-to-writing-a-go-tool/)  
- [go-command-generate](https://yushuangqi.com/blog/2017/go-command-generate.html)