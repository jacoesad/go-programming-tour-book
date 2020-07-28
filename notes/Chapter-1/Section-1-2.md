## 1.2 单词格式转换

本节通过编写一个小工具，来实现单词字符串各种格式的转换。

### 1.2.1 安装`Cobra`

安装本项目依赖的基础库`Cobra`，在项目根目录中执行：

```bash
$ go get -u github.com/spf13/cobra@v1.0.0
go: downloading github.com/spf13/cobra v1.0.0
go: downloading github.com/spf13/pflag v1.0.3
go: downloading github.com/inconshreveable/mousetrap v1.0.0
go: github.com/spf13/pflag upgrade => v1.0.5
go: downloading github.com/spf13/pflag v1.0.5
```

### 1.2.2 初始化`cmd`和`word`子命令

对目录进行初始化，目录结构如下所示：

```bash
$ tree
.
├── cmd
│   ├── root.go
│   └── word.go
├── go.mod
├── go.sum
├── internal
│   └── word
│       └── word.go
├── main.go
└── pkg
```

在本项目中创建入口文件main.go，并新增三个目录，分别是cmd、internal、pkg。

首先，在cmd目录下新建word.go，用于放置单词格式转换的子命令，并新增如下代码：

 `tour/cmd/word.go`

```go
var wordCmd = &cobra.Command{
	Use:   "word",
	Short: "单词格式转换",
	Long:  "支持多种单词格式转换",
	Run: func(cmd *cobra.Command, args []string) {},
}

func init() {}
```

然后，在cmd目录下新建root.go，用于放置根命令，并新增如下代码：

 `tour/cmd/root.go`

```go
var rootCmd = &cobra.Command{
	Use:   "",
	Short: "",
	Long:  "",
}

func Execute() error {
	return rootCmd.Execute()
}

func init() {
	rootCmd.AddCommand(wordCmd)
}
```

最后，在启动文件main.go中，新增如下代码：

 `tour/main.go`

```go
func main() {
	err := cmd.Execute()
	if err != nil {
		log.Fatalf("cmd.Execute err %v", err)
	}
}
```

### 1.2.3 单词转换

我们需要对单词转换类型进行编码，功能具体如下：

-   单词全部转为小写；
-   单词全部转为大写；
-   下画线单词转为大写驼峰单词；
-   下画线单词转为小写驼峰单词；
-   驼峰单词转为下画线单词；

在项目的internal目录下，新建word目录以及文件，并在word.go中新增代码。

#### 1. 单词全部转换为大些/小写

```go
func ToUpper(s string) string {
	return strings.ToUpper(s)
}

func ToLower(s string) string {
	return strings.ToLower(s)
}
```

#### 2. 下画线单词转为大写驼峰单词

```go
func UnderscoreToUpperCamelCase(s string) string {
	s = strings.Replace(s, "_", " ", -1)
	s = strings.Title(s)
	return strings.Replace(s, " ", "", -1)
}
```

#### 3. 下画线单词转为小写驼峰单词

```go
func UnderscoreToLowerCamelCase(s string) string {
	s = UnderscoreToUpperCamelCase(s)
	return string(unicode.ToLower(rune(s[0]))) +s[1:]
}
```

#### 4. 驼峰单词转为下画线单词

```go
func CamelCaseToUnderscore(s string) string {
	var output []rune
	for i, r := range s {
		if i == 0 {
			output = append(output, unicode.ToLower(r))
			continue
		}

		if unicode.IsUpper(r){
			output = append(output, '_')
		}

		output = append(output, unicode.ToLower(r))
	}
	return string(output)
}
```

### 1.2.4 `word`子命令

在完成单词的转换方法后，编写word子命令，将对应的方法集成到Command中。在`tour/cmd/word.go`中，定义目前单词所支持的转换模式枚举值，新增如下代码：

`tour/cmd/word.go`

```go
const (
	ModeUpper                      = iota + 1 // 全部单词转为大写
	ModeLower                                 // 全部单词转为小写
	ModeUnderscoreToUpperCamelcase            // 下划线单词转为大写驼峰单词
	ModeUnderscoreToLowerCamelcase            // 下划线单词转为小写驼峰单词
	ModeCamelcaseToUnderscore                 // 驼峰单词转为下划线单词
)
```

接下来对具体的单词子命令进程设置和集成，新增/修改如下代码：

`tour/cmd/word.go`

```go
var desc = strings.Join([]string{
	"该子命令支持各种单词格式转换，模式如下：",
	"1：全部单词转为大写",
	"2：全部单词转为小写",
	"3：下划线单词转为大写驼峰单词",
	"4：下划线单词转为小写驼峰单词",
	"5：驼峰单词转为下划线单词",
}, "\n")

var wordCmd = &cobra.Command{
	Use:   "word",
	Short: "单词格式转换",
	Long:  desc,
	Run: func(cmd *cobra.Command, args []string) {
		var content string
		switch mode {
		case ModeUpper:
			content = word.ToUpper(str)
		case ModeLower:
			content = word.ToLower(str)
		case ModeUnderscoreToUpperCamelcase:
			content = word.UnderscoreToUpperCamelCase(str)
		case ModeUnderscoreToLowerCamelcase:
			content = word.UnderscoreToLowerCamelCase(str)
		case ModeCamelcaseToUnderscore:
			content = word.CamelCaseToUnderscore(str)
		default:
			log.Fatalf("暂不支持该转换模式，请执行 help word 查看帮助文档")
		}

		log.Printf("输出结果：%s", content)
	},
}
```

上述代码中，核心在于子命令word的`cobra.Command`调用和设置，其中一共包含如下三个常用选项，分别是：

-   Use：子命令的命令标识
-   Short：简短说明，在help命令输出的帮助信息中展示。
-   Long：完整说明，在help命令输出的帮助信息中展示。

下面根绝单词转换所需的参数，即单词内容和转换的模式，进行命名行参数的设置和初始化，新增如下代码：

`tour/cmd/word.go`

```go
var str string
var mode int8

func init() {
	wordCmd.Flags().StringVarP(&str, "str", "s", "", "请输入单词内容")
	wordCmd.Flags().Int8VarP(&mode, "mode", "m", 0, "请输入单词转换模式")
}
```

在VarP系列的方法中，

-   第一个参数为需要绑定的变量，
-   第二个参数为接受该参数的完整的命令标识，
-   第三个参数对应为短标识，
-   第四个参数为默认值，
-   第五个参数为使用说明。

### 1.2.5 验证

一般来说，在拿到一个CLI应用程序后，我们会先执行help命令查看其帮助，方法如下：

```bash
$ go run main.go help word
该子命令支持各种单词格式转换，模式如下：
1：全部单词转为大写
2：全部单词转为小写
3：下划线单词转为大写驼峰单词
4：下划线单词转为小写驼峰单词
5：驼峰单词转为下划线单词

Usage:
   word [flags]

Flags:
  -h, --help         help for word
  -m, --mode int8    请输入单词转换模式
  -s, --str string   请输入单词内容
```

手动验证五种模式是否正常，方法如下：

```bash
$ go run main.go word -s=jacoesad -m=1
输出结果：JACOESAD
$ go run main.go word -s=JACOESAD -m=2
输出结果：jacoesad
$ go run main.go word -s=jacoe_sad -m=3
输出结果：JacoeSad
$ go run main.go word -s=jacoe_sad -m=4
输出结果：jacoeSad
$ go run main.go word -s=jacoeSad -m=5
输出结果：jacoe_sad
$ go run main.go word -s=JacoeSad -m=5
输出结果：jacoe_sad
```

### 1.2.6 小结

基于第三方开源库`Cobra`和标准库`strings`、`unicode`实现了多种模式的单词转换功能。

