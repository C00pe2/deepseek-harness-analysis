# Cordis 框架深度解析

Status: 分析记录
日期: 2026-08-14
主题: vendored Cordis 框架在 DeepSeek Harness 中的集成

本文是对 `vendor/cordis/` 下 vendored Cordis 框架的研究/理解记录。包含第一性原理的推演、一个具体比喻,以及按特性拆解的实现分析(带源码定位)。

源码路径使用 `vendor/cordis/src/<file>.ts:<line>` 形式。所有断言都附带行号引用,读者可直接对照 vendored 源码验证。

---

# 第零部分 — 第一性原理:Cordis 要解决什么问题

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

**比喻**:
- isolate = 6 楼和 7 楼各有"会议室 A"(同名不同房间)
- intercept = CEO 公告:"所有会议室必须有投影仪"(自顶向下级联)
- mixin = 每层楼都能 `ctx.print(...)` 调前台打印机(service 方法伪装)

### 9. Reflect / Proxy(反射层)

**定义**:让 `ctx.foo` 自动解析 service 的 Proxy 机制。

**设计初衷**:经典 DI 要求 `ctx.inject('foo').getService('foo').method()`——啰嗦。
Proxy 让 `ctx.foo` 触发 DI,API 简洁;同时,服务方法被调时 `this` 自动跟调用者 ctx。

**比喻**:电梯的自动导航——你说"我要打印机",电梯根据当前位置自动找到最近的,
不需要你记"打印机的服务名是什么"。

### 10. Symbol(共享符号表)

**定义**:Cordis 用一组固定的 unique symbols 做协议标记。

**设计初衷**:框架内部协议标记(`isolate` / `intercept` / `shadow` 等)
不能撞用户起的属性名。用 Symbol 作 key,跨 realm 共享,字符串属性名互不干扰。

**比喻**:楼栋内部的"内部代号"——住户看不见的门牌号。物业靠代号管门,
外人靠名字找房,两套编号互不冲突。

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

> Alice(LLM provider)在 6 楼的小组被合并到 8 楼。

**阶段 1 — 搬入**
Alice 调用 `ctx.plugin(MyLLMPlugin, { model: 'v4' })`。物业给她一间房(Fiber),
钥匙环空着但已挂这层楼。状态:`PENDING`(等待依赖)。

**阶段 2 — 依赖到位**
Bob 在 5 楼声明 `inject = ['llm']`。Bob 开张,物业检查:Alice 的钥匙环需要 `llm`。
Bob 跑完,Alice 收到 `llm`。状态:`PENDING → LOADING → ACTIVE`。
epoch 变化:`'__INACTIVE__' → ':1'`(Bob 的 uid)。

**阶段 3 — 日常广播**
`ctx.events.emit('model-request', ...)`。PA 广播,所有订阅者并行触发。
Alice 的 listener 本身是注册过的 effect;这层楼拆除时,她的 listener 自动消失。

**阶段 4 — 换供应商**
物业通知:"LLM 供应商从 DeepSeek 换成 Pi-AI"。6 楼的 `llm` inject 被替换。
Bob 的 epoch 变化:`':1' → ':2'`。Bob `_unload` 跑所有 disposer,再 `_reload`。
Bob:`ACTIVE → UNLOADING → ACTIVE`。

**阶段 5 — 搬出**
物业通知:"8 楼合并完成,6 楼租户搬出"。`ctx.registry.delete(plugin)` 触发
`Alice.fiber.dispose()`。Alice 的 disposer 按反向插入顺序执行。
状态:`ACTIVE → UNLOADING → DISPOSED`。uid 置 null(再调 `ctx.effect()` 抛错)。

要点:**Alice 从未写过一行清理代码**。她只管注册,物业负责撤销。
这就是 Cordis 的核心承诺:**注册即效果——记下来,framework 收回**。

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

## 附录:源码索引

本附录供查证。所有概念 / 模块对应的源码位置集中在此,正文不重复。

| 概念 / 模块 | 源码定位 |
|---|---|
| Context | `vendor/cordis/src/context.ts:42-146` |
| Fiber | `vendor/cordis/src/fiber.ts:184-754` |
| Plugin | `vendor/cordis/src/registry.ts:92-146` |
| Service | `vendor/cordis/src/service.ts:11-115` |
| Event | `vendor/cordis/src/events.ts:131-352` |
| Effect / Disposable | `fiber.ts:74-93`、`utils.ts:5-40` |
| Isolation / Intercept / Mixin | `context.ts:121-125, 139-145`、`reflect.ts:364-390` |
| Reflect / Proxy | `vendor/cordis/src/reflect.ts:133-418` |
| Epoch | `vendor/cordis/src/fiber.ts:611-639` |
| Symbol | `vendor/cordis/src/utils.ts:50-73` |
| `context.ts` 模块 | `vendor/cordis/src/context.ts` |
| `service.ts` 模块 | `vendor/cordis/src/service.ts` |
| `fiber.ts` 模块 | `vendor/cordis/src/fiber.ts` |
| `events.ts` 模块 | `vendor/cordis/src/events.ts` |
| `registry.ts` 模块 | `vendor/cordis/src/registry.ts` |
| `reflect.ts` 模块 | `vendor/cordis/src/reflect.ts` |
| `utils.ts` 模块 | `vendor/cordis/src/utils.ts` |
| 桶导出 | `vendor/cordis/src/index.ts` |

---

# 第二部分 — 设计哲学:七个核心命题

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

# 第四部分 — 12 个重要特性:是什么、为什么、怎么做

> 每个特性按 **特征 → 动机 → 实现** 记录,带文件与行号引用。

## ① 五种分发模态

**特征**:`emit | parallel | serial | bail | waterfall`。

**动机**:plugin 协调至少是五种关系(通知 / 聚合 / 抢答 / 投票 / 中间件链)。只有 `emit` 一个模态,会迫使调用者在 listener 内部用闭包变量 hack 出超时或状态,业务逻辑散落多个 listener。分别命名,强迫调用者先想清楚"我要的是哪种模式"——而不是把所有可能的参数混在一个方法签名里。

**实现**(`vendor/cordis/src/events.ts`):

```ts
// events.ts:32
export type DispatchMode = 'emit' | 'parallel' | 'serial' | 'bail' | 'waterfall'
```

每个模态有自己的方法,共享一个入口:

- `emit`(`:194-196`):不 await listener 返回值,只调用。
- `parallel`(`:183-187`):`Promise.allSettled`,任一 reject 抛 `AggregateError`。
- `serial`(`:204-209`):for-await,首个 `isBailed(result)` 时 return。
- `bail`(`:217-222`):`serial` 的同步版本。
- `waterfall`(`:234-243`):`next()` 是最后一个参数;每个 listener 包裹下一个;不调 `next()` 否决链的剩余部分。

`isBailed`(`:13-15`):`value !== null && value !== false && value !== undefined`。按约定,**沉默即同意**。

## ② 注册即效果(Disposable + Effect 树)

**特征**:每个注册 API 返回 disposer;所有注册挂到 fiber 的 `_disposables`;fiber unload 反向执行。

**动机**:传统 framework 要求 plugin 作者维护 listener 列表、写清理钩子、处理"plugin A 被卸载,B 的依赖丢了"的级联卸载。漏一处就泄漏。Cordis 统一承诺:**只管注册,不管清理**。plugin 作者写函数体 `() => { /* 副作用 */ }`,framework 决定何时撤销。

**实现**(`vendor/cordis/src/fiber.ts:418-561`):

数据结构是 `DisposableList<T>`(`vendor/cordis/src/utils.ts:5-40`):

```ts
class DisposableList<T extends WeakKey> {
  private sn = 0
  private map = new Map<number, T>()          // serial → value
  private weak = new WeakMap<T, number>()     // value → serial
  push(value) { /* 返回删除函数,O(1) */ }
  clear() { return [...this.map.values()].reverse() }  // reverse 是 disposal 顺序
}
```

fiber 用这个 list 持有 `_disposables`。注册时:

```ts
// fiber.ts:520
removeWrapper = this._disposables.push(wrapper)
try { task = this._execute(runner) } catch { ... }
```

卸载时:

```ts
// fiber.ts:675-696
await Promise.all(this._disposables.clear().map(async (dispose) => {
  try { await runDisposable(dispose) }
  catch (reason) { this.ctx.logger.error(reason) }
}))
```

四个微妙之处:

1. **生成器 effect 增量收集**(`:375-395`):每个 yield 出的值立即被收集,所以中途崩溃的生成器也能清理。
2. **`effectInertia` WeakMap**(`:112`):让正在进行的清理可被其他 caller await,防止两条清理路径竞态。
3. **`setupBarrier`**(`:467-473`):async effect body 还在 setup 时,dispose 必须 await body 完成。
4. **基于 epoch 的过期跳过**(`:508-509`):如果新 epoch 已经替换当前,dispose 是 no-op。

## ③ 三种 plugin 形状 + `inject` 声明

**特征**:`Function | Constructor | Object` 三种 plugin 形状;统一的 `inject` 字段声明依赖。

**动机**:函数式最简单适合小型 plugin;类式适合 OO 风格 + 继承;对象式支持更复杂元数据(`Config`、`provide`、`intercept`)。统一 `inject` 让 framework **自动**推断加载顺序,而不是 plugin 自己协调。

**实现**(`vendor/cordis/src/registry.ts`):

```ts
// registry.ts:316-336
plugin(plugin, config, getOuterStack) {
  const callback = this.resolve(plugin)
  // ...
  let runtime = this._internal.get(callback)
  if (!runtime) {
    runtime = { name, callback, fibers: new DisposableList(), Config: plugin.Config }
    this._internal.set(callback, runtime)
  }
  const fiber = new Fiber(this.ctx, config, Inject.resolve(plugin.inject), runtime, getOuterStack)
  // ...
}
```

`Inject.resolve()`(`registry.ts:71-88`)归一化三种形式:

- 数组 `['a', 'b']`:无配置,只声明需要 `a` 和 `b`。
- 对象 `{ a: cfgA }`:带配置。
- 带 `symbols.checkProto` 的对象:通过原型继承父类 inject(decorator 模式)。

`isConstructor`(`vendor/cordis/src/utils.ts:79-89`)区分函数与类 plugin:

```ts
export function isConstructor(func) {
  if (!func.prototype) return false  // async / arrow
  if (func instanceof GeneratorFunction) return false
  if (AsyncGeneratorFunction !== Function && func instanceof AsyncGeneratorFunction) return false
  return true
}
```

`@Inject()` decorator(`registry.ts:37-60`)让类方法 **延迟到服务可用时执行**:

```ts
// registry.ts:46-55
decorator.addInitializer(function () {
  const property = this[symbols.tracker]?.property
  ;(this[symbols.initHooks] ??= []).push(() => {
    (this.ctx as Context).inject(inject, (ctx) => {
      return value.call(property ? withProps(this, { [property]: ctx }) : this)
    })
  })
})
```

## ④ 通过 Symbol 隔离

**特征**:`ctx.isolate(name, label?)` 为同名服务创建独立实例;Symbol 作 store key。

**动机**:同名服务在测试 vs 生产、或同一进程内的多个 agent 下可能需要不同实现。朴素的命名空间前缀污染 API。Symbol 键控的 store 自然分区。

**实现**(`vendor/cordis/src/context.ts:121-125` + `reflect.ts:277-305`):

```ts
// context.ts:121-125
isolate(name: string, label?: symbol) {
  const shadow = Object.create(this[symbols.isolate])
  shadow[name] = label ?? Symbol(name)
  return this.extend({ [symbols.isolate]: shadow })
}
```

生产侧:

```ts
// reflect.ts:286-287
this.ctx.root[symbols.isolate][name] ??= Symbol(name)
const key = this.ctx[symbols.isolate][name]
this.store[key] = impl
```

读取侧:

```ts
// reflect.ts:237-243
_getImpl(name, strict = true) {
  const key = this.ctx[symbols.isolate][name]
  const impl = key && this.store[key]
  if (!impl) return
  if (strict && impl.fiber.state !== FiberState.ACTIVE) return
  return impl
}
```

root 的 Symbol 在所有未 isolate 的子 context 间共享。子 context 调 `isolate()` 分配新 Symbol;store key 改变;JS 原型查找负责其余。

## ⑤ Intercept(根到叶配置合并)

**特征**:`ctx.intercept(name, config)` 通过原型链收集所有 ancestor 配置,根到叶方向,可用 `Config.merge` 时用 schemastery 语义合并。

**动机**:不同部署 / 用户偏好在不动 plugin 源码的情况下需覆盖配置。Props drilling 啰嗦;env vars 不可追溯。Intercept 让"在任何层加 override"成为 **一等公民操作**,合并方向、策略都明确。

**实现**(`vendor/cordis/src/service.ts:86-102`):

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

走法从当前 context 的 intercept 映射向上到 root,经 `Object.getPrototypeOf`。`hasOwn` 防止重复收集。`unshift` 保证根的条目先。

## ⑥ Fiber 状态机 + epoch 反应性

**特征**:`PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED`。状态转移由 **epoch 字符串** 驱动。

**动机**:plugin 的依赖图是动态的——服务被卸载、重载或替换,**所有依赖它的 plugin 都必须重新评估**。手动监听每个服务变化会写出"a 变了 → 通知 b → b 变了 → 通知 c"的级联代码,极脆弱。Epoch 机制:**epoch 是依赖图的指纹字符串**。epoch 变 → reload;epoch 不变 → 啥都不做。字符串相等性天然实现"依赖完全恢复"检测。

**实现**(`vendor/cordis/src/fiber.ts`):

`_refresh()` 计算 epoch(`fiber.ts:611-623`):

```ts
_refresh() {
  let epoch = ''
  for (const name of Object.keys(this.inject)) {
    const impl = this._store[name]
    if (!impl) { epoch = INACTIVE; break }
    epoch += ':' + impl.fiber.uid
  }
  this._setEpoch(epoch)
}
```

`_setEpoch(epoch)` 触发 reload/unload(`fiber.ts:625-639`):

```ts
private _setEpoch(epoch: string) {
  const oldEpoch = this._runner.epoch
  if (epoch === oldEpoch) return
  this._runner.epoch = epoch
  if (this.inertia) return
  this._updateState(() => {
    if (epoch !== INACTIVE && oldEpoch === INACTIVE) {
      this.inertia = this._reload()
      return FiberState.LOADING
    } else {
      this.inertia = this._unload()
      return FiberState.UNLOADING
    }
  })
}
```

`_reload()` 和 `_unload()`(`:646-696`)各自 `await Promise.resolve()` 先让出微任务——给其他 caller 抢先 unload 的机会,避免过期 load。

`INACTIVE = '__INACTIVE__'`(`:176`)是哨兵,代表"依赖缺失"。

## ⑦ 基于 Proxy 的 Context + Traceable

**特征**:`ctx.foo` 自动解析服务;服务方法的 `this` 自动绑定到调用者 ctx。

**动机**:经典 DI 需要显式 `ctx.inject('foo')`——啰嗦。Cordis 用 Proxy 让 `ctx.foo` 触发 DI——**这是 Cordis 看起来像"动态属性"的魔法**。

更进一步:服务方法被调用时,它的 `this` 应该是 **调用者** 的 ctx(谁在调它),不是 **服务提供者** 的 ctx。这样 logger 能记录"哪个 plugin 在调用我"。

**实现**(`vendor/cordis/src/reflect.ts:135-206` 是 Proxy handler):

```ts
get: (target, prop, ctx) => {
  if (isSpecialProperty(prop)) return Reflect.get(target, prop, ctx)
  if (Reflect.has(target, prop)) return getTraceable(ctx, Reflect.get(target, prop, ctx))

  const error = new Error(`cannot get property "${prop}" without inject`)
  const def = target.reflect.props[prop]
  if (def?.type === 'accessor') return def.get.call(ctx, ctx[symbols.receiver], error)

  if (!ctx.fiber.runtime) return ctx.reflect.get(prop, false)
  return ctx.events.waterfall('internal/get', ctx, prop, error, () => {
    const key = target[symbols.isolate][prop]
    let fiber = (ctx[symbols.shadow] ?? ctx).fiber
    while (true) {
      const impl = fiber.store?.[prop]
      if (impl) return getTraceable(ctx, impl.value)
      if (prop in fiber.inject) {
        error.message = `cannot get required service "${prop}" in inactive context`
        throw error
      }
      if (!fiber.runtime) throw error
      if (fiber.parent[symbols.isolate][prop] !== key) throw error
      fiber = fiber.parent.fiber
    }
  })
}
```

`getTraceable(ctx, value)`(`utils.ts:117-125`)把值包一层 Proxy,让方法调用看到调用者 ctx:

```ts
// utils.ts:165-218
function createTraceable(ctx, value, tracker) {
  if (ctx[symbols.shadow] && !tracker.noShadow) {
    ctx = Object.getPrototypeOf(ctx)
  }
  return new Proxy(value, {
    get(target, prop, receiver) {
      if (prop === symbols.original) return target
      if (prop === tracker.property) return ctx   // this.ctx = 调用者 ctx
      // ...
    },
  })
}
```

`createShadowMethod`(`utils.ts:156-163`)是调用时的技巧:

```ts
function createShadowMethod(ctx, value, outer, shadow) {
  return new Proxy(value, {
    apply: (target, thisArg, args) => {
      if (thisArg === outer) thisArg = shadow
      return getTraceable(ctx, Reflect.apply(target, thisArg, args))
    },
  })
}
```

`noShadow` 存在因为 logger(`reflect.ts:200-203`)需要原始 fiber 来 derive 名字,它不剥离 shadow。

## ⑧ Listener 的 scope 过滤

**特征**:每个 `Hook` 携带 `ctx`;dispatch 应用 `thisArg[Context.filter]` 来 admit 或跳过每个 listener。

**动机**:scope 模块需要 **祖先 listener 收到 descendant 事件,反之不然**。给每个 scope 复制一份完整 listener 列表,内存爆炸;不复制,scope A 注册的 listener 会被 scope B 的事件触发,违背"scope B 看不见 A 的状态"。filter 机制:**dispatch 时检查 listener.ctx 与 dispatch 起点 ctx 的关系**,祖先 listener 通过,descendant listener 不通过——同一份 listener 列表支持双向过滤。

**实现**(`vendor/cordis/src/events.ts:165-175`):

```ts
dispatch(type, args) {
  const thisArg = ...args.shift()
  const name = args.shift()
  if (!name.startsWith('internal/')) {
    this.emit('internal/dispatch', type, name, args, thisArg)
  }
  const filter = thisArg?.[Context.filter]
  return (this._hooks[name] || [])
    .filter(hook => hook.global || !filter || filter.call(thisArg, hook.ctx))
    .map(hook => hook.callback.bind(thisArg))
}
```

`hook.global`(`events.ts:114-117`)是跨 scope listener 的逃逸口。

`scopeTarget(base, key)`(`packages/core/scope/src/index.ts:170-185`)构造一个带 filter 的 ctx:

```ts
scopeTarget(base, key): Context {
  return base.fiber.context.extend({
    [Context.filter]: (target: Context) => {
      for (cursor = key; cursor !== undefined; cursor = scopeParents.get(cursor)) {
        if (target.fiber === ...) return true
      }
      return false
    },
  })
}
```

祖先 admit;后代不通过。

## ⑨ HMR 与配置热重载

**特征**:`Fiber.update(config)` 走 `internal/update` waterfall;vendored `hmr` 插件监听文件变化触发 reload。

**动机**:开发者改 `cordis.yml` 不希望手动重启服务。Cordis 提供 reload 协议,让配置变化 → 自动 update → 触发 reload,同时允许中间件(persistence、validation)拦截。

**实现**(`vendor/cordis/src/fiber.ts:736-753`):

```ts
update(config, noSave = false) {
  this.assertActive()
  this._config = config
  if (this.state !== FiberState.ACTIVE) {
    this._error = undefined
    this._setEpoch(INACTIVE)
    this._refresh()
    return
  }
  config = this._resolveConfig(config)
  return this.context.waterfall(this, 'internal/update', config, noSave, () => {
    this.config = config
    this._error = undefined
    return this.restart()
  })
}
```

`EventsService.on('internal/update', ...)`(`events.ts:148-155`)预装一个 global listener,强制 update 走瀑布链:

```ts
this.on('internal/update', function (config, noSave, next) {
  const cbs = [...this._hooks['internal/update'] || []]
  const _next = () => {
    const cb = cbs.shift() ?? next
    return cb.call(this, config, noSave, _next)
  }
  return _next()
}, { global: true, prepend: true })
```

Persistence 插件在 `internal/update` 上注册 listener,持久化新 config 或否决变更。

`hmr` 插件(`vendor/hmr/src/index.ts:134` 的 `registerConfig(filename, refresh)`)监听文件 watcher,调用 `fiber.update(newConfig)`,自动 reload。

## ⑩ Listener 跟随 fiber 自动清理

**特征**:`ctx.on()` 返回的 disposer 是 fiber 的 effect;fiber unload 时 listener 同步移除。

**动机**:事件总线最常见的泄漏:**listener 还在订阅,但触发它的对象已经死了**。Cordis 把 listener 注册 = effect,根除泄漏。

**实现**(`vendor/cordis/src/events.ts:254-302`):

```ts
// events.ts:254-260
register(label, hooks, callback, options) {
  const method = options.prepend ? 'unshift' : 'push'
  return this.ctx.fiber.effect(() => {
    hooks[method]({ ctx: this.ctx, callback, ...options })
    return () => this.unregister(hooks, callback)
  }, label)
}

// events.ts:288-302
on(name, listener, options) {
  // ...
  listener = this.ctx.reflect.bind(listener)   // trace this 与 args
  const result = this.bail(this.ctx, 'internal/listener', name, listener, options)
  if (result) return result

  const hooks = this._hooks[name] ||= []
  const label = `ctx.on(${JSON.stringify(name)})`
  return this.register(label, hooks, listener, options)
}
```

`ctx.reflect.bind(listener)`(`reflect.ts:408-417`)是 listener 的特殊处理:

```ts
bind(callback) {
  return new Proxy(callback, {
    apply: (target, thisArg, args) => {
      return Reflect.apply(target, this.trace(thisArg), args.map(arg => this.trace(arg)))
    },
  })
}
```

——listener 的 `this` 和参数都被 trace 包裹,使 listener 内部调用 `ctx` 自动解析到自己的 fiber ctx。

## ⑪ 内部事件总线(framework 自协调)

**特征**:9 个 `internal/*` 事件——`internal/dispatch`、`internal/plugin`、`internal/status`、`internal/listener`、`internal/service`、`internal/get`、`internal/set`、`internal/update`、`internal/config`。

**动机**:framework 自身的协调点和业务事件走同一套事件总线,**第三方可以用同样模式观察/拦截 framework 内部行为**。

**实现**(`vendor/cordis/src/events.ts:329-352`):

```ts
export interface Events {
  'internal/plugin'(fiber: Fiber): void
  'internal/status'(fiber: Fiber, oldValue: FiberState): void
  'internal/config'(this: Fiber, config: any, next: () => any): any
  'internal/service'(this: Context, name: string, value: any): void
  'internal/update'(this: Fiber, config: any, noSave: boolean, next: () => void | Promise<void>): void | Promise<void>
  'internal/get'(ctx: Context, name: string, error: Error, next: () => any): any
  'internal/set'(ctx: Context, name: string, value: any, error: Error, next: () => boolean): boolean
  'internal/listener'(this: Context, name: string, listener: any, prepend: boolean): void
  'internal/dispatch'(mode: DispatchMode, name: string, args: any[], thisArg: any): void
}
```

每个事件都通过 `@mode` JSDoc 声明模式——这是约定:`internal/get` 是 waterfall,`internal/dispatch` 是 emit。**listener 实现必须知道模式才能正确调用 `next()` 或返回值**。

`internal/dispatch` 在 `dispatch()`(`events.ts:165-175`)开头主动 emit,但 **仅对非 internal 事件触发**:

```ts
// events.ts:168-170
if (!name.startsWith('internal/')) {
  this.emit('internal/dispatch', type, name, args, thisArg)
}
```

——internal 事件的派发本身不被 trace,避免无限递归。

## ⑫ Mixin:把服务方法搬到 ctx 上

**特征**:`ctx.on`、`ctx.plugin`、`ctx.emit` 等"看起来像 ctx 原生方法",实际转发到 service method。

**动机**:不写 mixin,调用方必须 `ctx.events.on(...)`、`ctx.registry.plugin(...)`——啰嗦。写 mixin,API 简洁,但代价是"看不到转发关系"。

Cordis 让 **99% 的调用是 `ctx.foo(...)`,1% 需要直接访问 service 时是 `ctx.events.on(...)`**——渐进暴露。

**实现**(`vendor/cordis/src/reflect.ts:219-222` + `:364-390`):

构造函数中预注册:

```ts
this.mixin('reflect', ['get', 'set', 'provide', 'accessor', 'mixin'])
this.mixin('fiber', ['runtime', 'effect'])
this.mixin('registry', ['inject', 'plugin'])
this.mixin('events', ['on', 'once', 'parallel', 'emit', 'serial', 'bail', 'waterfall'])
```

`mixin(source, mixins)` 是生成器 effect:

```ts
// reflect.ts:364-390
mixin(source, mixins) {
  return this.ctx.fiber.effect(function* () {
    const entries = Array.isArray(mixins) ? mixins.map(k => [k, k]) : Object.entries(mixins)
    for (const [key, value] of entries) {
      yield self.accessor(value, { get, set })
    }
  }, `ctx.mixin(${JSON.stringify(source)})`)
}
```

`accessor(name, options)`(`reflect.ts:345-353`)创建 computed property:

```ts
return this.ctx.fiber.effect(() => {
  this.props[name] = { type: 'accessor', ...options }
  return () => delete this.props[name]
}, `ctx.accessor(${JSON.stringify(name)})`)
```

Proxy handler 的 `get` 见到 `def?.type === 'accessor'` 就调 `def.get.call(ctx, ctx[symbols.receiver], error)`——**所以 `ctx.on` 实际是 `ctx.events.on.bind(ctx.events)` 通过 accessor 实现**。

---

# 第五部分 — 汇总表

| # | 特性 | 解决什么 | 核心实现位置 |
|---|---|---|---|
| 1 | 五种 dispatch 模态 | 协调语义独立,不靠参数化 | `events.ts:32, 183-243` |
| 2 | Disposable effect | 注册即清理,根因解决泄漏 | `fiber.ts:418-561`, `utils.ts:5-40` |
| 3 | 三种 plugin 形状 + inject | 统一依赖声明,自动加载顺序 | `registry.ts:71-88, 316-336` |
| 4 | Symbol 隔离 | 同名多实例无命名污染 | `context.ts:121-125`, `reflect.ts:237-243` |
| 5 | Intercept 配置合并 | 根到叶覆盖,可追溯 | `service.ts:86-102` |
| 6 | Epoch 反应性 | 依赖变化自动 reload | `fiber.ts:611-639, 646-696` |
| 7 | Proxy + Traceable | DI 无感 + 自动 `this` 绑定 | `reflect.ts:135-206`, `utils.ts:117-218` |
| 8 | Scope filter | 祖先 listener admit 后代事件 | `events.ts:165-175` |
| 9 | HMR / 配置更新 | 可拦截的热重载 | `fiber.ts:736-753`, `events.ts:148-155` |
| 10 | Listener 自动清理 | 跟随 fiber unload | `events.ts:254-302` |
| 11 | 内部事件总线 | framework 自协调复用同一总线 | `events.ts:329-352` |
| 12 | Mixin | 服务方法伪装 ctx 方法 | `reflect.ts:219-222, 364-390` |

十二个特性共享一个底层约束:**framework 的任何状态变化必须可被 listener 观察、可被 effect 撤销**。这条约束让 Cordis 避免"framework 代码 + 业务代码"两条独立路径——timeout policy 是 listener;tool call cancellation 是 effect;fiber 状态机驱动两者。

# 第六部分 — 一句话总结

Cordis 的核心思想不是"plugin framework",而是 **"原型链作用域上的 disposable effect"**。其余都是在这对原语上的工程精修。
