# Go 基于中间件的资源权限校验及多线程优化

工作中遇到的需求，需要为资源添加权限校验，根据用户、资源类型、资源id、所需权限，判断用户是否有权限。

权限系统已有现成的，是RBAC模型。用户信息已经通过一个中间件存入context中。

只需要为接口设计一套可以高度复用的权限校验系统即可。

## 需求设计
希望尽可能可以高度复用，可以针对每个接口定制资源类型、所需权限、从请求信息中获取资源id。

因此采用中间件的形式实现，使用方式如下所示。参数分别为资源类型、所需权限、资源id参数在哪里（Path、Query、Body等）、资源id路径（即键值，对于body中json数据可能会有几层嵌套，因此采用slice类型）。

```go
group.GET("/:id",
  ResourcePermissionHandler(ResourceTypeApp, PermissionRead, ParamTypePath, []string{"id"}),
  c.Get)
group.POST("/update",
  ResourcePermissionHandler(ResourceTypeApp, PermissionWrite, ParamTypeBody, []string{"data", "id"}),
  c.Update)
group.POST("/delete",
  ResourcePermissionHandler(ResourceTypeApp, PermissionOwn, ParamTypeQuery, []string{"id"}),
  c.Delete)
```

## 中间件
```go
// 增加资源权限校验
//
//  resourceType: 资源类型,
//  permission: 权限,
//  resourceIdParamType: 资源ID参数类型,
//  resourceIdParamPath: 资源ID参数路径, 例如: ["id"], ["data", "id"].
func ResourcePermissionHandler(
    resourceType ResourceType,
    permission Permission,
    resourceIdParamType ParamType,
    resourceIdParamPath []string,
) gin.HandlerFunc {
  return func(c *gin.Context) {
    id, err := getResourceId(c, resourceIdParamType, resourceIdParamPath) // 获取资源ID
    if err != nil {
      ctl.AbortWithCode(c, 500, errors.Wrap(err, "get resource id failed")) // 返回中写入错误信息
      return
    }
    // 校验权限
    var ok bool
    switch resourceType {
    case ResourceTypeAppInstance:
      ok, err = checkAppInstance(c, id, permission) // instance资源需要特殊处理
    default:
      ok, err = CheckResourcePermission(c, resourceType, id, permission) // 校验权限, 调用RBAC模型权限系统
    }
    if err != nil {
      ctl.AbortWithCode(c, 500, errors.Wrap(err, "check resource permission failed")) // 返回中写入错误信息
      return
    }
    if !ok {
      ctl.AbortWithCode(c, 403, errors.New("permission denied")) // 返回中写入错误信息
      return
    }
    c.Next() // 调用下一个Handler
  }
}
```

## 获取资源id
根据参数类型和路径获取资源ID。

对于Header、Path、Query参数，一般没有嵌套结构，直接使用idParamPath中第一个key获取即可。

对于Body，通常是JSON格式的数据，可能有嵌套结构。因此遍历idParamPath，按照key一层一层取，直到获取到资源id。

```go
// 根据参数类型和路径获取资源ID
func getResourceId(c *gin.Context, idParamType ParamType, idParamPath []string) (uint, error) {
  if len(idParamPath) == 0 {
    return 0, errors.New("resource id param path is required")
  }
  switch idParamType {
  case ParamTypeHeader:
    return 0, errors.New("resource id param type Header is unimplemented")
  case ParamTypePath:
    id, err := strconv.ParseUint(c.Param(idParamPath[0]), 10, 64)
    if err != nil {
      return 0, err
    }
    return uint(id), nil
  case ParamTypeQuery:
    id, err := strconv.ParseUint(c.Query(idParamPath[0]), 10, 64)
    if err != nil {
      return 0, err
    }
    return uint(id), nil
  case ParamTypeBody:
    var data any
    if err := c.ShouldBindJSON(&data); err != nil {
      return 0, err
    }
    for _, path := range idParamPath {
      m, ok := data.(map[string]any)
      if !ok {
        return 0, errors.New("resource id param path is invalid")
      }
      data = m[path]
    }
    id, ok := data.(float64)
    if !ok {
      return 0, fmt.Errorf("resource id param type conversion failed, data type: %T", data)
    }
    return uint(id), nil
  default:
    return 0, errors.New("resource id param type is invalid")
  }
}
```

## 多线程优化
还有个特别的需求，instance是属于某个app的，对于instance，用户如果有对应的app的权限，那也是可以通过的。

所以需要调用2次权限系统。很明显这是可以通过并行执行来优化的。最简单的方法是goroutine并行执行，使用sync.WaitGroup等待2次请求完成。
```go
type result struct {
  ok  bool
  err error
}
goNum := 2      // 任务goroutine数量
cCp := c.Copy() // 创建在 goroutine 中使用的副本
ch := make(chan result, goNum)
defer close(ch) // 关闭channel
var wg sync.WaitGroup

wg.Add(goNum)
go func() {
  defer wg.Done()
  app, err := instanceSvc.GetInstanceByID(cCp, id)
  if err != nil {
    ch <- result{false, err}
    return
  }
  ok, err := CheckResourcePermission(cCp, ResourceTypeApp, app.ApplicationID, permission) // 校验权限, 调用RBAC模型权限系统
  if err != nil {
    ch <- result{false, err}
    return
  }
  ch <- result{ok, nil}
}()
go func() {
  defer wg.Done()
  ok, err := CheckResourcePermission(cCp, ResourceTypeAppInstance, id, permission) // 校验权限, 调用RBAC模型权限系统
  if err != nil {
    ch <- result{false, err}
    return
  }
  ch <- result{ok, nil}
}()

wg.Wait() // 等待所有任务 goroutine 执行完毕
// 处理结果
```

这里还有很多可以优化的点：
1. 不需要等待2次调用都完成，只要有一个返回是true就可以通过了。
2. 通过for + select 获取channel中的返回值，只要有一个为true，就直接通过。
3. 因为主线程不再等待2个任务完成，需要额外的一个线程等待任务执行，回收资源。
4. 增加超时机制。

```go
// 校验应用实例权限
// 同时检查应用和应用实例权限, 任意一个通过则通过
func checkAppInstance(c *gin.Context, id uint, permission Permission) (bool, error) {
  type result struct {
    ok  bool
    err error
  }
  goNum := 2      // 任务goroutine数量
  cCp := c.Copy() // 创建在 goroutine 中使用的副本
  ch := make(chan result, goNum)
  var wg sync.WaitGroup

  wg.Add(goNum)
  go func() {
    defer wg.Done()
    defer func() {
      if r := recover(); r != nil {
        log.Errorf("check permission AppInstance panic: %v", r)
      }
    }()
    app, err := instanceSvc.GetInstanceByID(cCp, id)
    if err != nil {
      ch <- result{false, err}
      return
    }
    ok, err := CheckResourcePermission(cCp, ResourceTypeApp, app.ApplicationID, permission) // 校验权限, 调用RBAC模型权限系统
    if err != nil {
      ch <- result{false, err}
      return
    }
    ch <- result{ok, nil}
  }()
  go func() {
    defer wg.Done()
    defer func() {
      if r := recover(); r != nil {
        log.Errorf("check permission AppInstance panic: %v", r)
      }
    }()
    ok, err := panshiSvc.CheckResourcePermission(cCp, ResourceTypeAppInstance, id, permission) // 校验权限, 调用RBAC模型权限系统
    if err != nil {
      ch <- result{false, err}
      return
    }
    ch <- result{ok, nil}
  }()
  // 额外的线程，等待并回收资源
  go func() {
    wg.Wait() // 等待所有任务 goroutine 执行完毕
    close(ch) // 关闭channel
  }()

  // 获取结果, 任意一个返回true则通过
  // 最多接受goNum个结果
  var err error
  timeout := time.NewTimer(5 * time.Second) // 超时定时器
  defer timeout.Stop()
  for i := 0; i < goNum; i++ {
    select {
    case result := <-ch:
      if result.ok {
        return true, nil // 任意一个通过则返回true
      }
      if result.err != nil {
        err = result.err // 记录错误
      }
    case <-timeout.C:
      return false, errors.New("check permission AppInstance timeout") // 超时
    }
  }
  return false, err
}
```
