```go
package web

import (
	"encoding/json"
	"github.com/gorilla/mux"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"net/http"
	"tencent.com/devops/registry/pkg/logger"
)

func GetLogLevelHandler(rw http.ResponseWriter, _ *http.Request) {
	rw.WriteHeader(http.StatusOK)
	var err = json.NewEncoder(rw).Encode(map[string]string{"level": logger.GetLevel().String()})
	if err != nil {
		zap.L().Error("get log level error", zap.Error(err))
	}
}

func SetLogLevelHandler(rw http.ResponseWriter, r *http.Request) {
	var levelStr = r.URL.Query().Get("level")
	var old = logger.GetLevel()
	var lv = new(zapcore.Level)
	var err = lv.Set(levelStr)
	if err != nil {
		rw.WriteHeader(http.StatusBadRequest)
		err = json.NewEncoder(rw).Encode(map[string]string{"error": err.Error()})
		if err != nil {
			zap.L().Error("write response error", zap.Error(err))
		}
		return
	}

	logger.SetLevel(*lv)
	zap.L().Info("Update Log Level", zap.Stringer("old_level", old), zap.Stringer("new_level", lv))
	rw.WriteHeader(http.StatusOK)
	err = json.NewEncoder(rw).Encode(map[string]string{"level": logger.GetLevel().String()})
	if err != nil {
		zap.L().Error("write response error but level has changed to: "+levelStr, zap.Error(err))
	}
}

func RunDebugServer(addr string) {
	var r = mux.NewRouter()
	var s = r.Path("/debug/log/level").Subrouter()
	s.Methods("GET").HandlerFunc(GetLogLevelHandler)
	s.Methods("POST").HandlerFunc(SetLogLevelHandler)

	if addr == "" {
		addr = "127.0.0.1:8090"
	}
	var server = &http.Server{Addr: addr, Handler: r}
	var err = server.ListenAndServe()
	if err != nil {
		zap.L().Error("debug server exist with error", zap.Error(err))
	}
}
```

https://www.jianshu.com/p/c8c1a960a239

​	从demo的RunDebugServer可以看到，`NewRouter`、`Path`、`Methods`、`HandlerFunc`和`ListenAndServe`是主要的几个函数。前4个函数是将你的API接口注册到r这个Router对象中，后者则是监听和提供服务。

- 自己的接口如何提供服务？
- 