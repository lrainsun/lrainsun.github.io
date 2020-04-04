---
layout: post
title:  "配置sublime的go编译开发环境"
date:   2020-01-15 20:45:00 +0800
categories: Go
tags: Go-Learning
excerpt: 配置sublime的go编译开发环境
mathjax: true
typora-root-url: ../
---

# 安装Sublime

goland木有license了，之前的license过期了。。老板的经费还木有申请下来，强迫症的人是忍受不了的，尽管我最近没怎么看go代码。。

所以，google了一下，可以sublime里搞一个小的go开发IDE

首先安装sublime，我原来用得是sublime2，下次又下载了个sublime3，好像差别不大，感觉上界面稍微洋气一丢丢

# Package control

然后安装package control (sublime text 的插件管理工具)

下载package control https://packagecontrol.io/Package%20Control.sublime-package

然后browse packages会打开packages所在文件夹，点到Installed Packages，把刚刚下载的文件拷贝进去，然后重启sublime

![image-20200115152903444](/assets/images/image-20200115152903444.png)

# Go build

然后安装go build

Preferences *>* Package control 菜单(MAC快捷键 `shift + command + p`)

在弹出的输入框输入`install` 选择Package control:install package，这一步会load repository会需要一点时间，等待一下

然后输入`Golang build` 选择Golang build 安装

安装好之后点击 Preferences Preferences *>* Package Setting *>* Golang Config *>* Setting - User 设置go安装路径和GOPATH

```yaml
{
	"PATH": "/usr/local/Cellar/go/1.13.3/bin",
	"GOPATH": "/Users/minsu/Documents/go"
}
```

这个时候可以用command + shift + b，然后go run, go get啥的了

我记得操作的时候，好像还有个啥错的，可是我记不起来了。。。

# gosublime

GoSublime 是一个交互式的go build 工具，使用起来也是很方便，主要配合Golang build使用。

首先下载gosublime git clone https://margo.sh/GoSublime，把gosumlime文件夹拷贝到Packages下（Preferences *>* Browse Packages），然后重启sublime。

修改gosublime配置Preferences > Package Settings > GoSublime > Settings - Default

```yaml
	"use_gs_gopath": true,
```

自动补全Preferences > Package Settings > GoSublime > Settings - User

```yaml
{
	"auto_complete": true,
	"auto_match_enabled": true
}
```

这时会有个错误提示：`*Not* using margo extension: Error: cannot find package "margo" in any of` 是因为gosublime还不支持mod

依次按 command + “.” and command +“x” 来打开margo.go 这个文件，然后用以下的内容进行替换，最后重启sublime

```go
package margo

import (
    "margo.sh/golang"
    "margo.sh/mg"
    "time"
)

// Margo is the entry-point to margo
func Margo(m mg.Args) {
    // See the documentation for `mg.Reducer`
    // comments beginning with `gs:` denote features that replace old GoSublime settings

    // add our reducers (margo plugins) to the store
    // they are run in the specified order
    // and should ideally not block for more than a couple milliseconds
    m.Use(
        // MOTD keeps you updated about new versions and important announcements
        //
        // It adds a new command `motd.sync` available via the UserCmd palette as `Sync MOTD`
        //
        // Interval can be set in order to enable automatic update fetching.
        //
        // When new updates are found, it displays the message in the status bar
        // e.g. `★ margo.sh/cl/18.09.14 ★` a url where you see the upcoming changes before updating
        //
        // It sends the following data to the url https://api.margo.sh/motd.json:
        // * current editor plugin name e.g. `?client=gosublime`
        //   this tells us which editor plugin's changelog to check
        // * current editor plugin version e.g. `?tag=r18.09.14-1`
        //   this allows us to determine if there any updates
        // * whether or not this is the first request of the day e.g. `?firstHit=1`
        //   this allows us to get an estimated count of active users without storing
        //   any personally identifiable data
        //
        // No other data is sent. For more info contact privacy at kuroku.io
        //
        &mg.MOTD{
            // Interval, if set, specifies how often to automatically fetch messages from Endpoint
            // Interval: 3600e9, // automatically fetch updates every hour
        },

        mg.NewReducer(func(mx *mg.Ctx) *mg.State {
            // By default, events (e.g. ViewSaved) are triggered in all files.
            // Replace `mg.AllLangs` with `mg.Go` to restrict events to Go(-lang) files.
            // Please note, however, that this mode is not tested
            // and saving a non-go file will not trigger linters, etc. for that go pkg
            return mx.SetConfig(mx.Config.EnabledForLangs(
                mg.AllLangs,
            ))
        }),

        // Add `go` command integration
        // this adds a new commands:
        // gs: these commands are all callable through 9o:
        // * go: Wrapper around the go command, adding linter support
        // * go.play: Automatically build and run go commands or run go test for packages
        //   with support for linting and unsaved files
        // * go.replay: Wrapper around go.play limited to a single instance
        //   by default this command is bound to ctrl+.,ctrl+r or cmd+.,cmd+r
        //
        // UserCmds are also added for `Go Play` and `Go RePlay`
        &golang.GoCmd{},

        // add the day and time to the status bar
        &DayTimeStatus{},

        // both GoFmt and GoImports will automatically disable the GoSublime version
        // you will need to install the `goimports` tool manually
        // https://godoc.org/golang.org/x/tools/cmd/goimports
        //
        // gs: this replaces settings `fmt_enabled`, `fmt_tab_indent`, `fmt_tab_width`, `fmt_cmd`
        //
        // golang.GoFmt,
        // or
        // golang.GoImports,

        // Configure general auto-completion behaviour
        &golang.MarGocodeCtl{
            // whether or not to include Test*, Benchmark* and Example* functions in the auto-completion list
            // gs: this replaces the `autocomplete_tests` setting
            ProposeTests: false,

            // Don't try to automatically import packages when auto-compeltion fails
            // e.g. when `json.` is typed, if auto-complete fails
            // "encoding/json" is imported and auto-complete attempted on that package instead
            // See AddUnimportedPackages
            NoUnimportedPackages: false,

            // If a package was imported internally for use in auto-completion,
            // insert it in the source code
            // See NoUnimportedPackages
            // e.g. after `json.` is typed, `import "encoding/json"` added to the code
            AddUnimportedPackages: false,

            // Don't preload packages to speed up auto-completion, etc.
            NoPreloading: false,

            // Don't suggest builtin types and functions
            // gs: this replaces the `autocomplete_builtins` setting
            NoBuiltins: false,
        },

        // Enable auto-completion
        // gs: this replaces the `gscomplete_enabled` setting
        &golang.Gocode{
            // show the function parameters. this can take up a lot of space
            ShowFuncParams: true,
        },

        // show func arguments/calltips in the status bar
        // gs: this replaces the `calltips` setting
        &golang.GocodeCalltips{},

        // use guru for goto-definition
        // new commands `goto.definition` and `guru.definition` are defined
        // gs: by default `goto.definition` is bound to ctrl+.,ctrl+g or cmd+.,cmd+g
        &golang.Guru{},

        // add some default context aware-ish snippets
        // gs: this replaces the `autocomplete_snippets` and `default_snippets` settings
        golang.Snippets,

        // add our own snippets
        // gs: this replaces the `snippets` setting
        MySnippets,

        // check the file for syntax errors
        // gs: this and other linters e.g. below,
        //     replaces the settings `gslint_enabled`, `lint_filter`, `comp_lint_enabled`,
        //     `comp_lint_commands`, `gslint_timeout`, `lint_enabled`, `linters`
        &golang.SyntaxCheck{},

        // Add user commands for running tests and benchmarks
        // gs: this adds support for the tests command palette `ctrl+.`,`ctrl+t` or `cmd+.`,`cmd+t`
        &golang.TestCmds{
            // additional args to add to the command when running tests and examples
            TestArgs: []string{},

            // additional args to add to the command when running benchmarks
            BenchArgs: []string{"-benchmem"},
        },

        // run `go install -i` on save
        // golang.GoInstall("-i"),
        // or
        // golang.GoInstallDiscardBinaries("-i"),
        //
        // GoInstallDiscardBinaries will additionally set $GOBIN
        // to a temp directory so binaries are not installed into your $GOPATH/bin
        //
        // the -i flag is used to install imported packages as well
        // it's only supported in go1.10 or newer

        // run `go vet` on save. go vet is ran automatically as part of `go test` in go1.10
        // golang.GoVet(),

        // run `go test -race` on save
        // golang.GoTest("-race"),

        // run `golint` on save
        // &golang.Linter{Name: "golint", Label: "Go/Lint"},

        // run gometalinter on save
        // &golang.Linter{Name: "gometalinter", Args: []string{
        //  "--disable=gas",
        //  "--fast",
        // }},
    )
}

// DayTimeStatus adds the current day and time to the status bar
type DayTimeStatus struct {
    mg.ReducerType
}

func (dts DayTimeStatus) RMount(mx *mg.Ctx) {
    // kick off the ticker when we start
    dispatch := mx.Store.Dispatch
    go func() {
        ticker := time.NewTicker(1 * time.Second)
        for range ticker.C {
            dispatch(mg.Render)
        }
    }()
}

func (dts DayTimeStatus) Reduce(mx *mg.Ctx) *mg.State {
    // we always want to render the time
    // otherwise it will sometimes disappear from the status bar
    now := time.Now()
    format := "Mon, 15:04"
    if now.Second()%2 == 0 {
        format = "Mon, 15 04"
    }
    return mx.AddStatus(now.Format(format))
}

// MySnippets is a slice of functions returning our own snippets
var MySnippets = golang.SnippetFuncs(
    func(cx *golang.CompletionCtx) []mg.Completion {
        // if we're not in a block (i.e. function), do nothing
        if !cx.Scope.Is(golang.BlockScope) {
            return nil
        }

        return []mg.Completion{
            {
                Query: "if err",
                Title: "err != nil { return }",
                Src:   "if ${1:err} != nil {\n\treturn $0\n}",
            },
        }
    },
)
```

# CTags

为了实现代码跳转，还需安装ctags，下载https://sourceforge.net/projects/ctags/

```shell
tar xzvf ctags-5.8.tar.gz

cd ctags-5.8

./configure

make

sudo make install
```

然后安装sublime ctags插件，shift+command+p，install packages > ctags

修改配置

Preferences > Package Settings > CTags > Settings - User，把Settings-Default 的内容全部复制到Settings-User，然后修改如下配置，command路径是ctags的可执行目录，安装完ctags会提示的

```yaml
    "autocomplete": true,
    "command": " /usr/local/bin/ctags",
```

这时你在打开的文件中，右键菜单中会多一个Navigate to Definition菜单项，左侧的工程/项目文件上右键会看到CTags: Rebuild Tags菜单项

首先在左侧执行CTags: Rebuild Tags，再到右边代码中就可以实现跳转了

如果rebuild tags的时候提示：

```shell
resort_ctags
    keys.setdefault(line.split('\t')[FILENAME], []).append(line)
IndexError: list index out of range
```

说明文件夹里内容太多了，可以选一个小一点的文件夹，如果成功会在目录下成功两个文件，然后就可以实现跳转啦

![image-20200115163830251](/assets/images/image-20200115163830251.png)

# run

最后使用shift + command + b，然后用gosublime，就会跳出交互式的窗口，go get , go install 、go build 、go clean都可以

# References

[1] [https://www.cnblogs.com/soundcode/p/9552876.html](https://www.cnblogs.com/soundcode/p/9552876.html)

[2] [https://margo.sh/b/migrate](https://margo.sh/b/migrate)

[3] [https://zhuanlan.zhihu.com/p/70624718](https://zhuanlan.zhihu.com/p/70624718)