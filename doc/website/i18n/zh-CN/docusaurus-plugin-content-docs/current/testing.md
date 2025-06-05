---
id: testing
title: Testing
---

如果在单元测试中使用 `ent.Client`，可以通过生成的 `enttest` 包创建客户端并自动执行模式迁移，操作如下：

```go
package main

import (
	"testing"

	"<project>/ent/enttest"

	_ "github.com/mattn/go-sqlite3"
)

func TestXXX(t *testing.T) {
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&_fk=1")
	defer client.Close()
	// ...
}
```

如需向 `Open` 传递功能选项，请使用 `enttest.Option`：

```go
func TestXXX(t *testing.T) {
	opts := []enttest.Option{
		enttest.WithOptions(ent.Log(t.Log)),
		enttest.WithMigrateOptions(migrate.WithGlobalUniqueID(true)),
	}
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&_fk=1", opts...)
	defer client.Close()
	// ...
}
```