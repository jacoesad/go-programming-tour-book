## 1.3 便捷的时间工具

本节完成一个时间相关的工具，提高获取指定格式时间的效率。

### 1.3.1 获取时间

在internal目录下新建目录和文件，结构如下：

```
├── internal
│   ├── timer
│   │   └── time.go
```

在timer.go中新增如下获取时间代码：

**项目代码** `tour/internal/timer/time.go`

```go
func GetNowTime() time.Time {
	return time.Now()
}
```

在`GetNowTime`方法中对标准库time的Now方法进行封装，用于返回当前本地时间的Time对象。此处的封装主要为了便于后续对Time对象做进一步统一管理。

### 1.3.2 推算时间

在time.go新增如下方法：

**项目代码** `tour/internal/timer/time.go`

```go
func GetCalculateTime(currentTimer time.Time, d string) (time.Time, error){
	duration, err := time.ParseDuration(d)
	if err != nil {
		return time.Time{}, err
	}

	return currentTimer.Add(duration), nil
}
```

添加`ParseDuration`方法是因为我们预先并不知道传入的值是什么，因此最好用`ParseDuration`处理一下，从字符串中解析出`duration`。

如果预先知道准备的`duration`，且不需要适配，那么可以直接使用`Add`方法进行处理：

**标准库** `time/time.go` line656

```go
const (
	Nanosecond  Duration = 1
	Microsecond          = 1000 * Nanosecond
	Millisecond          = 1000 * Microsecond
	Second               = 1000 * Millisecond
	Minute               = 60 * Second
	Hour                 = 60 * Minute
)
```

**代码示例**

```go
timer.GetNowTime().Add(time.Second * 60)
```

### 1.3.3 初始化子命令

现在，需要将上面方法集成到子命令中，即创建项目的time子命令，在项目的cmd目录下新建time.go，新增如下代码：

**项目代码** `tour/cmd/time.go`

```go
var calculateTime string
var duration string

var timeCmd = &cobra.Command{
	Use:   "time",
	Short: "时间格式处理",
	Long:  "时间格式处理",
	Run:   func(cmd *cobra.Command, args []string) {},
}
```

在项目的cmd/root.go文件中进行相应的注册：

**项目代码** `tour/cmd/root.go`

```go
func init() {
	rootCmd.AddCommand(wordCmd)
	rootCmd.AddCommand(timeCmd)
}
```

每一个子命令都要注册到rootCmd中，否则无法使用。

#### 1. time now子命令

如果要获取当前时间，在time子命令下新增一个now子命令，用于处理具体的逻辑。在time.go中新增如下代码：

**项目代码** `tour/cmd/time.go`

```go
var nowTimeCmd = &cobra.Command{
	Use:   "now",
	Short: "获取当前时间",
	Long:  "获取当前时间",
	Run: func(cmd *cobra.Command, args []string) {
		nowTime := timer.GetNowTime()
		log.Printf("输出结果：%s, %d", nowTime.Format("2006-01-02 15:04:05"), nowTime.Unix())
	},
}
```

> **！！！注意！！！**
>
> `nowTime.Format("2006-01-02 15:04:05")`
>
> 这个部分一定需要用这个时间，否则解析会不正确。

在获取当前的Time对象后，一共输出了两种时间格式，分别是：

1. 第一种：通过Format方法输出按照既定的`2006-01-02 15:04:05`格式化时间；
2. 第二种：通过调用Unix方法返回的UNIX时间，即时间戳，值了自UTC 1970年1月1日起经过的秒数。

如果想要定义其他时间格式，则可以使用标准库time，它支持的格式如下：

**标准库** `time/format.go` line73

```go
const (
	ANSIC       = "Mon Jan _2 15:04:05 2006"
	UnixDate    = "Mon Jan _2 15:04:05 MST 2006"
	RubyDate    = "Mon Jan 02 15:04:05 -0700 2006"
	RFC822      = "02 Jan 06 15:04 MST"
	RFC822Z     = "02 Jan 06 15:04 -0700" // RFC822 with numeric zone
	RFC850      = "Monday, 02-Jan-06 15:04:05 MST"
	RFC1123     = "Mon, 02 Jan 2006 15:04:05 MST"
	RFC1123Z    = "Mon, 02 Jan 2006 15:04:05 -0700" // RFC1123 with numeric zone
	RFC3339     = "2006-01-02T15:04:05Z07:00"
	RFC3339Nano = "2006-01-02T15:04:05.999999999Z07:00"
	Kitchen     = "3:04PM"
	// Handy time stamps.
	Stamp      = "Jan _2 15:04:05"
	StampMilli = "Jan _2 15:04:05.000"
	StampMicro = "Jan _2 15:04:05.000000"
	StampNano  = "Jan _2 15:04:05.000000000"
)
```

使用预定义格式的代码如下：

**代码示例**

```go
t := time.Now().Format(time.RFC3339)
```

#### 2. time calc子命令

如果需要时间推算，则在time子命令下新增一个calc子命令，用于处理具体的逻辑。在time.go中新增如下代码：

**项目代码** `tour/cmd/time.go`

```go
var calculateTimeCmd = &cobra.Command{
	Use:   "calc",
	Short: "计算所需时间",
	Long:  "计算所需时间",
	Run: func(cmd *cobra.Command, args []string) {
		var currentTimer time.Time
		var layout = "2006-01-02 15:04:05"
		if calculateTime == "" {
			currentTimer = timer.GetNowTime()

		} else {
			var err error
			if !strings.Contains(calculateTime, " ") {
				layout = "2006-01-02"
			}
			currentTimer, err = time.Parse(layout, calculateTime)
			if err != nil {
				t, _ := strconv.Atoi(calculateTime)
				currentTimer = time.Unix(int64(t), 0)
			}
		}
		calculateTime, err := timer.GetCalculateTime(currentTimer, duration)
		if err != nil {
			log.Fatalf("timer.GetCalculateTime err: %v", err)
		}
		log.Printf("输出结果: %s, %d", calculateTime.Format(layout), calculateTime.Unix())
	},
}
```

在时间处理上，调用`strings.Contains`方法对空格进行包含判断：

- 若存在空格，则按既定的tld格式进行格式化；
- 否则以`2006-01-02`格式处理器；
- 若出现异常错误，则直接按时间戳格式进行转换处理。

最后我们对time命令的now、calc子命令和起关联的命令行参数进行注册，代码如下：

**项目代码** `tour/cmd/time.go`

```go
func init() {
	timeCmd.AddCommand(nowTimeCmd)
	timeCmd.AddCommand(calculateTimeCmd)

	calculateTimeCmd.Flags().StringVarP(&calculateTime, "calculate", "c", "",
		`需要计算的时间，有效单位为时间戳或已格式化后的时间`)
	calculateTimeCmd.Flags().StringVarP(&duration, "duration", "d", "",
		`持续时间，有效时间单位为"ns", "us"(or "μs"), "ms", "s", "m", "h"`)
}
```

### 1.3.4 验证

我们对功能进行验证，分别获取当前时间，以及推算的所传入时间的后五分钟和前两小时：

```bash
$ go run main.go time now
输出结果：2020-07-30 11:31:35, 1596079895
$ go run main.go time calc -c="2020-07-30 11:31:35" -d=5m
输出结果: 2020-07-30 11:36:35, 1596108995
$ go run main.go time calc -c="2020-07-30 11:31:35" -d=-2h
输出结果: 2020-07-30 09:31:35, 1596101495
```

### 1.3.5 时区问题

对于需要输入和输出时间的程序来说，必须要考虑系统所处的时区。在Go语言中，Location用来表示地区相关的时区，一个Location可能表示多个时区。

在标准库time中，提供了Location的两个时区：Local和UTC。Local表示当前系统本地时区；UTC表示通用协调时间，也就是零时区。标准库time默认使用的是UTC时区。

#### 1. Local是如何表示本地时区的

时区信息UNIX系统以标准格式存于文件中。这些文件位于`/usr/share/zoneinfo`中，而本地时区可以通过`/etc/localtime`获取。这是一个符号链接，指向`/usr/share/zoneinfo`中的某一个时区。例如本机中：

```bash
$ ls -l /etc/localtime
/etc/localtime -> /var/db/timezone/zoneinfo/Asia/Shanghai
```

在初始化Local是，标准库time通过读取`/etc/localtime`即可获取系统本地时区，代码如下：

**标准库 **`time/zoneinfo_unix.go ` line28

```go
func initLocal() {
	// consult $TZ to find the time zone to use.
	// no $TZ means use the system default /etc/localtime.
	// $TZ="" means use UTC.
	// $TZ="foo" means use /usr/share/zoneinfo/foo.

	tz, ok := syscall.Getenv("TZ")
	switch {
	case !ok:
		z, err := loadLocation("localtime", []string{"/etc/"})
		if err == nil {
			localLoc = *z
			localLoc.name = "Local"
			return
		}
	case tz != "" && tz != "UTC":
		if z, err := loadLocation(tz, zoneSources); err == nil {
			localLoc = *z
			return
		}
	}

	// Fall back to UTC.
	localLoc.name = "UTC"
}
```

#### 2. 设置时区

我们可以通过标准库中的`LoadLocation`方法根据名称获取特定时区的Location实例，原型如下：

**标准库** `time/zoneinfo.go` line281

```go
func LoadLocation(name string) (*Location, error) {
	if name == "" || name == "UTC" {
		return UTC, nil
	}
	if name == "Local" {
		return Local, nil
	}
	if containsDotDot(name) || name[0] == '/' || name[0] == '\\' {
		// No valid IANA Time Zone name contains a single dot,
		// much less dot dot. Likewise, none begin with a slash.
		return nil, errLocation
	}
	zoneinfoOnce.Do(func() {
		env, _ := syscall.Getenv("ZONEINFO")
		zoneinfo = &env
	})
	var firstErr error
	if *zoneinfo != "" {
		if zoneData, err := loadTzinfoFromDirOrZip(*zoneinfo, name); err == nil {
			if z, err := LoadLocationFromTZData(name, zoneData); err == nil {
				return z, nil
			}
			firstErr = err
		} else if err != syscall.ENOENT {
			firstErr = err
		}
	}
	if z, err := loadLocation(name, zoneSources); err == nil {
		return z, nil
	} else if firstErr == nil {
		firstErr = err
	}
	return nil, firstErr
}
```

在该方法中，如果传入的name是UTC或为空，则返回UTC；如果传入的name是Local，则返回当前的本地时区Local；否则name应该是IANA时区数据库中记录的地点名，如“America/New_York”。

为了保证获取的时间与期望时时区一致，我们需要修改获取时间的代码，设置当前时区为Asia/Shanghai，代码如下：

**项目代码** `tour/internal/timer/time.go`

```go
func GetNowTime() time.Time {
	location, _ := time.LoadLocation("Asia/Shanghai")
	return time.Now().In(location)
}
```

#### 3. 需要注意的 `time.Parse/Format`

在前面，我们用到了`time.Format`方法，与此相对应的`time.Parse`没有介绍。Parse方法会解析格式化的字符串并返回它表示的时间值，十分常见，并且有一个非常要注意的点。示例程序如下：

**示例代码** `example/ex-4.go`

```go
package main

import (
	"log"
	"time"
)

func main() {
	location, _ := time.LoadLocation("Asia/Shanghai")
	inputTime := "2020-07-30 12:34:56"
	layout := "2006-01-02 15:04:05"
	t, _ := time.Parse(layout, inputTime)
	dateTime := time.Unix(t.Unix(), 0).In(location).Format(layout)

	log.Printf("输入时间：%s，输出时间：%s", inputTime, dateTime)
}
```

运行，输出结果为：

```bash
$ go run ex-4.go
输入时间：2020-07-30 12:34:56，输出时间：2020-07-30 20:34:56
```

在调用Format方法前已经设置了时区，为什么还会出现时区问题？

实际上，因为Parse方法会尝试在入参的参数重分析并读取时区信息。如果如餐的参数没有制定时区信息，那么会默认使用UTC时间。因此在这种情况下，我们采用ParseInLocation方法指定时区，就可以解决，代码如下：

**示例代码** `example/ex-5.go`

```go
	t, _ := time.ParseInLocation(layout, inputTime, location)
	dateTime := time.Unix(t.Unix(), 0).In(location).Format(layout)
```

运行，输出结果为：

```bash
$ go run ex-5.go
输入时间：2020-07-30 12:34:56，输出时间：2020-07-30 12:34:56
```

也就是说，所有解析与格式化的操作最好制定时区信息，否则当项目已经上线，并且遇到了时区问题时，再进行清洗数据就比较麻烦。

#### 4. 我的系统时区是对的

在我们开发事，用的是本地或者预装好多开发环境，时区往往都是设置正确的，例如符合我们东八区的需求。可以查看本地localtime文件，命令如下：

```bash
$ cat /etc/localtime
TZ
...
CST-8
```

可以发现，实际上就是CST-8，即中国标准时间，UTC+8，因此不设置也不会出现异常。但是到了其他不熟环境就不一定，例如Docker中，假设景象没有经过时区调整，就会出现问题，比如日记写入时间不对、标准库time的转换存在问题等等。

### 1.3.6 参考时间的格式

`2006-01-02 15:04:05`是一个参考的时间格式，如同其他语言中的`Y-m-d H:i:s`格式，其功能是用于格式化时间。

为什么要用`2006-01-02 15:04:05`呢。在Go语言中，强调必须显示参考时间的格式，因此每个字符串都是一个时间戳，并非随便写的时间点。

```
Jan 2 15:04:05 2006 MST
1   2 3  4  5  6    -7
```















