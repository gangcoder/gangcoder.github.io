<!-- ---
title: Echo 指南
date: 2019-08-10 15:18:44
category: showcode, echo
--- -->

# Echo 使用指南

Echo 框架使用示例：

```go
package main

import (
	"net/http"

	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"
)

// Handler
func hello(c echo.Context) error {
	return c.String(http.StatusOK, "Hello, World!")
}

func main() {
	// Echo instance
	e := echo.New()

	// Middleware
	e.Use(middleware.Logger())
	e.Use(middleware.Recover())

	// Routes
	e.GET("/", hello)

	// Start server
	e.Logger.Fatal(e.Start(":1323"))
}
```

## 参考资料

- github.com/labstack/echo

