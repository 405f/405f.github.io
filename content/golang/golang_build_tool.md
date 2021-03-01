---
title: "Golang_build_tool"
date: 2021-03-01T14:11:18+08:00
draft: false
tags: [golang]
---

## Build 
- compile packages and dependencies
 [定义](https://github.com/golang/go/blob/5ff7ec98b7727b3641df25200345b1aa50b6ff35/src/cmd/go/internal/work/build.go#L33)

- src/cmd/go/main.go 直接引入work 包
[import by ](https://github.com/golang/go/blob/5ff7ec98b7727b3641df25200345b1aa50b6ff35/src/cmd/go/main.go)

这个包暴露了 基本的内置命令
```go
func init() {
	base.Go.Commands = []*base.Command{
		bug.CmdBug,
		work.CmdBuild,
		clean.CmdClean,
		doc.CmdDoc,
		envcmd.CmdEnv,
		fix.CmdFix,
		fmtcmd.CmdFmt,
		generate.CmdGenerate,
		modget.CmdGet,
		work.CmdInstall,
		list.CmdList,
		modcmd.CmdMod,
		run.CmdRun,
		test.CmdTest,
		tool.CmdTool,
		version.CmdVersion,
		vet.CmdVet,

		help.HelpBuildConstraint,
		help.HelpBuildmode,
		help.HelpC,
		help.HelpCache,
		help.HelpEnvironment,
		help.HelpFileType,
		modload.HelpGoMod,
		help.HelpGopath,
		get.HelpGopathGet,
		modfetch.HelpGoproxy,
		help.HelpImportPath,
		modload.HelpModules,
		modget.HelpModuleGet,
		modfetch.HelpModuleAuth,
		help.HelpPackages,
		modfetch.HelpPrivate,
		test.HelpTestflag,
		test.HelpTestfunc,
		modget.HelpVCS,
	}
}

```
