# Cordis 框架深度解析

Status: 分析记录
日期: 2026-08-14
主题: vendored Cordis 框架在 DeepSeek Harness 中的集成

本文是对 `vendor/cordis/` 下 vendored Cordis 框架的研究/理解记录。包含第一性原理的推演、一个具体比喻,以及按特性拆解的实现分析(带源码定位)。

源码路径使用 `vendor/cordis/src/<file>.ts:<line>` 形式。所有断言都附带行号引用,读者可直接对照 vendored 源码验证。

---

# 第一部分 — 第一性原理:Cordis 要解决什么问题

## 组合问题

Cordis 围绕一个问题展开:**如何让独立的、可复用的能力单元(plugin)在一个共享上下文里既能看见彼此,也能被可控地独立加载/卸载/重载,且整个生命周期对调用者透明?**

这个问题隐藏三个相互耦合的子问题:

1. **可见性(依赖)**:plugin A 需要 plugin B 的能力,但又不想在编译期硬编码具体实现。
2. **时序(生命周期)**:plugin 的加载顺序未定;插件可能要异步加载,必须等待依赖、必须优雅卸载、必须不泄漏地重载。
3. **空间(隔离)**:同一个服务名在不同场景(测试 vs 生产、本地 vs 远程)下指向不同实现;它们互不污染。

想象一下，传统软件工程中，当你需要上线一个新服务的时候，你需要：先停机，写代码，合代码，部署发布，上线测试；为了避免架构大改，因此在设计之初就要尽可能解耦，避免逻辑耦合导致架构演进困难。

而Cordis的解决方案是运行时加载+插件可逆热插拔。即不用停机，就可以在运行时加载、卸载、重载插件。而每个插件在卸载时，会干净地、完整地把自己引入的所有内容都清理掉。

经典 DI 框架(Guice、Spring、NestJS)在 (1) 上做得好。(2) 和 (3) 通常需要手写 `start`/`stop` 钩子加配置 profile —— 当 plugin 数量大、重载频繁、HMR 介入时,这种手工钩子会脆裂。

## Cordis 的统一答案

> 把"plugin 的存在"和"plugin 的清理"做成同一个可组合对象(`Disposable`);把它挂在一个生命周期节点(`Fiber`)上;让可见性走原型链加一个 Symbol 键控的隔离映射;让协调走一个五模态事件分发器。

换句话说:**Cordis 用一对原语(Disposable + 原型链)同时表达依赖、生命周期和隔离**,避免"DI 框架"与"生命周期框架"的概念分裂。

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

# 第三部分 — 具体比喻:共享办公楼

> 用具体事物替换抽象概念,让每个机制变得可触摸。

## 楼栋

一个 **Cordis context 是一栋共享办公楼**:

- **楼栋本身**是根 context。它有前台、广播系统、住户登记册、日志室。
- 每个 **楼层**是一个 fiber——一个 plugin 实例。
- 楼层之间通过 **电梯**(原型链)连通:5 楼想要打印机时,电梯自动先去 6 楼、再去楼顶找。
- 一层楼只能看到自己楼层有的东西,除非它明确 inject 了上面的服务。

## 注册 = 交钥匙

Alice(一个 LLM provider)搬进来时:

```ts
ctx.on('meeting-start', () => console.log('hi'))
ctx.effect(() => {
  setInterval(() => logger.info('日报'), 1000)
  return () => clearInterval(...)
})
```

每登记一项,物业给她一把钥匙。钥匙上挂着标签:"这层楼拆除时,先做 X"。

Alice 不需要记得什么时候取消任何东西。物业保管所有钥匙。当 Alice 搬出(她的 fiber 被 dispose),物业 **按反向顺序用完所有钥匙**,然后归档这层楼。

与 `try { ... } finally { ... }` 的对比:那种模式要求 Alice 记得在哪一层清理。**Cordis 把"清理"从 plugin 作者的责任移交给 framework。**

## PA 系统(事件)

楼栋有一台 PA 系统,有 **五种使用方式**:

| 模态 | 你说的话 | 发生什么 |
|---|---|---|
| `emit` | "下班了!" | 所有人都听到,没人等待 |
| `parallel` | "5 点全员开会" | 所有人并行响应,你等所有人到齐 |
| `serial` | "第一个同意的举手" | 你按顺序问,第一个同意的胜出 |
| `bail` | "有人反对吗?" | 同步投票,首个反对胜出 |
| `waterfall` | "3 号线 Bob 决定放行" | Bob 不接电话 → 否决;Bob 接 → 下一个人 |

这些不是"快/慢"变体。它们是 **不同的社交协议**。Cordis 给它们分别命名,让调用者被迫选对。

## 楼层隔离

6 楼和 7 楼都有"会议室 A",但它们是不同的房间:

```
6F 会议室 A → CEO 专用
7F 会议室 A → 部门专用,与 6 楼无关
```

```ts
ctx.isolate('meeting-room-a')  // 创建一个新"分身",从此以下是新世界
```

技术上:物业给"会议室 A"分配一个 Symbol 作门牌号。6 楼用原 Symbol;7 楼用新 Symbol。登记册 `reflect.store` 用 Symbol 作 key——电梯自动查找当前楼层对应的房间。

## Intercept(自顶向下规则级联)

今天 CEO 宣布:"所有会议室从现在起必须有投影仪":

```ts
ctx.intercept('meeting-room', { projector: true })
```

规则按 **自顶向下** 叠加:楼顶先,6 楼次之,7 楼再次,plugin 自己的 inject 最后:

```
[ 6F 默认 { chair: 4 } ]          ← 最先
+ [ 7F { projector: true } ]      ← 然后
+ [ 8F { mic: 2 } ]               ← 最后
= { chair: 4, projector: true, mic: 2 }
```

每个 plugin 不需要知道 CEO 的全局规则。它只声明"读我的 intercept 层 + 我自己的 inject",级联自动发生。

## Mixin(共享打印机)

6 楼前台有一台打印机,但每层楼都想直接 `ctx.print(...)`:

```ts
this.mixin('registry', ['plugin'])  // 把 `registry.plugin` 抄到 ctx 上
```

技术上:物业在每层楼装一个 **自动转发器** —— `ctx.plugin` 实际是 `ctx.registry.plugin.bind(ctx.registry)`。服务方法伪装成 context 原生方法;这是 Cordis API 能读起来像 `ctx.foo` 而不是 `ctx.inject('foo').getService('foo').method()` 的关键。

## 完整生命周期故事

> Alice 在 6 楼的小组被合并到 8 楼。

```
阶段 1 — 搬入
  Alice 调用 ctx.plugin(MyLLMPlugin, { model: 'v4' })
  → 物业给她一间房(Fiber)
  → 她的钥匙环还空着,但已经挂在这层楼上
  → 状态:PENDING(等待依赖)

阶段 2 — 依赖到位
  Bob 在 5 楼声明 inject = ['llm']
  → Bob 这层楼开张;物业检查:Alice 的钥匙环需要 `llm`
  → Bob 跑完;Alice 收到 `llm`
  → Alice 状态:PENDING → LOADING → ACTIVE
  → epoch 字符串变化:'__INACTIVE__' → ':1'(Bob 的 fiber uid)

阶段 3 — 日常广播
  ctx.events.emit('model-request', ...)
  → PA 广播
  → 所有订阅者并行触发
  → Alice 的 listener 本身是注册过的 effect;这层楼拆除时,她的 listener 自动消失

阶段 4 — 换供应商
  物业通知:"LLM 供应商从 DeepSeek 换成 Pi-AI"
  → 6 楼的 `llm` inject 被替换
  → Bob 的 epoch 变化:':1' → ':2'
  → Bob 的 epoch ≠ _runner.epoch → _unload()
  → Bob:ACTIVE → UNLOADING → 所有 disposer 跑 → ACTIVE → _reload()

阶段 5 — 搬出
  物业通知:"8 楼合并完成,6 楼租户搬出"
  → ctx.registry.delete(plugin)
  → 物业调用 Alice 的 fiber.dispose()
  → Alice 的 disposer 按反向插入顺序执行
  → 状态:ACTIVE → UNLOADING → DISPOSED
  → uid 置 null(再调 ctx.effect() 抛 INACTIVE_EFFECT)
```

要点:**Alice 从未写过一行清理代码**。她只管注册,物业负责撤销。这就是 Cordis 的核心承诺:"注册即效果——记下来,framework 收回。"

## DeepSeek Harness 在 Cordis 上加了什么

Cordis 是 **物业 + 楼层 + 钥匙 + PA**。DeepSeek Harness 是在这上面盖的 **有具体业务的大楼**——它要解决"agent 与 LLM 对话"这件事:

```
Cordis 提供           DeepSeek Harness 加什么
─────────────────────────────────────────────────────
DI                →   18 个 capability seam
                     (LLM / Shell / FS / Subagent / ...)
event bus         →   3 个 SessionEvent 类型域
                     (session/event 模型真相源)
                     (agent/* 实时协调)
                     (capability/* 策略钩子)
fiber 生命周期    →   turn / step 循环
                     (一个 turn = 0+ 个 step)
                     (inbox / claim / reject / drive)
prototype scope   →   Layered Scope
                     (可见性向下,事件向上)
                     (skill registry 的 scoped layers)
disposable        →   append-only Session 日志
                     (turn 结束不能撤,只能补偿)
```

扩展比喻:Cordis 提供"钥匙环协议"。DeepSeek Harness 加"会议录音机"——每次会议(session turn)都永久写档,即使房间被拆(session turn 结束)录音也保留。DeepSeek Harness 还加了"应急守则"——agent 只能在自己的小范围内玩(scope)。

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