# Structured Logging for Humans

[![go.dev][pkg-img]][pkg] [![goreport][report-img]][report] [![coverage][cov-img]][cov] ![stability-stable][stability-img]

## Features

* No Dependencies
* Intuitive Interfaces
* Rotating File Writer
* Pretty & Template Console Writer
* JournalD & EventLog Writer
* StdLog & Grpc & Logr Interceptor
* Contextual Fields
* High Performance

## Interfaces

### Logger
```go
// DefaultLogger is the global logger.
var DefaultLogger = Logger{
	Level:      DebugLevel,
	Caller:     0,
	TimeField:  "",
	TimeFormat: "",
	Writer:     os.Stderr,
}

// A Logger represents an active logging object that generates lines of JSON output to an io.Writer.
type Logger struct {
	// Level defines log levels.
	Level Level

	// Caller determines if adds the file:line of the "caller" key.
	Caller int

	// TimeField defines the time filed name in output.  It uses "time" in if empty.
	TimeField string

	// TimeFormat specifies the time format in output. It uses time.RFC3389 in if empty.
	// If set with `TimeFormatUnix`, `TimeFormatUnixMs`, times are formated as UNIX timestamp.
	TimeFormat string

	// Writer specifies the writer of output. It uses os.Stderr in if empty.
	Writer io.Writer
}
```

### FileWriter & ConsoleWriter
```go
// FileWriter is an io.WriteCloser that writes to the specified filename.
type FileWriter struct {
	// Filename is the file to write logs to.  Backup log files will be retained
	// in the same directory.
	Filename string

	// FileMode represents the file's mode and permission bits.  The default
	// mode is 0644
	FileMode os.FileMode

	// MaxSize is the maximum size in bytes of the log file before it gets rotated.
	MaxSize int64

	// MaxBackups is the maximum number of old log files to retain.  The default
	// is to retain all old log files
	MaxBackups int

	// LocalTime determines if the time used for formatting the timestamps in
	// log files is the computer's local time.  The default is to use UTC time.
	LocalTime bool

	// HostName determines if the hostname used for formatting in log files.
	HostName bool

	// ProcessID determines if the pid used for formatting in log files.
	ProcessID bool
}

// ConsoleWriter parses the JSON input and writes it in an
// (optionally) colorized, human-friendly format to os.Stderr
//
// Default output format:
//     {Time} {Level} {Goid} {Caller} > {Message} {Key}={Value} {Key}={Value}
type ConsoleWriter struct {
	// ColorOutput determines if used colorized output.
	ColorOutput bool

	// QuoteString determines if quoting string values.
	QuoteString bool

	// EndWithMessage determines if output message in the end.
	EndWithMessage bool

	// Template specifies an optional text/template for creating a
	// user-defined output format, available arguments are:
	//    type . struct {
	//        Time     string    // "2019-07-10T05:35:54.277Z"
	//        Level    string    // "info"
	//        Caller   string    // "prog.go:42"
	//        Goid     string    // "123"
	//        Message  string    // "a structure message"
	//        Stack    string    // "<stack string>"
	//        KeyValue []struct {
	//            Key   string       // "foo"
	//            Value interface{}  // "bar"
	//        }
	//    }
	// See https://github.com/phuslu/log#template-console-writer for example.
	//
	// If Template is not nil, ColorOutput, QuoteString and EndWithMessage are override.
	Template *template.Template

	// Writer is the output destination. using os.Stderr if empty.
	Writer io.Writer
}
```

## Getting Started

### Simple Logging Example

A out of box example. [![playground][play-simple-img]][play-simple]
```go
package main

import (
	"github.com/phuslu/log"
)

func main() {
	log.Printf("Hello, %s", "世界")
	log.Info().Str("foo", "bar").Int("number", 42).Msg("hi, phuslog")
	log.Error().Str("foo", "bar").Int("number", 42).Msgf("oops, %s", "phuslog")
}

// Output:
//   {"time":"2020-03-22T09:58:41.828Z","message":"Hello, 世界"}
//   {"time":"2020-03-22T09:58:41.828Z","level":"info","foo":"bar","number":42,"message":"hi, phuslog"}
//   {"time":"2020-03-22T09:58:41.828Z","level":"error","foo":"bar","number":42,"message":"oops, phuslog"}
```
> Note: By default log writes to `os.Stderr`

### Customize the configuration and formatting:

To customize logger filed name and format. [![playground][play-customize-img]][play-customize]
```go
log.DefaultLogger = log.Logger{
	Level:      log.InfoLevel,
	Caller:     1,
	TimeField:  "date",
	TimeFormat: "2006-01-02",
	Writer:     os.Stderr,
}
log.Info().Str("foo", "bar").Msgf("hello %s", "world")

// Output: {"date":"2019-07-04","level":"info","caller":"prog.go:16","foo":"bar","message":"hello world"}
```

### Rotating File Writer

To log to a rotating file, use `FileWriter`. [![playground][play-file-img]][play-file]
```go
package main

import (
	"time"

	"github.com/phuslu/log"
	"github.com/robfig/cron/v3"
)

func main() {
	logger := log.Logger{
		Level:      log.ParseLevel("info"),
		Writer:     &log.FileWriter{
			Filename:   "main.log",
			FileMode:   0600,
			MaxSize:    50*1024*1024,
			MaxBackups: 7,
			LocalTime:  false,
		},
	}

	runner := cron.New(cron.WithSeconds(), cron.WithLocation(time.UTC))
	runner.AddFunc("0 0 * * * *", func() { logger.Writer.(*log.FileWriter).Rotate() })
	go runner.Run()

	for {
		time.Sleep(time.Second)
		logger.Info().Msg("hello world")
	}
}
```

### Pretty Console Writer

To log a human-friendly, colorized output, use `ConsoleWriter`. [![playground][play-pretty-img]][play-pretty]

```go
if log.IsTerminal(os.Stderr.Fd()) {
	log.DefaultLogger = log.Logger{
		Caller: 1,
		Writer: &log.ConsoleWriter{
			ColorOutput:    true,
			QuoteString:    true,
			EndWithMessage: true,
		},
	}
}

log.Printf("a printf style line")
log.Info().Err(errors.New("an error")).Int("everything", 42).Str("foo", "bar").Msg("hello world")
```
![Pretty logging][pretty-img]
> Note: pretty logging also works on windows console

### Template Console Writer

To log with user-defined format(e.g. glog), using `ConsoleWriter.Template`. [![playground][play-template-img]][play-template]

```go
package main

import (
	"text/template"
	"github.com/phuslu/log"
)

var glog = (&log.Logger{
	Level:      log.InfoLevel,
	Caller:     1,
	TimeFormat: "0102 15:04:05.999999",
	Writer: &log.ConsoleWriter{
		Template: template.Must(template.New("").Parse(
			`{{.Level.First}}{{.Time}} {{.Goid}} {{.Caller}}] {{.Message}}`)),
	},
}).Sugar(nil)

func main() {
	glog.Infof("hello glog %s", "Info")
	glog.Warnf("hello glog %s", "Earn")
	glog.Errorf("hello glog %s", "Error")
	glog.Fatalf("hello glog %s", "Fatal")
}

// Output:
// I0725 09:59:57.503246 19 console_test.go:183] hello glog Info
// W0725 09:59:57.504247 19 console_test.go:184] hello glog Earn
// E0725 09:59:57.504247 19 console_test.go:185] hello glog Error
// F0725 09:59:57.504247 19 console_test.go:186] hello glog Fatal
```
> Note: refer to [ColorTemplate](https://github.com/phuslu/log/blob/master/console.go#L355) and [sprig](https://github.com/Masterminds/sprig) to make it functional.

### JournalWriter & EventlogWriter

To log to linux systemd journald, using `JournalWriter`.

```go
log.DefaultLogger.Writer = &log.JournalWriter{}
log.Info().Int("number", 42).Str("foo", "bar").Msg("hello world")
```

To log to windows system event, using `EventlogWriter`.

```go
log.DefaultLogger.Writer = &log.EventlogWriter{
	Source: ".NET Runtime",
	ID:     1000,
}
log.Info().Int("number", 42).Str("foo", "bar").Msg("hello world")
```

### MultiWriter

To log to different writers by different levels, use `MultiWriter`.

```go
log.DefaultLogger.Writer = &log.MultiWriter{
	InfoWriter: &log.FileWriter{
		Filename: "file.info.log",
		MaxSize:  100_000_000,
	},
	WarnWriter: &log.FileWriter{
		Filename: "file.warn.log",
		MaxSize:  20_000_000,
	},
	ErrorWriter: &log.FileWriter{
		Filename: "file.error.log",
		MaxSize:  20_000_000,
	},
	StderrWriter: &log.ConsoleWriter{
		ColorOutput:    true,
		EndWithMessage: true,
	},
	StderrLevel: ErrorLevel,
}
log.Info().Int("number", 42).Str("foo", "bar").Msg("a info log")
log.Warn().Int("number", 42).Str("foo", "bar").Msg("a warn log")
log.Error().Int("number", 42).Str("foo", "bar").Msg("a error log")
```

### Sugar Logger

In contexts where performance is nice, but not critical, use the `SugaredLogger`. It's 20% slower than `Logger` but still faster than other structured logging packages [![playground][play-sugar-img]][play-sugar]

```go
package main

import (
	"github.com/phuslu/log"
)

func main() {
	sugar := log.DefaultLogger.Sugar(log.NewContext(nil).Str("tag", "hi suagr").Value())
	sugar.Infof("hello %s", "世界")
	sugar.Infow("i am a leading message", "foo", "bar", "number", 42)

	sugar = sugar.Level(log.ErrorLevel)
	sugar.Printf("hello %s", "世界")
	sugar.Log("number", 42, "a_key", "a_value", "message", "a suagr message")
}
```

### StdLog & Logr & Grpc Interceptor

Using wrapped loggers for stdlog/grpc/logr. [![playground][play-interceptor-img]][play-interceptor]

```go
package main

import (
	stdLog "log"
	"github.com/go-logr/logr"
	"github.com/phuslu/log"
	"google.golang.org/grpc/grpclog"
)

func main() {
	ctx := log.NewContext(nil).Str("tag", "hi log").Value()

	var stdlog *stdLog.Logger = log.DefaultLogger.Std(log.InfoLevel, ctx, "prefix ", stdLog.LstdFlags)
	stdlog.Print("hello from stdlog Print")
	stdlog.Println("hello from stdlog Println")
	stdlog.Printf("hello from stdlog %s", "Printf")

	var grpclog grpclog.LoggerV2 = log.DefaultLogger.Grpc(ctx)
	grpclog.Infof("hello %s", "grpclog Infof message")
	grpclog.Errorf("hello %s", "grpclog Errorf message")

	var logrLog logr.Logger = log.DefaultLogger.Logr(ctx)
	logrLog = logrLog.WithName("a_named_logger").WithValues("a_key", "a_value")
	logrLog.Info("hello", "foo", "bar", "number", 42)
	logrLog.Error(errors.New("this is a error"), "hello", "foo", "bar", "number", 42)
}
```

### Contextual Fields

To add preserved `key:value` pairs to each event, use `NewContext`. [![playground][play-context-img]][play-context]

```go
ctx := log.NewContext(nil).Str("ctx_str", "a ctx str").Value()

logger := log.Logger{Level: log.InfoLevel}
logger.Debug().Context(ctx).Int("no0", 0).Msg("zero")
logger.Info().Context(ctx).Int("no1", 1).Msg("first")
logger.Info().Context(ctx).Int("no2", 2).Msg("second")

// Output:
//   {"time":"2020-07-12T05:03:43.949Z","level":"info","ctx_str":"a ctx str","no1":1,"message":"first"}
//   {"time":"2020-07-12T05:03:43.949Z","level":"info","ctx_str":"a ctx str","no2":2,"message":"second"}
```

### High Performance

A quick and simple benchmark with logrus/zap/zerolog, performance advantage comes from [inline function][inline function], [loop unrolling][loop unrolling] and [runtime linkage][runtime linkage].

```go
// go test -v -run=none -bench=. -benchtime=10s -benchmem log_test.go
package main

import (
	"io/ioutil"
	"testing"

	"github.com/phuslu/log"
	"github.com/rs/zerolog"
	"github.com/sirupsen/logrus"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

var fakeMessage = "Test logging, but use a somewhat realistic message length. "

func BenchmarkLogrus(b *testing.B) {
	logger := logrus.New()
	logger.SetFormatter(&logrus.JSONFormatter{})
	logger.SetOutput(ioutil.Discard)
	for i := 0; i < b.N; i++ {
		logger.WithFields(logrus.Fields{"foo": "bar", "int": 42}).Info(fakeMessage)
	}
}

func BenchmarkZap(b *testing.B) {
	logger := zap.New(zapcore.NewCore(
		zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig()),
		zapcore.AddSync(ioutil.Discard),
		zapcore.InfoLevel,
	))
	for i := 0; i < b.N; i++ {
		logger.Info(fakeMessage, zap.String("foo", "bar"), zap.Int("int", 42))
	}
}

func BenchmarkZeroLog(b *testing.B) {
	logger := zerolog.New(ioutil.Discard).With().Timestamp().Logger()
	for i := 0; i < b.N; i++ {
		logger.Info().Str("foo", "bar").Int("int", 42).Msg(fakeMessage)
	}
}

func BenchmarkPhusLog(b *testing.B) {
	logger := log.Logger{Writer: ioutil.Discard}
	for i := 0; i < b.N; i++ {
		logger.Info().Str("foo", "bar").Int("int", 42).Msg(fakeMessage)
	}
}
```
Performance results on Amazon Linux c5.large instance:
```
BenchmarkLogrus-8        3216494              3763 ns/op            1834 B/op         31 allocs/op
BenchmarkZap-8          15588312               789 ns/op             128 B/op          1 allocs/op
BenchmarkZeroLog-8      32658090               400 ns/op               0 B/op          0 allocs/op
BenchmarkPhusLog-8      71507228               161 ns/op               0 B/op          0 allocs/op
```

### Acknowledgment
This log is heavily inspired by [zerolog][zerolog], [glog][glog], [quicktemplate][quicktemplate], [zap][zap] and [lumberjack][lumberjack].

[pkg-img]: http://img.shields.io/badge/godoc-reference-5272B4.svg?style=flat-square
[pkg]: https://godoc.org/github.com/phuslu/log
[report-img]: https://goreportcard.com/badge/github.com/phuslu/log
[report]: https://goreportcard.com/report/github.com/phuslu/log
[cov-img]: http://gocover.io/_badge/github.com/phuslu/log
[cov]: https://gocover.io/github.com/phuslu/log
[stability-img]: https://img.shields.io/badge/stability-stable-green.svg
[play-simple-img]: https://img.shields.io/badge/playground-NGV25aBKmYH-29BEB0?style=flat&logo=go
[play-simple]: https://play.golang.org/p/NGV25aBKmYH
[play-customize-img]: https://img.shields.io/badge/playground-U2TYAgV7VCR-29BEB0?style=flat&logo=go
[play-customize]: https://play.golang.org/p/U2TYAgV7VCR
[play-file-img]: https://img.shields.io/badge/playground-nS--ILxFyhHM-29BEB0?style=flat&logo=go
[play-file]: https://play.golang.org/p/nS-ILxFyhHM
[play-pretty-img]: https://img.shields.io/badge/playground-CD1LClgEvS4-29BEB0?style=flat&logo=go
[play-pretty]: https://play.golang.org/p/CD1LClgEvS4
[pretty-img]: https://user-images.githubusercontent.com/195836/87854177-b16da980-c942-11ea-9b00-5f1b092452f3.png
[play-template-img]: https://img.shields.io/badge/playground-0sQ03po5N3X-29BEB0?style=flat&logo=go
[play-template]: https://play.golang.org/p/0sQ03po5N3X
[play-context-img]: https://img.shields.io/badge/playground-oAVAo302faf-29BEB0?style=flat&logo=go
[play-context]: https://play.golang.org/p/oAVAo302faf
[play-sugar-img]: https://img.shields.io/badge/playground-iGfD_wOcA6c-29BEB0?style=flat&logo=go
[play-sugar]: https://play.golang.org/p/iGfD_wOcA6c
[play-interceptor]: https://play.golang.org/p/upmVP5cO62Y
[play-interceptor-img]: https://img.shields.io/badge/playground-upmVP5cO62Y-29BEB0?style=flat&logo=go
[inline function]: https://en.wikipedia.org/wiki/Inline_function
[loop unrolling]: https://en.wikipedia.org/wiki/Loop_unrolling
[runtime linkage]: https://github.com/golang/go/issues/15006
[zerolog]: https://github.com/rs/zerolog
[glog]: https://github.com/golang/glog
[quicktemplate]: https://github.com/valyala/quicktemplate
[zap]: https://github.com/uber-go/zap
[lumberjack]: https://github.com/natefinch/lumberjack
