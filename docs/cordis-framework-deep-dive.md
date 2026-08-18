# Cordis 框架深度解析
Deepseek Harness，为什么选择Cordis？

Cordis 是一个**插件式元框架**：你把一个个插件像积木一样组合在一起，拼出一个完整的应用。每块积木可以随时拆掉，换掉，拆掉时它留下的所有东西都会自动清理干净。

在Deepseek Harness中，传统Agent框架中写好的Agent loop、工具注册表、记忆模块、Adapter, 都是插件。

# 第零部分 — Cordis 要解决什么问题

## 组合问题

Cordis 围绕一个问题展开:**如何让独立的、可复用的能力单元(plugin)在一个共享上下文里既能看见彼此,也能被可控地独立加载/卸载/重载,且整个生命周期对调用者透明?**

这个问题隐藏三个相互耦合的子问题:

1. **可见性(依赖)**:plugin A 需要 plugin B 的能力,但又不想在编译期硬编码具体实现。
2. **时序(生命周期)**:plugin 的加载顺序未定;插件可能要异步加载,必须等待依赖、必须优雅卸载、必须不泄漏地重载。
3. **空间(隔离)**:同一个服务名在不同场景(测试 vs 生产、本地 vs 远程)下指向不同实现;它们互不污染。

想象一下,传统软件工程中,当你需要上线一个新服务的时候,你需要：先停机,写代码,合代码,部署发布,上线测试；为了避免架构大改,因此在设计之初就要尽可能解耦,避免逻辑耦合导致架构演进困难。

而Cordis的解决方案是**运行时加载+服务可逆**。即不用停机,就可以在运行时加载、卸载、重载插件。而每个插件在卸载时,会干净地、完整地把自己引入的所有内容都清理掉。

经典 DI 框架(Guice、Spring、NestJS)在 (1) 上做得好。(2) 和 (3) 通常需要手写 `start`/`stop` 钩子加配置 profile —— 当 plugin 数量大、重载频繁、HMR 介入时,这种手工钩子会脆裂。

## Cordis 的统一答案

> 把"plugin 的存在"和"plugin 的清理"做成同一个可组合对象(`Disposable`);把它挂在一个生命周期节点(`Fiber`)上;让可见性走**原型链**加一个 **Symbol** 键控的**隔离映射**;让**协调**走一个**五模态事件分发器**。

换句话说:**Cordis 用一对原语(Disposable + 原型链)同时表达依赖、生命周期和隔离**,避免"DI 框架"与"生命周期框架"的概念分裂。

---

# 第一部分 — 基础概念

## 一、10 个基础概念

### 1. Context

**用法预览**(6-8 行)
  const root = new Context()
  const child = root.extend()
  child.provide('fs', new LocalFileSystem(...))
  child.fs
  root.fs
  child.fs.readFile('x.txt')

**定义**:JS 对象,其原型链即 scope 继承链;自身挂载 4 个内置服务(events / logger / reflect / registry)和一个 fiber。

**设计初衷**:不发明新 DI 词汇,让读服务看起来像读属性。

**什么是原型链？**

JavaScript 的对象继承机制。每个 JS 对象都有一个隐藏的"父对象"链接,叫 prototype。读属性时找不到,JS 就顺着这个链接往上找,直到找不到为止。这条从对象出发,沿 prototype 一路向上的链,就叫"原型链"。

用最简例子建立直觉
```
const parent = { greeting: 'hello' }
const child = Object.create(parent)   // ← 关键:child 的"父对象"是 parent
child.name = 'Alice'

console.log(child.name)       // 'Alice'  — child 自身有这个属性
console.log(child.greeting)   // 'hello'  — child 没有,但 parent 有,JS 顺着链找到
console.log(child.color)      // undefined — 链走到 null 都找不到
```

Object.create(parent) 这一行的含义:*"创建一个新空对象,把这个新对象的 prototype 指向 parent"*。这个新对象就叫 child,它的"父亲"是 parent。
读属性的查找顺序(JS 引擎规则)
读 child.xxx 时,JS 按这个顺序查:
1. child 自身有没有 xxx? → 有就用,没有继续
2. child 的 prototype(也就是 parent)有没有 xxx? → 有就用,没有继续
3. parent 的 prototype(Object.prototype)有没有 xxx? → 有就用,没有继续
4. Object.prototype 的 prototype 是 null → 彻底找不到,返回 undefined
这条从对象出发,沿 prototype 一路向上的链,就叫"原型链"。

Cordis 的 Context 树是这样构造的:
class Context {
  constructor() {
    /* ... */
  }
  extend(meta = {}) {
    const self = Object.create(getTraceable(this, this))   // ← 关键
    for (const prop of Reflect.ownKeys(meta)) {
      Object.defineProperty(self, prop, Reflect.getOwnPropertyDescriptor(meta, prop)!)
    }
    return self
  }
}
extend() 的核心就是 Object.create(parent)。这意味着:
const root = new Context()       // root 有 events / logger / reflect / registry
const child = root.extend()      // child 的 prototype 是 root
const grandchild = child.extend()  // grandchild 的 prototype 是 child
读 grandchild.logger 时,JS 自动沿 prototype 链:
- grandchild 自身有没有 logger? 没有
- grandchild 的 prototype 是 child,child 有没有? 没有(我们没注册)
- child 的 prototype 是 root,root 有吗? 有!
- 返回 root 上挂的那个 logger service
整个查找过程不需要 Cordis 写一行"lookup service"的代码——JS 引擎自己完成。

**Cordis 为什么这么选**
1. 零新词汇:scope / inheritance / lookup 都是 JS 已经有的概念,作者不用学新东西
2. 零额外开销:JS 引擎的 prototype lookup 是 C++ 级别优化
3. shadowing 天然支持:在 child 上设同名 own property,自动覆盖 parent——Cordis 的 intercept() / isolate() 用这一性质
4. Proxy 增强:Cordis 在 Context 上加了 Proxy handler,使得"找不到的属性"自动 fallback 到 service store 查找——prototype 链查完了 JS 没找到,Cordis 才接手
一个 Cordis 真实场景
```
const root = new Context()
const child = root.extend()
child.provide('fs', new LocalFileSystem(...))

// 读 root.fs
// 1. root 自身有 fs 吗? 没有
// 2. root 的 prototype 是 Object.prototype,有 fs 吗? 没有
// 3. JS 走完链,放弃
// 4. ReflectService.handler.get trap 接管
// 5. trap 发现 root 没注入 fs,但 child 注入了——
//    Proxy 不会"往上"找,但 ctx 本身的 fiber 是同一个 root
//    所以 root 的 ReflectService 也能看到 child 的 provide?
//    —— 实际上不对,provide 是注册到 root 的 store 上,
//    因此 root.fs 也能读到,这是 fiber 链的另一回事
```

注意最后一点:prototype 链管不了 service store 的查找,store 查找走的是 fiber 链。这两条链在 Cordis 里是两件事:
- prototype 链 = JS 原生对象继承(管 own property + 静态定义的服务)
- fiber 链 = ctx.fiber.parent.fiber.parent.fiber...(管 service store 查找)
Cordis 把这两条链组合起来用——prototype 链负责"哪些属性我能读",fiber 链负责"这些属性对应的 service 实现从哪来"。
fiber链后续我们再详细展开

**为什么是 JS 的原型链,而不是其他语言的继承**
一句话解释：Java / C++ / Python / C# 的继承都是类继承(class-based inheritance),无法在运行时动态改变。
JS 的原型链是对象继承(object-based inheritance)——不是类之间的关系,是对象之间的关系：任何对象可以在运行时改变自己的 prototype；任何对象可以在运行时给 prototype 加属性,这符合Cordis的设计初衷：运行时动态改变服务。

具体场景:
1. Context.extend() 运行时建父子
class Context {
  extend(meta = {}) {
    const self = Object.create(getTraceable(this, this))   // ← 运行时建子
    // ...
  }
}
你调 ctx.extend() 时,Cordis 当场建一个新对象,挂到 prototype 链上。在 Java 里这等于"运行时凭空造一个类的子类"——做不到。在 JS 里就是一行。

2. provide() 运行时挂服务
child.provide('fs', new LocalFileSystem(...))
provide 在 root context 的 service store 里插入一条实现,这个 store 本身是个普通对象,运行时可改。Java 里的 Spring container 要做类似事,得用反射 + 字节码编织 + 重启 classloader——复杂且脆弱。

3. intercept() 运行时叠配置
ctx.intercept('llm', { maxTokens: 1000 })
每次 intercept 都在 prototype 链上加一层 mapping——运行时累加,运行时撤销(disposer)。Java 里的类似操作要么"重启应用",要么用 AOP 编织——都不是"自然"的写法。

4. HMR(代码热重载)
vendor/hmr/src/index.ts 监听文件,改完直接 reload fiber。整个 prototype 链可重建——因为对象关系在 JS 里就是数据,改完重新 Object.create 即可。Java 里的 hot reload 要 JVMTI + 自定义 classloader,本质上是"绕开 JVM 的限制"。

5. Isolate 运行时建独立 scope
const isolated = ctx.isolate('meeting-room')
isolate 创建一个新 context,Symbol 作 service key——两个 Symbol 在同一对象上可以独立存活。类继承做不到"同名属性有多个版本",prototype 链 + Symbol 天然支持。

当然,存在语言具备类似的能力,但因为语言生态或者功能不完善,所以没有被广泛使用：
Python 有 __mro__ 多继承 + 动态类创建(type() 元类),能模拟一部分。但 Python 类的元类是固定的,运行时改 MRO 是反模式,社区不推荐。
Lua 有 metatable,机制上和 JS prototype 几乎一样——Lua 也能写 Cordis。Cordis 没选 Lua 是生态原因(没有 npm、TypeScript、Web 平台原生集成)。
Ruby 有 singleton class / eigenclass,部分支持动态扩展,但每次开新对象都要走 class system。
Self 语言——JS 的 prototype 链就是从 Self 抄的。Self 是第一个把"对象继承对象"做到极致的语言。

**深挖一层：为什么 JS 原型链可以在运行时继承对象**
根本原因:JS 对象结构层面的设计
JS 对象本质上就是两个东西:
```
const obj = { x: 1 }
// 内部上,obj 实际是:
// {
//   [[Prototype]]: Object.prototype,   ← 一个普通的可读写槽
//   x: 1                                ← 普通属性
// }
```
[[Prototype]] 是一个数据槽,不是固化的元数据
JS 把"父对象引用"设计成运行时可写的数据,这是从语言设计层就定下的——不是引擎优化,不是语法糖,是核心数据结构就是这样的。

**Scope 是怎么"工作"的**
```
root.provide('fs', rootFs)        // 在 root scope 注册 fs
child.provide('auth', childAuth)  // 在 child scope 注册 auth

root.fs      // rootFs     ← 自己 scope 注册的
root.auth    // undefined  ← child scope 注册的,root 看不见

child.fs     // rootFs     ← 继承自 root scope
child.auth   // childAuth  ← 自己 scope 注册的

grandchild.fs    // rootFs     ← 沿 prototype 链继承到 root
grandchild.auth  // childAuth  ← 沿 prototype 链继承到 child
```
Scope 继承是单向向下的:child 看得见 parent,parent 看不见 child。这个符合大家对继承的一贯理解

### 2. Plugin

**定义**:可被加载的能力单元,三种形状(函数 / 类 / 对象)。

**设计初衷**:三种形状对应不同写作风格——函数最小、类适合 OO 继承、对象适合带元数据。
框架归一化这三种形状,统一加载和元数据协议。

**什么是 plugin(插件)**
Plugin = 一段往 ctx 上注册东西的代码。例:
```
const myLoggerPlugin = (ctx, config) => {
  ctx.on('request-start', () => console.log('request started'))
  setInterval(() => console.log('tick'), 1000)
  ctx.provide('logger', { info: (msg) => console.log(msg) })
}
```
这段代码本身是死的(只写在那里,没跑)。plugin 必须被加载才会执行。

并且，同一个 plugin 可以被加载多次
```
ctx.plugin(myLoggerPlugin, { name: 'logger-A' })  // 加载第一次
ctx.plugin(myLoggerPlugin, { name: 'logger-B' })  // 加载第二次
两次加载跑的是同一段代码,但两次是完全独立的实例——各自的 config、各自的副作用、各自的生命周期。
如果只卸载 logger-A,logger-B 完全不受影响。
```

**什么是副作用**
一句话解释：一个函数除了"算一个返回值"以外,对外部世界做的任何事。任何超出本职的事——写终端、改全局变量、启动 timer——都是可以认为是副作用。
比如：
ctx.on(...) 注册了一个 listener。
setInterval(...) 启动了一个 timer。
ctx.provide(...) 注册了一个 service。
这些都是副作用。如果 fiber A 卸载时不清理:
- listener 还在响应事件 → 内存泄漏 + 行为异常
- timer 还在跑 → 持续输出
- service 还在 store 里 → 别的 fiber 还在用它

### 3. Fiber

**定义**:plugin 的运行时实例 + 副作用集合 + 生命周期状态机。

**设计初衷**:一个 plugin 可被加载多次,每次加载需要独立的生命周期和副作用隔离。
把"运行时实例"和"它产生的所有副作用"绑在一起,就能一次 unload 一并清理。

Fiber = "一个 plugin 的一次加载实例",它拥有:
- 这次加载的 config
- 这次加载产生的所有副作用(listener / timer / service / 等)
- 自己的生命周期状态机

**Fiber 的关键事实**
1. 一个 plugin 对应一个 Fiber,但多次加载对应多个 Fiber
ctx.plugin(myPlugin, { ... })  // Fiber 1
ctx.plugin(myPlugin, { ... })  // Fiber 2
两个 Fiber,跑同一段 plugin 代码,但独立。

2. Fiber 的"_disposables"是副作用集合
// fiber.ts:202-203
public readonly _disposables = new DisposableList<Disposable>()
每个 fiber 持有这个 list。Plugin body 注册的每项副作用都进 list。
// fiber.ts:520
removeWrapper = this._disposables.push(wrapper)
注册即加入。注册 = effect,disposer 进 fiber 的 disposables。

3. Fiber 有生命周期状态机
// fiber.ts:147-154
const enum FiberState {
  PENDING,    // 等待依赖
  LOADING,    // plugin body 正在跑
  ACTIVE,     // 正常运行
  FAILED,     // 启动失败
  UNLOADING,  // 正在撤销
  DISPOSED,   // 已删除
}
状态转移:
PENDING → LOADING → ACTIVE       正常启动
ACTIVE   → UNLOADING → DISPOSED  卸载
ACTIVE   → UNLOADING → ACTIVE    重载(配置变了)
任何态 → FAILED                   启动报错

4. Fiber 是 Context 的"住户"
Fiber 跟 Context 一一对应:
- 一个 Context 有一个 fiber 字段
- 一个 fiber 有一个 ctx 字段
- Fiber 是"plugin 在这个 context 里住着的状态"

**为什么叫"Fiber"**
这个词来自 OS 调度术语。Fiber = 轻量级线程——有自己的执行状态,但合作式调度(不是抢占式)。
Cordis 的 plugin lifecycle 跟 OS fiber 很像:
- 每个 plugin 在自己的 fiber 上跑
- 每个 fiber 有自己的状态
- 调度是合作式的(framework 决定何时 unload)

### 4. Service

**定义**:注册到 ctx 上的命名能力,持有者提供具体实现。

**设计初衷**:把"这个类实例代表一个能力"和"它对 ctx 可见"绑成一个动作。
构造时自动 provide,消除"忘了 register 就用不上"的失误。

*Service = "服务"*——一个 ctx 上的命名能力。
ctx.llm      // ← LLM 服务(能调模型)
ctx.fs       // ← 文件系统服务(能读文件)
ctx.shell    // ← shell 服务(能跑命令)
ctx.agents   // ← agent 注册表服务(能创建 agent)
你读 ctx 上的属性,拿到的不是普通数据,而是能做事的对象——这就是 service 的本质:能力。

Service 的整个故事围绕两个核心动作展开:Provide(提供) vs Inject(需要)
1. Provide = "我来提供这个服务"
ctx.provide('llm', myLlmImpl)
// 现在 ctx.llm === myLlmImpl
或者用 Service 基类:
class LlmRuntime extends Service {
  constructor(ctx: Context) {
    super(ctx, 'llm')   // ← 这一行等于 ctx.provide('llm', this)
  }
}
super(ctx, 'llm') 内部就是调 ctx.reflect.provide('llm', this)。构造时自动注册,不会忘。

2. Inject = "我需要这个服务"
class MyPlugin {
  static inject = ['llm']   // ← 声明依赖
  apply(ctx) {
    ctx.llm.stream(...)     // 用服务
  }
}
声明 inject = ['llm'] 后,Cordis 会等 llm 服务注册了才让这个 plugin 启动。

完整流程图
Alice 的 LLM plugin 加载
  ↓
构造 LlmRuntime 实例
  ↓
super(ctx, 'llm')
  ↓
ctx.reflect.provide('llm', this)   ← 注册到 ctx 的 service store
  ↓
现在 ctx.llm === LlmRuntime 实例

Bob 的工具 plugin 加载
  ↓
Bob 的 inject = ['llm']
  ↓
Bob 等 'llm' 出现(因为 inject 声明了)
  ↓
'llm' 已注册 → Bob 启动
  ↓
Bob.apply(ctx) 里调 ctx.llm.stream(...)

**plugin和service是什么关系?**
Plugin 是"加载动作"(跑一段代码);Service 是"命名对象"(出现在 ctx 上,可被读)。两者正交,可以互相嵌套。
|维度|Plugin|Service|
|---|---|---|
|是什么|一段代码(function/class) |一个对象实例|
|何时产生|ctx.plugin(p, config) 调用时|super(ctx, 'name') 或 ctx.provide('name', obj) 时|
|出现位置|不直接出现——只产生副作用|直接以 name 为 key 出现在 ctx 上|
|主要目的|注册副作用 / 提供服务|作为"可读的能力对象"|
|状态|通常 stateless(每次跑都是新执行)|通常 hold 状态(配置、adapter 实例等)|
Plugin 是动词,Service 是名词。

它们可以互相嵌套——三种模式
1. 模式 1:Plugin 提供 Service(最常见)
Plugin body 里调 ctx.provide('name', obj),把一个对象注册成 service:
class LlmPlugin {
  static inject = ['config']
  apply(ctx, config) {
    const llm = new SomeLlmImpl(config)
    ctx.provide('llm', llm)   // ← Plugin 提供一个 Service
  }
}

ctx.plugin(LlmPlugin, { apiKey: 'xxx' })
// 结果: ctx.llm === SomeLlmImpl 实例,由 LlmPlugin 的 fiber 拥有
生命周期:Service 由 Plugin 的 fiber 拥有。Plugin unload → Service 自动消失。
2. 模式 2:Service 类直接作为 Plugin 加载(更紧凑)
Service 类本身就是 new (ctx, config) => any 的形状,符合 Plugin 接口——可以直接被加载为 Plugin:
class LlmRuntime extends Service {
  static inject = ['config']
  static Config = Schema.object({ maxTokens: Schema.number() })
  
  constructor(ctx, config) {
    super(ctx, 'llm')   // ← 这一行 = 自动 ctx.provide('llm', this)
    // 初始化可以用 config (已经合并过 intercept)
  }
}

ctx.plugin(LlmRuntime, { maxTokens: 1000 })
// 结果: ctx.llm === LlmRuntime 实例,同时获得完整 lifecycle
生命周期:Service 实例由 fiber 拥有。Plugin unload → fiber dispose → Service 自动从 ctx 消失。
3. 模式 3:直接 provide,不走 Plugin
const llm = new SomeLlmImpl()
ctx.provide('llm', llm)   // ← 没有 fiber,没有 lifecycle
生命周期:Service 跟普通对象一样,ctx 在就在,不在就不在。没有 plugin unload 自动清理这个福利。

### 5. Event(事件)

**定义**:带类型签名的命名协调通道,有 5 种分发模态。

**设计初衷**:plugin 之间的协调至少是五种关系——通知、集合、抢答、投票、中间件链。
如果只有 `emit`,所有模式都被迫塞进一个 fire-and-forget 接口再手动加状态。
Cordis 给五种语义独立命名,强迫调用者选对。

Event 的三件套 + 五种分发模态

一、三件套
1. ctx.on(name, listener) — 注册 listener
ctx.on('user-login', (userId) => {
  console.log(`${userId} logged in`)
})
// 返回一个 disposer
作用:把 listener 加入事件的监听列表。返回 disposer(不是 boolean,不是 Promise)。见下方dispose()

2. ctx.emit(...) / parallel(...) / ... — 派发事件
ctx.emit('user-login', 'alice-123')
作用:触发事件,让所有 listener 被调用。5 种模态选哪种,决定 listener 怎么被调用。

3. dispose() — 取消监听
const dispose = ctx.on('user-login', handler)
dispose()   // ← listener 从列表移除,之后不会再被调用
作用:撤销注册。Listener 立刻从监听列表消失。Fiber unload 时,所有 listener 的 disposer 会自动跑——你也可以手动调。

二、五种分发模态
5 种模态的差异主要在三个维度:
- 是否 await listener —— 是否等所有 listener 跑完才返回
- 是否返回值 —— 调用方能不能拿到 listener 的返回值
- 是否短路 —— 首个满足条件的 listener 之后是否还跑

|模态|await?|返回 listener 值?|短路|
|---|---|---|---|
|emit|否|否|否|
|parallel|是|否|否|
|serial|是|是|是|
|bail|否|是|是|
|waterfall|可选|是|是(不调 next())

下面逐个看。
1. 模态 1:emit — fire-and-forget
ctx.emit('user-login', userId)
机制:
- 立刻调每个 listener
- 不 await listener 的返回值
- 不收集返回值
- 不抛异常(即使 listener throw,emit 也不抛——错被 logger 吞掉)
listener 视角:
ctx.on('user-login', (userId) => {
  console.log(userId)   // 跑不跑完不影响 emit 调用方
})
语义:这是"广播通知",不是"问问题"。调用方发完就走,不在乎 listener 干啥。

2. 模态 2:parallel — 并行 fan-out + 聚合
try {
  await ctx.parallel('before-save', document)
  // 所有 listener 并行跑,等所有人完成
} catch (e) {
  // 任何一个 listener throw → 整个 parallel throw AggregateError
}
机制:
- 所有 listener 并行启动(Promise.all)
- await 所有人完成
- 任一 reject → 抛 AggregateError
listener 视角:
ctx.on('before-save', async (doc) => {
  await cache.invalidate(doc.id)   // 异步操作 OK
  await log.record('save', doc)
})
语义:多个订阅者独立做事,且所有人得跑完才能继续。任一失败 → 整个失败。

3. 模态 3:serial — 有序 racing
const handler = await ctx.serial('resolve-handler', request)
// listener 按注册顺序跑
// 第一个返回非 null/false/undefined 的胜出
// 后面不再跑
// handler = 胜出 listener 的返回值
机制:
- listener 按注册顺序,串行 await(不是并行)
- 每跑完一个,检查返回值
- 第一个"有意义"的返回(非 null/false/undefined)胜出,链停
- 如果所有 listener 都返回"无意见"(null/undefined/false),serial 整体也返回 undefined
listener 视角:
// 注册顺序:plugin A 先,plugin B 后
ctx.on('resolve-handler', (req) => {
  if (req.type === 'special') return specialHandler
  // 不 return = undefined = "我处理不了,问下一个"
})

ctx.on('resolve-handler', (req) => {
  return defaultHandler  // 默认 handler,啥都接
})
// 普通请求由第二个 listener 接收;特殊请求由第一个
语义:"第一个能处理的负责",有顺序(先注册的先尝试)。

4. 模态 4:bail — 同步 voting
const veto = ctx.bail('permission-check', action)
// 同步跑 listener,首个返回非空对象胜出
// 没 await
机制:
- 同步跑 listener(不是 async,不用 await)
- 第一个返回非 null/false/undefined 的胜出
- 同步返回结果
- 没异步支持——listener 不能是 async function
listener 视角:
ctx.on('permission-check', (action) => {
  if (action.type === 'dangerous') {
    return { allowed: false }   // 否决
  }
  // 不 return = "没意见"
})
语义:bail 是 serial 的同步版本——同样 racing,只是不 await。

5. 模态 5:waterfall — middleware chain
const result = await ctx.waterfall('process', input,
  (input, next) => {
    const step1 = doStep1(input)
    return next(step1)   // ← 关键:调 next 才继续
  }
)
机制:
- listener 是 (args, next) => result
- next 是 args 最后一个参数,调用 next 才进入下一个 listener
- 不调 next = 否决整条链(包括最终的 builtin behavior)
- 链结束时,所有 listener 嵌套包裹,每个监听者有机会"包住"下一个
listener 视角:
ctx.on('process', async (input, next) => {
  console.log('before')
  const output = await next(input)   // ← 进下一个 listener
  console.log('after')
  return output
})
// 多个 listener 嵌套包裹,顺序 = 注册顺序的逆序
语义:中间件 / 转换管道。每个 listener 包下一个,可以做 pre/post hook。可以否决(不调 next)。

### 6. Epoch(epoch 字符串)

**定义**:fiber 当前依赖集合的指纹字符串。

**设计初衷**:plugin 的依赖图是动态的——服务被卸载 / 重载 / 替换。手动级联通知脆弱。Epoch 用一个字符串给依赖集合"指纹":变了说明依赖变了,触发 reload。

epoch 是一个字符串,用 ':' + impl.fiber.uid 拼接每个 inject 依赖:
_refresh() {
  let epoch = ''
  for (const name of Object.keys(this.inject)) {
    const impl = this._store[name]
    if (!impl) { epoch = INACTIVE; break }  // 缺失 → INACTIVE
    epoch += ':' + impl.fiber.uid             // 拼接
  }
  this._setEpoch(epoch)
}
字符串相等性 = 依赖状态完全相同。
- 同样的服务、同样的实现、同样的顺序 → 同样字符串 → 不动
- 任一变化 → 字符串变 → 触发动作

**什么变化才会影响epoch字符串？**
Epoch 只对两个东西敏感:inject 集合本身、和提供每个 inject 的 fiber 的 uid。 其他一切变化都不影响 epoch。
_refresh() {
  let epoch = ''
  for (const name of Object.keys(this.inject)) {   // ← inject 集合
    const impl = this._store[name]
    if (!impl) { epoch = INACTIVE; break }
    epoch += ':' + impl.fiber.uid                   // ← 提供者的 fiber.uid
  }
  this._setEpoch(epoch)
}
两个变量 = inject 集合 + 提供者的 fiber.uid。 其他都不参与。

### 7. Effect / Disposable

**定义**:Effect 是 Cordis 对"可撤销副作用"的统称;Disposable 是 effect 返回的撤销函数;Fiber 的 _disposables 列表把这两件事组装成完整的生命周期。
每个注册 API(`plugin` / `on` / `provide` / `effect` / `accessor` / `mixin`)都是 effect,都返回 disposer。

**设计初衷**:传统 framework 要求作者维护 listener 列表、写清理钩子、处理级联卸载,漏一处就泄漏。Cordis 统一承诺:**只管注册,不管清理**——
作者写 `() => { /* 副作用 */ }`,framework 决定何时撤销。

**Framework 怎么决定何时撤销，规则是什么?**
没有魔法,只有字符串比较。Framework 通过 5 种触发条件决定撤销,核心算法是"对比 epoch 字符串":变了就卸载/重载,没变就什么都不做。
1. 显式手动撤销
// 三种手动入口
ctx.registry.delete(plugin)    // 1. 从 registry 删 plugin → 所有 fiber 撤销
fiber.dispose()                // 2. 直接调某个 fiber 的 dispose
dispose()                      // 3. ctx.on / ctx.effect 返回的 dispose 被调
任何时候代码显式调 dispose,framework 立即执行撤销。

2. Epoch 变 INACTIVE(依赖缺失)
// 场景:某 plugin 提供 'fs',然后被 dispose
ctx.registry.delete(fsPlugin)
// → store 里 'fs' 的 Impl 被删除
// → notify(['fs']) 触发所有 inject 'fs' 的 fiber
// → 这些 fiber 的 _refresh() 发现 inject 里 'fs' 没 impl 了
// → epoch = '__INACTIVE__'
// → _setEpoch(INACTIVE) 触发 _unload
规则:任何一个 inject 的服务消失,inject 它 fiber 必须 unload。

3. Epoch 字符串变化(依赖换了)
// 场景:替换 'fs' 的实现
ctx.registry.delete(oldFsPlugin)
ctx.provide('fs', newFsImpl)   // 不同的 fiber.uid

// inject 'fs' 的 fiber:
// epoch 旧:':5'(oldFs 的 uid)
// epoch 新:':7'(newFs 的 uid)
// 不同 → _setEpoch(newEpoch) 触发 unload → reload
规则:依赖变了 = 重新评估 fiber 的 body(load)。

4. HMR / config 更新
// 场景:用户改了 cordis.yml,触发文件 watcher
fiber.update(newConfig)
// → 走 internal/update waterfall
// → 重新 resolve config
// → restart fiber(unload → reload)
规则:config 改了 = 重新加载 fiber。

5. 根 context dispose
// 应用关闭时,根 context dispose
ctx.dispose()
// → 所有 fiber 反序撤销

决策算法(核心)
framework 决定"要不要撤销/重载"时,走这个流程:
触发条件发生
  ↓
_fiber._refresh()                // 重算 epoch
  ↓
新 epoch == 旧 epoch?
  ├─ 是 → 什么都不做(stable)
  └─ 否 → 进入下一步
  ↓
新 epoch == INACTIVE?
  ├─ 是 → _unload()(依赖缺失,只能撤销)
  └─ 否 → _setEpoch(newEpoch)
            ↓
          _unload() + _reload()(依赖变了,卸载旧 + 加载新)

### 8. Isolation / Intercept / Mixin(三个作用域操作符)

**定义**:
- `isolate(name, label?)` — 给同名 service 创建独立 Symbol key
- `intercept(name, config)` — 在配置合并链上插入一项
- `mixin(source, mixins)` — 把 service 方法以 accessor 形式暴露到 ctx 上

**设计初衷**:同名多实例、配置级联、服务方法伪装 ctx 方法——三者都是
"在原型链上做不同映射"。Cordis 用同一套底层机制覆盖三种需要。

一、Isolate(同名多实例)
问题: 同一个 service name,需要不同 scope 下指向不同实例。比如:
- 测试 vs 生产的同一个服务名指向不同实现
- 多个 agent 都需要 "LLM" 但用不同 model
- 不同 preset 需要独立的子 agent 注册

例子:公司里两个部门都有"经理"。A 部门的"经理"是 Alice,B 部门的"经理"是 Bob。你在公司目录下说"经理是谁",答案是 Alice;你跳进 B 部门的目录说"经理是谁",答案是 Bob。

解法: 用 Symbol 隔离同一个 name 在不同 scope 下的 store key。
const root = new Context()
root.provide('meeting-room', ceoRoom)    // 用 root 的 isolate Symbol
const child = root.isolate('meeting-room')   // 创建新 scope,'meeting-room' 改用新 Symbol
child.provide('meeting-room', deptRoom)  // child 自己 scope 的 Symbol

root['meeting-room']   // ceoRoom  ← root 的 Symbol
child['meeting-room']  // deptRoom ← child 的 Symbol(isolate 后改的)

相当于在每个"区域"里给这个名字发一个不同的身份证。所以虽然两个部门都叫"经理",但物业内部记录的是"经理 A 区-身份证 123"和"经理 B 区-身份证 456"。你问的时候物业看你站在哪个区,翻对应的身份证,告诉你对应的人。

机制
// context.ts:121-125
isolate(name: string, label?: symbol) {
  const shadow = Object.create(this[symbols.isolate])
  shadow[name] = label ?? Symbol(name)   // ← 新 Symbol
  return this.extend({ [symbols.isolate]: shadow })
}
// reflect.ts:286-287
this.ctx.root[symbols.isolate][name] ??= Symbol(name)  // root 的固定 Symbol
const key = this.ctx[symbols.isolate][name]            // 当前 scope 的 Symbol
this.store[key] = impl                                  // store 用 Symbol 作 key
JS 原型链查找自动按当前 scope 的 Symbol 解析,不需要任何特殊代码。

实操场景
// 创建两个独立的 LLM scope
const root = new Context()
root.provide('llm', prodLlm)

const testing = root.isolate('llm')
testing.provide('llm', mockLlm)

// 测试场景用 testing 的 ctx
testing.llm   // mockLlm
root.llm      // prodLlm(不受影响)

何时用
- 测试场景需要 mock 某个 service
- 多 agent / 多 preset 需要独立 LLM 配置
- subagent 有自己的 ctx 树

二、Intercept(配置级联)
问题: 不同层级想覆盖某个 service 的 config,但不改源码:
- 根 config 提供默认值
- 用户层覆盖某些字段
- 临时 session 再覆盖

例子:连锁餐厅。总部定了菜单默认值(所有店都有);北京店想加北京烤鸭;上海店想减一道北方菜、加小笼包。每家店不需要重新写菜单,只需要说"我要加什么、减什么"。默认值由源头提供,每家店只关心自己改的部分。

解法:沿原型链走 intercept map,合并所有 ancestor 的 config(root → leaf 方向)。
class LlmRuntime extends Service {
  static Config = Schema.object({
    maxTokens: Schema.number().default(1000),
    temperature: Schema.number().default(0.7)
  })
  constructor(ctx, config) {
    super(ctx, 'llm')
    // config 已经合并过所有 intercept 层
    this.maxTokens = config.maxTokens   // 1000 或覆盖值
    this.temperature = config.temperature
  }
}

打开一个层(比如子 ctx),你说:"菜单里 maxTokens=500"。框架把所有层说过的话按"总部 → 分部 → 店"的顺序合并起来,合并后的菜单才是这家店实际用的。

// 默认
ctx.provide('llm', new LlmRuntime(ctx, defaultConfig))

// 某个子 scope 覆盖
ctx.intercept('llm', { maxTokens: 500 })
// LlmRuntime 构造时拿到 { maxTokens: 500, temperature: 0.7 }
// (没被覆盖的字段用 default)

机制
// service.ts:86-102
[symbols.resolveConfig](base?: T, head?: T): T {
  let intercept = this.ctx[Context.intercept]
  const configs: any[] = []
  while (this.name in intercept) {
    if (Object.hasOwn(intercept, this.name)) configs.unshift(intercept[this.name])  // ← unshift:根在前
    intercept = Object.getPrototypeOf(intercept)   // ← 沿原型链向上走
  }
  if (base) configs.unshift(base)
  if (head) configs.push(head)
  if (this['Config']?.merge) return this['Config'].merge(...configs)
  return Object.assign({}, ...configs)
}
unshift 让根的 intercept 先合并,然后是更深的层级,最后是 plugin 自己的 base/head。
合并方向
[ 根 intercept: { maxTokens: 1000 } ]     ← 最先
+ [ 子 intercept: { maxTokens: 500 } ]    ← 然后覆盖
+ [ base config ]                         ← 然后
+ [ head config ]                         ← 最后
= 合并结果

何时用
- 用户设置覆盖默认配置
- Profile-level 配置覆盖 base
- A/B testing 临时改某个参数

三、Mixin(service 方法 → ctx 属性)
问题: 每次都写 ctx.events.emit(...) 很啰嗦。想让 ctx.emit(...) 直接可用——但 emit 不是 ctx 的 own property,是 events service 的方法。

例子:你手机里有"相机"App。你想拍照,需要从桌面找到相机图标、点开、点拍照按钮。三步。但你的锁屏上有一个"相机"快捷按钮。一点就拍照——你不用先打开相机 App 再点拍照。

解法: 在 ctx 上创建 accessor,把对 ctx.foo 的访问转发到 ctx.<service>.foo。Mixin 不是"创造新方法",是"把 service 的方法复印一份快捷方式放在 ctx 上"。

// 没有 mixin:
ctx.events.emit('foo', data)   // 必须走 service

// 有 mixin(emit 已 mix 到 ctx):
ctx.emit('foo', data)          // 看起来是 ctx 方法,实际转发到 ctx.events.emit
机制
ReflectService 构造时预注册 4 个 mixin:
// reflect.ts:219-222
this.mixin('reflect', ['get', 'set', 'provide', 'accessor', 'mixin'])
this.mixin('fiber', ['runtime', 'effect'])
this.mixin('registry', ['inject', 'plugin'])
this.mixin('events', ['on', 'once', 'parallel', 'emit', 'serial', 'bail', 'waterfall'])
所以 ctx.emit ctx.on ctx.plugin ctx.inject ctx.provide 都是 accessor,转发到对应的 service。
mixin(source, mixins) 是生成器 effect:
// reflect.ts:364-390
mixin(source, mixins) {
  return this.ctx.fiber.effect(function* () {
    const entries = Array.isArray(mixins) ? mixins.map(k => [k, k]) : Object.entries(mixins)
    for (const [key, value] of entries) {
      yield self.accessor(value, { get, set })   // ← 每个 mixin 创建 accessor
    }
  }, `ctx.mixin(${JSON.stringify(source)})`)
}
Accessor 的 getter
get(receiver, error) {
  const service = getTarget(this, error)        // this['events']
  if (isNullable(service)) return service
  const mixin = receiver ? withProps(receiver, service) : service
  const value = Reflect.get(service, key, mixin)  // service[key],bind 到 ctx
  // ...
}
关键:服务方法被调用时,this 跟调用者 ctx(自动绑方法)。

### 9. Reflect / Proxy(反射层)

**定义**:让 `ctx.foo` 自动解析 service 的 Proxy 机制。

**设计初衷**:它解决的问题——经典 DI 太啰嗦

你见过的 DI 框架,服务调用长这样:
ctx.inject('llm').getService().stream(...)
ctx.getBean('llm').execute(...)
container.resolve('llm').call(...)

Cordis 想做的是:
ctx.llm.stream(...)

少一层包装,看起来像在读对象的属性。怎么做到的?Proxy。ctx 是一个 Proxy,不是普通对象
new Context() 返回的不是普通的 Context 实例,而是包了一层 Proxy 的 Context。
这意味着:你读 ctx 上的任何属性,都会被 Proxy 拦截,不会直接走普通的 JS 属性查找。

Proxy 拦截后做什么?查三层:
第 1 层:ctx 自己有的属性?
  ├─ events、logger、reflect、registry、fiber、root → 这些是内置服务
  │  → 直接返回
  └─ 不是 → 走第 2 层

第 2 层:在 service store 里找?
  ├─ 找到了 → 返回这个 service(再包一层 Proxy,见下文)
  └─ 找不到 → 走第 3 层

第 3 层:你 inject 过这个 service 吗?
  ├─ inject 过但找不到 → 抛 "依赖缺失"
  └─ 没 inject 过 → 抛 "无此 property"
所以 ctx.foo 不是"找属性",是"按规则层层查"。

一句话回答"为什么需要 Proxy"
让"读 ctx 的属性"看起来像普通属性访问,但实际触发 service store 查找 + service 方法的 this 绑定。 用户代码简洁,框架做对的事。

### 10. Symbol(共享符号表)

**定义**:Cordis 用一组固定的 unique symbols 做协议标记。

**设计初衷**:框架内部协议标记(`isolate` / `intercept` / `shadow` 等)不能撞用户起的属性名。用 Symbol 作 key,跨 realm 共享,字符串属性名互不干扰。

Symbol 是 Cordis 在对象上贴的"内部标签",专门用来标记框架私有的元数据——因为 Symbol 不会跟用户的属性名撞车。

这个有点像Python中的关键字，在编码时候是要可以避开的。
Python 关键字和 Cordis Symbol 解决的是同一个问题:框架/语言需要一个"内部命名空间",防止跟用户的命名冲突。
Python 的解法是:语言级别硬性禁止。
Cordis 的解法是:用 Symbol 创建一个独立的、不可见的命名空间。

## 二、模块骨架(7 个 src 文件)

`vendor/cordis/src/index.ts` 桶导出,真正的逻辑在 7 个文件里。

| 模块 | 一句话职责 |
|---|---|
| `context.ts` | Context 类 + 三个作用域操作符 |
| `service.ts` | Service 基类 + intercept 配置合并 |
| `fiber.ts` | 生命周期 + effect + epoch 反应性 |
| `events.ts` | 5 模态事件分发 |
| `registry.ts` | Plugin 形状归一化 + 启动 |
| `reflect.ts` | Proxy handler + service store |
| `utils.ts` | DisposableList + getTraceable + symbols |

模块依赖:

```
       ┌──────────────┐
       │   context.ts │  ← 根
       └──────┬───────┘
              │
       ┌──────▼───────┐
       │   fiber.ts   │  ← 生命周期、effect、epoch
       └──────┬───────┘
              │
   ┌──────────┼──────────┬────────────┐
   ▼          ▼          ▼            ▼
events.ts  registry.ts  reflect.ts  service.ts
   │          │          │
   └──────────┴──────────┴────→ utils.ts (叶子,只依赖 cosmokit)
```

## 三、5 个一等公民

10 个概念中,**5 个承担 Cordis 的核心机制**:

| 一等公民 | 核心作用 |
|---|---|
| Effect / Disposable | plugin 作者只写 register,framework 负责 cleanup |
| 原型链 Context | scope 复用 JS 原生查找语义 |
| Symbol 隔离 | 同名服务多实例 |
| 5 模态事件 | 协调语义独立命名 |
| Epoch 反应性 | 依赖变化自动 reload |

剩下 5 个概念(Service / Plugin / Proxy / State machine / Reflect symbols)是工程化封装。

## 四、把它们串起来看:完整生命周期

> 一家餐厅的故事:主厨 Alice 入职,做菜,被合并,离职。

**阶段 1 — 入职(`PENDING`)**
Alice 来餐厅报到,被分到"热菜 6 号工位"(Fiber)。
工位还空着——她宣布"我需要食材才能开火",物业在工位门口贴上"等食材到"。

```ts
ctx.plugin(chefAlicePlugin, { station: 'hot-6' })
// 物业给她一个工位(Fiber),钥匙环空着但已挂这工位
// 状态:PENDING(等食材)
```

**阶段 2 — 食材到位(`LOADING → ACTIVE`)**
供应商送来 `tomato` 和 `garlic`,在仓库登记。
物业检查 Alice 工位的需求:她需要 `tomato` 和 `garlic`——都到了。
Alice 拿到钥匙,开始备料(跑 plugin body)。

```ts
ctx.provide('tomato', tomatoBatch)
ctx.provide('garlic', garlicBatch)
// 物业查:Alice 的 inject 依赖都到位了
// 状态:PENDING → LOADING → ACTIVE
// 食材清单编号变化:'__INACTIVE__' → ':tomato:garlic'(epoch)
```

**阶段 3 — 营业(emit)**
餐厅开始接单:`ctx.emit('order', { dish: 'stir-fry' })`。
所有员工(listener)同时收到——洗碗的备盘子,前台的准备喊号,Alice 开始炒菜。
每个员工忙完就回去等下一单——他们都是 effect,Alice 离职时跟着走。

```ts
ctx.emit('order', { dish: 'stir-fry' })
// 物业 PA 喊一声
// 所有订阅的 listener 并行触发
// Alice 自己的 listener(厨房工单)也是 effect,她走时自动撤销
```

**阶段 4 — 换供应商(reload)**
食材供应商从"老张"换成"小李"——`tomato` 这条线换了新货。
物业检查:Alice 的清单编号从 `':tomato:garlic'` 变成 `':tomato-new:garlic'`。
编号变了——Alice 重新备料(卸旧 + 装新),但工位不动。

```ts
ctx.registry.delete(oldTomatoSupplier)
ctx.provide('tomato', newTomatoBatch)
// 物业通知所有工位:tomato 变了
// Alice 的食材清单编号变了(epoch 不同)
// Alice:ACTIVE → UNLOADING(撤旧备料)→ ACTIVE(用新食材重新备料)
```

**阶段 5 — 离职(`DISPOSED`)**
餐厅合并,6 号工位撤销。`ctx.registry.delete(chefAlicePlugin)`。
物业执行 Alice 的工位撤离清单(dispose 链):清冰箱、洗厨具、归还钥匙、撕工位标签。
工位拆除,Alice 的 uid 置 null(再来这个工位就报错"工位已撤销")。

```ts
ctx.registry.delete(chefAlicePlugin)
// 物业执行 Alice 的所有清理:
//   1. 移除她的 listener
//   2. 撤销她的 timer(炖汤忘了关火这种事)
//   3. 撤销她注册的 service
// 状态:ACTIVE → UNLOADING → DISPOSED
```

## 五、Cordis 之上:DeepSeek Harness 加了什么

```
Cordis 提供           DeepSeek Harness 加什么
─────────────────────────────────────────────────────
DI                →   18 个 capability seam
                     (LLM / Shell / FS / Subagent / ...)
event bus         →   3 个 SessionEvent 类型域
                     (session/event 模型真相源、agent/* 实时协调、capability/* 策略钩子)
fiber 生命周期    →   turn / step 循环(inbox / claim / reject / drive)
prototype scope   →   Layered Scope(可见性向下,事件向上)
disposable        →   append-only Session 日志(turn 结束不能撤,只能补偿)
```

# 第二部分 — 设计哲学与七个核心命题

## 第一性原理
> **应用是一棵在运行时由配置组合的插件树，每个插件向共享上下文贡献可你效应**

这是 Cordis 不可再约简的公理。从这一条公理出发，所有其他设计都是逻辑推论：

```
公理: 应用 = 运行时组合的插件树 + 可你效应
 |- 插件需要互相找到 -> 服务注册表 + 上下文代理
 |- 插件加载顺序不确定 -> 依赖声明 (inject)
 |- 插件可以卸载 -> 效应必须可逆 (effect + disposer)
 |- 插件需要通信 -> 解耦的事件总线
 |- 不同部分需要不同服务实例 -> 作用于隔离 (isolate/intercept/extend)
 |- 应用由配置组合 -> 声明式加载器 (Loader/Include)
 |_ 配置可在运行时变更 -> HMR + 事务性更新
```

## 1. 注册即效果(Registration is Effect)

`Context` 上每个动作都返回 disposer:`ctx.plugin`、`ctx.on`、`ctx.provide`、`ctx.effect`、`ctx.accessor`、`ctx.mixin`。plugin 作者从不"清理"——他们只"注册",由 framework 决定何时撤销。

这由 `Fiber.assertActive()`(`vendor/cordis/src/fiber.ts:351-354`)强制,它在 fiber 已 dispose 的情况下抛 `INACTIVE_EFFECT`。

## 2. Effect 组成图,不是栈

`_disposables` 是 `DisposableList`(`vendor/cordis/src/utils.ts:5-40`),不是 `Set` 或 `Map`。它支持 O(1) 按值删除、按插入顺序迭代、反向 disposal。注册时 disposer **立即**加入父 fiber 的 disposables,在 plugin body 跑之前——这样 plugin 加载失败时,framework 仍可撤销已注册的内容,不泄漏。

生成器 effect 类型(`Iterable<Disposable, void, void>`,`vendor/cordis/src/fiber.ts:83-93`)让你增量 yield disposer。每个 yield 出的值立即被收集,所以中途崩溃的生成器也能被清理。

## 3. 原型链即作用域

Cordis **不引入自己的 DI token 词汇**,也 **不通过闭包传递上下文**。`Context.extend(meta)`(`vendor/cordis/src/context.ts:99-107`)字面上就是 `Object.create(parent)` 加属性复制:

```ts
extend(meta = {}): this {
  const shadow = Reflect.getOwnPropertyDescriptor(this, symbols.shadow)?.value
  const self = Object.create(getTraceable(this, this))
  for (const prop of Reflect.ownKeys(meta)) {
    Object.defineProperty(self, prop, Reflect.getOwnPropertyDescriptor(meta, prop)!)
  }
  if (!shadow) return self
  return Object.assign(Object.create(self), { [symbols.shadow]: shadow })
}
```

子 context 继承父的每个属性。`meta` 的 own property shadow 父的。**scope、intercept、isolate 都只是同一个原型链上的不同映射**。

`getTraceable`(`vendor/cordis/src/utils.ts:117-125`)只做一件事:把服务裹一层 Proxy,使其方法调用落到正确的 fiber context。

## 4. 隔离即 Symbol

`Context.isolate(name, label?)`(`vendor/cordis/src/context.ts:121-125`):

```ts
isolate(name: string, label?: symbol) {
  const shadow = Object.create(this[symbols.isolate])
  shadow[name] = label ?? Symbol(name)
  return this.extend({ [symbols.isolate]: shadow })
}
```

赋给 `name` 的 Symbol 成为 service store 的 key。两个对 `name` 有不同 Symbol 的 context 看到不同的 store 条目——自然、无污染的命名空间隔离。

`ReflectService.provide`(`vendor/cordis/src/reflect.ts:286-287`)是匹配的生产侧:

```ts
this.ctx.root[symbols.isolate][name] ??= Symbol(name)  // root 拿到固定 key
const key = this.ctx[symbols.isolate][name]            // 子 isolate 拿到自己的 key
this.store[key] = impl
```

root 的 Symbol 在所有未 isolate 的子 context 之间共享。子 context 调 `isolate()` 分配新 Symbol;store 现在对同一 `name` 用不同 key。JS 原型查找负责其余部分。

## 5. 拦截即配置合并

`Service[symbols.resolveConfig]`(`vendor/cordis/src/service.ts:86-102`)从叶子到根走 intercept 映射,收集条目,然后合并——`base` 最先,`head` 最后:

```ts
[symbols.resolveConfig](base?: T, head?: T): T {
  let intercept = this.ctx[Context.intercept]
  const configs: any[] = []
  while (this.name in intercept) {
    if (Object.hasOwn(intercept, this.name)) configs.unshift(intercept[this.name])
    intercept = Object.getPrototypeOf(intercept)
  }
  if (base) configs.unshift(base)
  if (head) configs.push(head)
  if (this['Config']?.merge) return this['Config'].merge(...configs)
  return Object.assign({}, ...configs)
}
```

`unshift` 产生根优先顺序。如果 service 声明了 Schemastery `Config`,framework 用 `Config.merge` 做语义合并;否则浅 `Object.assign`。plugin 从不读"全局配置"——它读"我的 inject + 我的 intercept 层",级联自动发生。

## 6. 事件有五种正交模态

`DispatchMode = 'emit' | 'parallel' | 'serial' | 'bail' | 'waterfall'`(`vendor/cordis/src/events.ts:32`)。五种模态编码五种不同的协调模式:

| 模态 | 异步? | 返回值? | 短路? | 用途 |
|---|---|---|---|---|
| `emit` | 否(在 listener 沉淀前返回) | 否 | 否 | fire-and-forget 通知 |
| `parallel` | 是 | 否 | 否 | fan-out 然后聚合 |
| `serial` | 是 | 是 | 是(首个 bail 胜出) | 有序决策链 |
| `bail` | 否 | 是 | 是(首个 bail 胜出) | 快速同步投票 |
| `waterfall` | 可选 | 是 | 是(不调 `next()` 即否决) | 中间件链 |

五种模态不是同一模式的"快/慢"变体。它们是 plugin 之间的**不同社交协议**。Cordis 强迫调用者选对那个——而不是把"我想等到结果"塞进一个 fire-and-forget 接口再用 closure 状态手动实现。

bail 判定(`isBailed`,`vendor/cordis/src/events.ts:13-15`)是 `value !== null && value !== false && value !== undefined` ——按约定,**沉默即同意**。

## 7. 错误栈合成而非传播

`composeError`(`vendor/cordis/src/utils.ts:268-281`)和 `handleError`(`:240-265`)在抛错时把外层栈(注册时捕获)拼接到内层异步栈中。effect 链让原生 V8 栈对调试无用;这个构造函数恢复了 fiber 边界的可见性。

```ts
function handleError(info, reason, getOuterStack): never {
  const lines = reason.stack.split('\n')
  let index = lines.indexOf(innerLines[2])
  index -= info.offset
  while (index > 0) {
    if (!lines[index - 1].endsWith(' (<anonymous>)')) break
    index -= 1
  }
  lines.splice(index, Infinity, ...getOuterStack())
  reason.stack = lines.join('\n')
  throw reason
}
```

`buildOuterStack`(`vendor/cordis/src/utils.ts:284-286`)惰性捕获注册点栈——所以外层上下文是 effect 注册的地方,而不是 effect 运行的地方。

---

# 第三部分 — Deepseek Harness 在 Cordis 基础上怎么构建的 Agent
## 核心问题
Cordis解决的是“如何把插件拼在一起”，DSH需要回答：
1. 如何驱动一个Agent的对话循环?(Agent Loop)
用户输入 → Inbox 排队 → driver 唤醒 → turn 循环(每个 step:抢消息 / 拼 prompt / 调 LLM / 跑 tool)→ 结果落 session log → 回屏幕。
6 个核心节点
用户键入 ──① Inbox──② wakeDriver──③ turn 循环──④ step(LLM call)──⑤ tool 执行──⑥ 屏幕
          (持久化)    (idle→running)  (turn/step event)  (chunk/message)  (call/result)   ↑
                                                          │                              │
                                                          └──── Session log ◀────────────┘
一句话本质
整条链路是单向数据流——LLM 只能通过 tool call 反过来影响 session log,没有"模型直接改写 log"的捷径。这就是"模型可见 = 日志可重建"的不变式。

6 个环节对应 package
|环节|Package|角色|
|---|---|---|
|① Inbox 排队|packages/core/agent/|AgentRegistry + Inbox + initiator AsyncLocalStorage|
|② wakeDriver|packages/core/agent-loop/|ReactLoopAgent 状态机(idle/running 转换)|
|③ turn 循环|packages/core/agent-loop/|turn()/step() 主体,turn/start/step/end event|
|④ step + LLM call|core/agent-loop + core/llm/|buildRequest + llm.stream + BlockAssembler|
|⑤ tool 执行|packages/core/tools/	5 阶段 pipeline|(pre-execute/execute/post-execute/result)|
|⑥ 屏幕输出|Web UI / ACP / CLI|UI 层(不是 agent-loop 本身)|

2. 如何让模型看到正确的上下文?(Session log + system prompt)
*prompt 是"拼出来的字符串",messages 是"从 session log 投影出来的"*——两件事各管一段。
4 个 prompt 来源
LLM 看到的内容
├─── system 字段(prompt 拼出来的)
│     ├─ sections(按 order 排序的文本段)
│     ├─ contexts(运行时快照,如 cwd)
│     ├─ tools(模型可见的工具 schema)
│     └─ variables(`{{name}}` 替换)
└─── messages 数组(session log 投影)
      ├─ user/message(直接 surface)
      ├─ assistant/message(content blocks)
      └─ tool/result(跟 tool/call 配对)
4 个 Cordis 原语各管一段
|概念|用途|
|---|---|
|SurfaceManager|维护"模型可见的 event seq 列表"——决定哪些 event 进 messages|
|ScopedLayers|同名 section 在 agent scope shadow 全局——决定 prompt 内容|
|system-prompt/assemble waterfall|允许 plugin 改 assembly——决定 prompt 最终形态|
|request/header 增量 fold|每次 buildRequest 比对前一次,只在变化时写 log|

3. 如何让Agent调用工具?(tool registry + executrion pipeline)
两件事
如何让Agent调用工具 = 
  ① Tool Registry(怎么注册 + 怎么让 LLM 看到工具)
  ② Execution Pipeline(LLM 决定调用后,实际怎么跑)
① Tool Registry(注册 + 可见性)
LLM 怎么"看到"工具
packages/core/tools/src/index.ts 提供 4 个 API 让 plugin 注册工具:
// 工具作者(plugin 端)调用:
ctx.tools.register({
  name: 'read_file',
  description: '...',
  parameters: Schema.object({ path: Schema.string() }),
  // optional:
  execute: async (args, exec) => { /* tool 实现 */ },
  isConcurrencySafe: (args) => true,  // 是否可以并行
})

// 或更紧凑的 defineTool helper:
const readFile = defineTool({
  name: 'read_file',
  parameters: ...,
  execute: ...,
})
ctx.tools.register(readFile)
Tool registry 干 3 件事
职责	用途
存储工具	name → ToolDefinition map
暴露 schema	ctx.systemPrompt 通过 wireSchemas(scope) 收集所有可见 tool 的 schema
scope 可见性	ctx.tools.view(agent) 返回 agent scope 可见的工具(同名工具在子 scope 可以 shadow)
关键点:tool schema 进 prompt
每个工具的 schema(OpenAI function calling 格式)被收集进 prompt assembly:
[tool/read_file]
- name: read_file
- description: read a file from disk
- parameters: { path: string }

[tool/bash]
- name: bash
- description: run a shell command
- parameters: { command: string, timeout?: number }
LLM 看到这些 schema,自己决定要调哪个、参数是啥。LLM 输出:
{ type: 'tool-call', id: 'abc', name: 'read_file', arguments: { path: '/etc/hosts' } }
② Execution Pipeline(5 阶段)
直观流程
LLM 输出"我要调 read_file"之后,framework 经过 5 道关才真正执行:
LLM 说: 调 read_file('/etc/hosts')
         │
         ▼
阶段 1: prepare  (tools/pre-execute waterfall)
         ├─ 检查参数合法性
         ├─ 检查权限(谁允许调)
         ├─ 检查沙箱(是否被允许访问这个路径)
         ├─ 检查并发性(可不可并行)
         └─ 询问用户(危险操作要确认)
         │
         ▼ allow → 阶段 2
         ▼ deny → 直接返回错误结果
         ▼ ask  → 等用户回复

阶段 2: dispatch  (tools/execute waterfall)
         └─ 实际调 tool.execute(args, exec)
         │   ← 可以被 plugin 拦截(改写参数、mock 结果)

阶段 3: finalize  (tools/post-execute waterfall)
         ├─ 检查结果(是否成功、是否需要重新格式化)
         ├─ 决定:accept / block / replace
         └─ block 时,可以给模型反馈"这个工具不能用,请试别的"

阶段 4: finish    (emit tools/result)
         ├─ freeze 结果(防止后修改)
         └─ 通知所有 listener(审计、日志)

阶段 5: surface   (session.append 'tool/result')
         └─ 落到 session log,作为下一次 LLM 调用的历史
并行调度
LLM 可能一次输出多个 tool calls,framework 怎么调度?
LLM 输出:
  [call1: read_file('a.txt'), call2: read_file('b.txt'), call3: bash('rm -rf /')]
        │
        ▼
executeToolCalls(ctx, turn, step, toolCalls, signal):
  for each tool:
    mode = ctx.tools.executionMode(call).kind
    // mode = 'parallel' 或 'exclusive'(可并行 / 必须独占)
    if mode === 'parallel':
      加入并行池(bounded: maxParallelToolCalls)
    else:
      等当前池跑完,独占跑这个

  并行池:
    pool.size < maxParallelToolCalls && 还有任务:
      启动新 tool(走 5 阶段 pipeline)
      pool.size++
    
    pool 不空:
      等任何一个跑完
      提交它的结果(按 model order 排序,保持 LLM 输出顺序)
      pool.size--
关键:虽然 tool 跑是并行的,但结果提交顺序跟 model 输出顺序一致(用 commitReady() model-order commit)。

4. 如何持久化对话? (session persitence)
整体架构
进程运行时(内存):
  Session
    events: SessionEvent[]  ← append-only,事件溯源
    surface: SurfaceManager ← 模型可见消息投影
    + 各种 fold(epoch / request header / context)

         ↓ 何时写?
         │
         ▼
Persistence Backend(磁盘):
  SessionLocation → SessionRawArtifact
  { sessionId, events: SessionEvent[], header: SessionHeader }
  
  两种 backend:
  - JSONL append-only 文件(简单)
  - SQLite(SCHEMA_VERSION 单调,支持 schema 迁移)
三个 package 的分工
|Package|角色|
|---|---|
|packages/core/session/|内存中的 Session + Surface + deriveMessages|
|packages/core/session-persistence/|Service Definition:SessionPersistence 接口|
|packages/core/session-persistence-jsonl/|JSONL 实现|
|packages/core/session-persistence-sqlite/|SQLite 实现|
何时写磁盘?
ctx.sessionPersistence.flush() 在以下时刻触发:
|触发|原因|
|---|---|
|idle timeout|session 闲置 N 秒后,自动 flush(防止进程崩溃丢数据)|
|显式 flush|用户主动 ctx.sessions.flush(sessionId)|
|process exit|SIGTERM / SIGINT 信号处理时,flush 所有活跃 session|
|compaction 后|compaction 改了 log,需要重新持久化|
|turn-end|某些配置下,每个 turn 结束就 flush|
启动时怎么恢复?
SessionStore.restore(sessionId)(packages/core/session/src/index.ts:792-...):
1. ctx.sessionPersistence.read(sessionId)
   ↓
   从 JSONL / SQLite 读出 SessionRawArtifact
   ↓
2. 校验:
   - sessionFormatVersion 匹配(SCHEMA_VERSION)
   - 所有 event 都能反序列化
   - hash 校验(防止损坏)
   ↓
3. 构造 Session:
   - new Session(header)
   - 对每个 event 调 session.append(...)
   - SurfaceManager 自动 rebuild(从 events 重建 visible nodes)
   - requestHeader / requestContext 自动 fold
   ↓
4. session 处于完全 ACTIVE 状态,可直接 ctx.agents.resume(session)
不变式
SESSION_FORMAT_VERSION = 0(packages/core/session/src/types.ts:56)——磁盘格式版本号。
- 任何写入磁盘的 event 都带版本号
- 启动时检查版本,不兼容则拒绝加载
- 这就是 "Backends reject old on-disk formats" 的实现
三层保证
┌─────────────────────────────────────────────────────────┐
│ Layer 1: append-only log                                │
│   events 一旦 append,不能改                               │
│   repair.ts 处理崩溃恢复(TOOL_NOT_STARTED 等)              │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 2: schema version                                 │
│   SCHEMA_VERSION 单调,disk format 升级需手动 migration    │
│   SQLite 通过 schema_version 字段跟踪                     │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 3: hash / checksum                                │
│   SessionHeader 带 hash,读时校验                          │
│   损坏 → 抛 SessionPersistenceCorruptionError            │
└─────────────────────────────────────────────────────────┘
一句话
内存 session log(append-only events)
  ↕ flush / restore
磁盘 artifact(events + header + version + hash)
持久化 = 把内存数组搬到磁盘 + 启动时搬回来。

5. 如何从配置组合出一个完整的Agent? (profiles + bundles + pressets)
Profile = 顶层组合入口,Bundles = 可安装单元,Plugins = 实际挂载的代码,Presets = 单 agent 的 per-scope override。
组合树
Profile(用户选)
  └── 列出要加载的 bundles
       └── 每个 bundle 是一组 plugins + config
            └── 实际 mount 到 root ctx 的代码

cordis.yml(分层加载)
  ├── profile 的 bundles 配置
  ├── home 级别 patch(~/.dsh/)
  ├── 用户级别 patch
  └── --patch CLI overlay
       ↓
       ↓ 全部按顺序叠到 root ctx

Preset(per-agent)
  └── 在 agent scope 里 shadow 父级的 plugin 配置
       (一个 profile 可以包含多个 preset,每个 agent 用一个 preset)
三个关键概念
1. Profile(顶层入口)
cordis.yml 里的顶层 key,定义"我要哪些 bundle":
# ~/.dsh/profiles/web.yml
dsh.profile:
  bundles:
    - '@deepseek-ai/dsh-base'           # 必需
    - '@deepseek-ai/dsh-web-app'        # web UI 加这个
dsh 命令的入口(--profile web)读这个文件,按顺序加载 bundles。
2. Bundle(可分发单元)
每个 bundle 在自己的 package.json 声明 dsh.bundle:
// packages/bundle/web-app/package.json
{
  "name": "@deepseek-ai/dsh-web-app",
  "dsh": {
    "bundle": "./bundle.cordis.yml"
  }
}
bundle.cordis.yml 里是 plugins 列表:
# packages/bundle/web-app/bundle.cordis.yml
plugins:
  - name: webserver
    apply: '@deepseek-ai/dsh-host-webserver'
  - name: frontend
    apply: '@deepseek-ai/dsh-host-frontend-static'
  - name: hmr
    apply: '@deepseek-ai/dsh-client-hmr'
Bundle 是 npm 包,可以直接 publish、install——这就是为什么 DeepSeek Harness 可以装第三方 bundle。
3. Preset(per-agent composition)
预设 = 单 agent 的完整配方(LLM config + persona + tools + sections):
ctx.agents.create({
  preset: 'code-review-agent',
  // 或内联:
  // preset: { name: 'foo', sections: [...], tools: [...], llm: {...} }
})
preset 在自己的 agent scope 里 mount plugins / 写 sections,shadow 父级(profile-level)的同名配置。
加载顺序(Patch Layering)
底层  Profile 列出 bundles → bundle 的 plugins → 挂载
                                ↓
                              用户 home (~/.dsh/patches/*.yml) → patch
                                ↓
                              用户 cwd (./.dsh/patches/*.yml) → patch
                                ↓
顶层  --patch CLI 参数 → 临时 patch
Patch 替换 row by id,或插入新 row。这让用户可以不改 bundle 源码就 override 任何插件。
真实例子:web profile
# ~/.dsh/profiles/web.yml
dsh.profile:
  bundles:
    - '@deepseek-ai/dsh-base'
    - '@deepseek-ai/dsh-web-app'
    - '@deepseek-ai/dsh-acp'  # 想加 ACP 服务也加一行
启动 dsh --profile web,Cordis:
1. 加载 base bundle(model / tools / session / sandbox / approval / settings / credentials / telemetry)
2. 加载 web-app bundle(webserver / frontend / hmr)
3. 加载 acp bundle(acp 服务)
4. 应用 home / cwd / --patch 的所有 patch
5. 全部 mount 到 root ctx
一句话
Profile 选 bundles → bundles 挂载 plugins → patches 覆盖 row
                                              ↓
                                              + Presets(per-agent override)
                                              ↓
                                              一个完整的 Agent

6. 如何让人类审批Agent的危险操作? (approval + permission)
Tool 跑之前,framework 会先问"这个操作人类批不批?"——批了才跑,不批就停。这个"先问再跑"由 approval service 管。
4 个参与者
1. Tool(framework 的执行管道)
   ↓ 说"这个操作有点危险,我不敢自己跑,问 user 吧"
   
2. Approval Service(framework 的中间人)
   ↓ 把"问 user"这件事广播出去,等 user 决定
   
3. UI Provider(实际问 user 的界面)
   - Web:弹 dialog
   - CLI:在终端问
   - ACP:发给客户端
   - headless:没 user,默认拒绝
   
4. User
   ↓ 选 allow / deny / modify
决策怎么流转
Tool 想跑
  ↓
Tool 的 pre-execute 阶段返回"ask"
  ↓
Approval service 收到
  ↓ 先查 policy:有没有预设规则已经决定?
  ├─ policy 决定 allow → 直接放行,不打扰 user
  ├─ policy 决定 deny  → 直接拒绝,不打扰 user
  └─ policy 没决定    → 真的问 user
      ↓
      UI provider 弹 dialog / 问 stdin / 发客户端
      ↓
      user 决定
      ↓
      落库 session event(approval/asked + approval/decided)
      ↓
      Tool 拿到结果,继续或终止
三层 policy 覆盖
底层  预设默认值     ─ "这个 agent 通常允许 tool/A,问 tool/B"
中层  session 覆盖  ─ "这次会话里 tool/C 永远 ask"
顶层  runtime 决定  ─ "user 这次说 deny"
                  (优先级最高,但只是这一次)

# 第四部分 — Q & A

> 本部分收集对 Cordis 与 DeepSeek Harness 的核心问题。问题按"直击本质"程度排序。

