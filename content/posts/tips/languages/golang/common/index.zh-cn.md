---
title: Golang Tips - 记录 Golang 开发中遇到的小问题和最佳实践
date: 2022-09-04T23:00:00+08:00
lastmod: 2022-09-06T21:47:00+08:00
categories:
  - Tips
  - Golang
tags:
  - Tips
  - Golang
  - 最佳实践
---

- 使用 `testing.T.Parallel()` 来并行运行测试，同时可以及时发现**资源竞争问题**

```go
package serializer

import (
	"testing"
)

func TestWriteProtobufToBinaryFile(t *testing.T) {
	t.Parallel() 
	// ...
}
```

- 使用 [Testify](https://github.com/stretchr/testify) 来简化测试的 `Assertions` 和 `Mocks`
- `golangci-lint` 配置 `testpackage` 和 `paralleltest` 来启动 test package 名称检查和并行测试检查
  - `testpackage`: According to blackbox testing approach, you should not use unexported functions and methods from source code in tests.
- `math/rand` 包默认每次使用**相同**的随机种子，因此如需要，要在需要的地方设置随机种子 `rand.Seed(time.Now().UnixNano())`，如使用随机数的包的 `init()` 中
- [copier](https://github.com/jinzhu/copier)：一个便捷的 copy 库
- [evans](https://github.com/ktr0731/evans) gRPC 客户端，交互式调用 gRPC server
