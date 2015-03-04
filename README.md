# logxi

Log Eleven (XI) is a [12 factor app](http://12factor.net/logs)
logger built for performance and happy development.

TL;DR

*   Faster and less allocations than logrus and log15
*   Enforces key-value pair logging
*   Loggers and their level are configurable via environment variables
*   JSON formatter or very efficient key-value text formatter for production
*   Happy, colorful, developer friendly logger in terminal. Warnings
    and errors are emphasized with their call stack.
*   Defines a simple interface for logging
*   Colored logging works on Windows
*   Loggers are named
*   Has level guards to avoid cost of passing arguments

### Installation

    go get -u github.com/mgutz/logxi/v1

### Getting Started

```go
import "github.com/mgutz/logxi/v1"

// create package variable for Logger interface
var logger log.Logger

func main() {
    // use default logger
    if log.IsInfo() {
        log.Info("Hello", "name", "mario")
    }

    // create a logger for your package, assigning a unique
    // name which can be enabled from environment variables
    logger = log.New("pkg")

    // specify a writer
    modelLogger = log.NewLogger(os.Stdout, "models")

    db, err := sql.Open("postgres", "dbname=testdb")
    if err != nil {
        modelLogger.Error("Could not open database", "err", err)
    }

    a := "apple"
    o := "orange"
    if log.IsDebug() {
        // use key-value pairs after message
        logger.Debug("OK", "a", a, "o", o)
    }
}
```

logxi defaults to showing warnings and above. To view all logs

    LOGXI=* go run main.go

## Higlights

This logger package

*   Is fast in production environment

    A logger should be efficient and minimize performance tax.
    logxi encodes JSON 2X faster than logrus and log15 with primitive types.
    When diagnosing a problem in production, troubleshooting often means
    enabling small trace data in `Debug` and `Info` statements for some
    period of time.

    ```
# primitive types
BenchmarkLogxi         100000     14832 ns/op    2112 B/op     72 allocs/op
BenchmarkLogrus         30000     43343 ns/op    7704 B/op    174 allocs/op
BenchmarkLog15          30000     58706 ns/op    8780 B/op    220 allocs/op

# nested object
BenchmarkLogxiComplex   30000     48154 ns/op    8201 B/op    200 allocs/op
BenchmarkLogrusComplex  20000     61914 ns/op   10889 B/op    257 allocs/op
BenchmarkLog15Complex   20000     85782 ns/op   12277 B/op    294 allocs/op
```

*   Is developer friendly in development environments. The default
    formatter in terminals is colorful, prints file and line numbers
    when warnings and errors occur.

    `HappyDevFormatter` is not too concerned with performance
    and should never be used in production environments.

*   Logs machine parsable output in production environments.
    The recommended formatter for production is `JSONFormatter`.

    `TextFormatter` may also be used which is MUCH faster than
    JSON but requires a unique separator.

*   Has level guards to avoid the cost of passing arguments. These _SHOULD_
    always be used.

        if log.IsDebug() {
            log.Debug("some ")
        }

        if log.IsInfo() {
            log.Info("some ")
        }

        if log.IsWarn() {
            log.Warn("some ")
        }

    Error and Fatal do not have guards, they MUST always log.

*   Conforms to a logging interface so it can be replaced.

        type Logger interface {
            Debug(string, ...interface{})
            Info(string, ...interface{})
            Warn(string, ...interface{})
            Error(string, ...interface{})
            Fatal(string, ...interface{})

            SetLevel(int)
            SetFormatter(Formatter)

            IsDebug() bool
            IsInfo() bool
            IsWarn() bool
            // Error, Fatal have no guards, they SHOULD always log
        }

*   Standardizes on key-value pair argument sequence for machine parsing

    ```go
log.Debug("inside Fn()", "key1", value1, "key2", value2)

// instead of carpal tunnel syndrome logging
log.WithFields({logrus.Fields{"m":"mypkg", "key1": value1, "key2": value2}}).Debug("inside Fn()")
```
    logxi logs `FIX_IMBALANCED_PAIRS =>` if key-value pairs are imbalanced

*   Supports Color Schemes

    `log.New` creates a logger that supports color schemes

        logger := log.New("mylog")

    To customize scheme

        # emphasize errors with white text on red background
        LOGXI_COLORS="ERR=white:red" yourapp

## Configuration

### Enabling/Disabling Loggers

By default logxi logs entries whose level is `LevelWarn` or above when
using a terminal. For non-terminals, entries with level `LevelError` and
above are logged.

To quickly see all entries use short form

    # enable all, disable log named foo
    LOGXI=*,-foo yourapp

To better control logs in production, use long form which allows
for granular control of levels

    # the above statement is equivalent to this
    LOGXI=*=DBG,foo=OFF yourapp

`DBG` should obviously not be used in production unless for
troubleshooting. See `LevelAtoi` in `logger.go` for values.
For example, there is a problem in the data access layer
in production.

    # Set all to Error and set data related packages to Debug
    LOGXI=*=ERR,models=DBG,dat*=DBG,api=DBG yourapp

### Format

The format may be set via `LOGXI_FORMAT` environment
variable. Valid values are `"happy", "text", "JSON"`

    # Use JSON in production with custom time
	LOGXI_FORMAT=JSON,t=2006-01-02T15:04:05.000000-0700 yourapp
    # Use fast text formatter
    LOGXI_FORMAT=text yourapp

### Color Schemes

The color scheme may be set with `LOGXI_COLORS` environment variable. For
example, the default dark scheme is emulated like this

    # on non-Windows
    export LOGXI_COLORS=key=cyan+h,value,misc=blue+h,DBG,WRN=yellow+h,INF=green+h,ERR=red+h
    yourapp

    # on windows, stick to basic colors, styles liked bold, etc will not work
    SET LOGXI_COLORS=key=cyan,value,misc=blue,DBG,WRN=yellow,INF=green,ERR=red
    yourapp

    # color only errors
    LOGXI_COLORS=ERR=red yourapp

See [ansi](http://github.com/mgutz/ansi) package for styling. An empty
value, like "value" and "DBG" above means use default foreground and
background on terminal.

## Extending

What about hooks? Implement your own `io.Writer` to write to external
services and use `JSONFormatter`.

What about other writers? 12 factor apps only concern themselves with
STDOUT. Use shell redirection operators to write to file or create
a custom `io.Writer`.

What about formatting? Key-value pairs only.

## Testing

```
# install godo task runner
go get -u gopkg.in/godo.v1/cmd/godo

# install dependencies
godo install -v

# run test
godo test

# run bench with allocs (requires manual cleanup)
godo allocs
```

## License

MIT License
