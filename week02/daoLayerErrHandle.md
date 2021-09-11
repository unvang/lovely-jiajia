#### 数据层的err需要wrap后返回吗？

- 举个例子：

未学本章前项目中的代码：

```
func GetUploadedFileList(clientId string) (*[]ClientLog, error) {
	var logList []ClientLog
	err := common.MysqlEngine.Where("client_id = ?", clientId).Find(&logList)
	if err != nil {
		log.Errorf("get uploaded file list err %v", err)
		return nil, err
	}
	return &logList, err
}
```

1. 看到上面返回了err，还打印了错误日志；err在调用方还会被打印，所以会打印多次；会造成性能浪费，排查起来也很麻烦

2. 排查问题时，也会发现返回的 err中的信息不够详细，还需要查log，对于运维是很痛苦的



##### 使用wrap后：

- 使用wrap后，对于dao层返回的err，上层调用接口可以直接返回，在前端或者第一级调用处打印

```
	rows, err := db.dbHandle.Query(sql_str)
	defer func() {
		if rows != nil {
			rows.Close()
		}
	}()
	if err != nil {
	//err返回后在main中写入logFile
		return items, errors.Wrap(err, "db err,table t_statis, func getCmdList")
	}
	...
	
```

- 看到的log：堆栈和附加的信息可以很快帮助定位问题

```
Error 1054: Unknown column 'heihei' in 'field list'
db err,table t_statis, func getCmdList
main.getItems
	/Users/work/Desktop/local/go-class/lovely-jiajia/week02/daoLayerErrHandle.go:207
main.main
	/Users/work/Desktop/local/go-class/lovely-jiajia/week02/daoLayerErrHandle.go:34
runtime.main
	/usr/local/go/src/runtime/proc.go:225
runtime.goexit
	/usr/local/go/src/runtime/asm_amd64.s:1371
```



- 不使用wrap，只返回err，对于业务层来讲，这些信息是不够定位问题的，需要加上额外的log信息，接口调用几次，就要添加几次，显然不如直接在 dao层wrap，业务调用时直接 return err

```
Error 1054: Unknown column 'heihei' in 'field list'
```



##### 总结

对于微服务场景，数据层往往和业务层分离，分别由不同的团队负责；

数据层往往会接入很多业务，log如果不清晰，排查问题非常麻烦；而每天几百T的日志量，也会占用很多资源

那么对于err handle，只打印一次获取到所需要的信息是很重要的，一小步的价值远远大于一步