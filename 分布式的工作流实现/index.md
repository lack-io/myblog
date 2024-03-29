# 分布式的工作流实现


本篇提供一个实现分布式工作流的思路。

系统组成部分：
- api (网关接口) : 为用户提供工作流的api接口
- discovery (服务发现) : 用于服务的注册和发现
- scheduler (调度中心) : 整体工作流的调度
- broker (消息队列) : 订阅发布和数据传输 
- node (工作节点) : 每个工作单元的提供者和执行者

实现思路：
1. node 启动时上传每个工作单元的基本信息: 包含工作单元输入输出、名称及其他内容
2. scheduler 保存这些信息并提供显示
3. scheduler 接收来自前端的数据并转化成工作流的信息，并执行

实现细节：
- scheduler 动态生成工作流信息，在 broker 中启动 topic 一个订阅者接收工作流的动态信息
- scheduler 根据工作单元信息寻找 node 并传输信息
- node 执行对应的工作单元发布信息到 broker 中工作流 topic
- scheduler 接收 topic 中的信息，如果中间发生错误，执行回滚操作
- node 需要的输入参数保存在 context.Context 中，返回参数则 publish 到 topic 中

```go
// 工作单元的接口
type WorkUnit interface {
  Do(ctx) error
  Undo(ctx) error
  String() string
}
```

```go
type workflow struct{
  units []WorkUnit
}

func Builder(ctx context.Context) *workflow {
  return &workflow{}
} 

func (w *workflow) Execute(ctx context.Context) error {
  undos := make([]func(ctx context.Context) error, 0)

  var err error
  defer func() {
    // 发生错误则逐个回滚
    if err != nil {
      for _, undo := range undos {
        undo()
      }
    }
  }

  for _, unit := range w.units {
    err = unit.Do()
    if err != nil {
      // 中间某步错误就结束
      return
    }
    // 回滚操作入栈
    undos = append(undos, unit.Undo)
  }
} 
```
