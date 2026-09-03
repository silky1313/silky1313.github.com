title: PI分析-插件系统
date: 2026-9-4 23:07:00
categories: 

- agent

---

（本文大幅度使用ai进行润色）

这里我们主要来看看pi的插件系统

先说结论，看完再回来体会：`agent.ts` 是底层 loop core，它压根不知道「插件」的存在，只认 `beforeToolCall` / `afterToolCall` 这类回调。**整个插件系统是** `AgentSession` **这一层搭起来的，它才是插件的宿主。** 所以这篇的主线是：插件从哪来 → 谁托管它 → 它怎么在正确的时刻被调用。

# 插件和 tool 到底啥区别？

这个问题不搞清楚，后面会一直拧巴。它俩是**包含关系**：

- **tool**：给**大模型**用的一个能力。模型自己决定调用（发 `toolCall`），运行时执行它的 `execute`。服务对象是 LLM。
- **extension（插件）**：一个 TypeScript 模块，用来扩展 **pi 宿主自身**的行为。工具只是它能注册的东西**之一**。

看一眼插件能干的事，`registerTool` 只占一条：


| 插件能做    | API                                           | 谁触发        |
| ------- | --------------------------------------------- | ---------- |
| 注册工具    | `pi.registerTool()`                           | 模型调用       |
| 拦截/修改事件 | `pi.on("tool_call", ...)` → `{ block: true }` | 事件发生时      |
| 注册命令    | `pi.registerCommand()`                        | 用户敲 `/xxx` |
| 交互 UI   | `ctx.ui.confirm/select`                       | 插件逻辑里      |
| 持久化状态   | `pi.appendEntry()`                            | 插件逻辑里      |


所以一个插件可以**一个工具都不注册**——比如纯做危险命令拦截的插件，只挂个 `pi.on("tool_call")` 钩子。这也说明：tool 可以脱离插件存在（内置工具就没插件），插件也可以不提供 tool。**插件站在工具之上，能决定工具要不要跑、结果改不改。**

# 一个插件长什么样

先建立直观印象。一个插件就是一个默认导出的工厂函数，参数 `pi` 就是 `ExtensionAPI`：

```ts
// 参考这个目录packages/coding-agent/docs/extensions.md
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // 1. 订阅事件
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Extension loaded!", "info");
  });

  // 2. 拦截工具调用
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("危险!", "允许执行 rm -rf 吗?");
      if (!ok) return { block: true, reason: "Blocked by user" };
    }
  });

  // 3. 注册一个给模型用的工具
  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "...",
    parameters: Type.Object({ name: Type.String() }),
    async execute(_id, { name }) {
      return { content: [{ type: "text", text: `Hello ${name}` }], details: undefined };
    },
  });
}
```

三种典型能力都在这了：**订阅事件（**`on`**）、拦截事件（**`on` **返回 block）、注册工具（**`registerTool`**）**。插件放在 `~/.pi/agent/extensions/`（全局）或 `.pi/extensions/`（项目级），会被自动发现。

# 插件从哪来：loader.ts

`loader.ts` 负责「把磁盘上的 `.ts` 文件变成运行时的插件对象」。

从入口方法开始看
```ts
/**
 * Discover and load extensions from standard locations.
 */
export async function discoverAndLoadExtensions(
	configuredPaths: string[],
	cwd: string,
	agentDir: string = getAgentDir(),
	eventBus?: EventBus,
): Promise<LoadExtensionsResult> {
	const resolvedCwd = resolvePath(cwd);
	const resolvedAgentDir = resolvePath(agentDir);
	const allPaths: string[] = [];
	const seen = new Set<string>();

	const addPaths = (paths: string[]) => {
		for (const p of paths) {
			const resolved = path.resolve(p);
			if (!seen.has(resolved)) {
				seen.add(resolved);
				allPaths.push(p);
			}
		}
	};

	// 1. Project-local extensions: cwd/${CONFIG_DIR_NAME}/extensions/
	const localExtDir = path.join(resolvedCwd, CONFIG_DIR_NAME, "extensions");
	addPaths(discoverExtensionsInDir(localExtDir));

	// 2. Global extensions: agentDir/extensions/
	const globalExtDir = path.join(resolvedAgentDir, "extensions");
	addPaths(discoverExtensionsInDir(globalExtDir));

	// 3. Explicitly configured paths
	for (const p of configuredPaths) {
		const resolved = resolvePath(p, resolvedCwd, { normalizeUnicodeSpaces: true });
		if (fs.existsSync(resolved) && fs.statSync(resolved).isDirectory()) {
			// Check for package.json with pi manifest or index.ts
			const entries = resolveExtensionEntries(resolved);
			if (entries) {
				addPaths(entries);
				continue;
			}
			// No explicit entries - discover individual files in directory
			addPaths(discoverExtensionsInDir(resolved));
			continue;
		}

		addPaths([resolved]);
	}

	return loadExtensions(allPaths, resolvedCwd, eventBus);
}
```

优先级从上到下：项目级 → 全局 → 显式配置。`discoverExtensionsInDir` 支持三种形态：直接的 `*.ts` 文件、带 `index.ts` 的子目录、带 `package.json`（含 `pi` 字段）的复杂包。具体你可以看看discoverExtensionsInDir和resolveExtensionEntries方法

## 加载插件
这里已经获取了所有path，接下来我们看看loadExtensions是怎么加载所有插件的
```ts
async function loadExtension(
	extensionPath: string,
	cwd: string,
	eventBus: EventBus,
	runtime: ExtensionRuntime,
	cacheToken?: ExtensionCacheToken,
): Promise<{ extension: Extension | null; error: string | null }> {
	const resolvedPath = resolvePath(extensionPath, cwd, { normalizeUnicodeSpaces: true });

	try {
    // 一种技术，导入未编译的ts，这里是导入了插件的export default function (pi: ExtensionAPI)这个方法
		const factory = await loadExtensionModule(resolvedPath, cacheToken);
		time(`${extensionPath} module import`, "extensions");
		if (!factory) {
			return { extension: null, error: `Extension does not export a valid factory function: ${extensionPath}` };
		}

		const extension = createExtension(extensionPath, resolvedPath);
		const api = createExtensionAPI(extension, runtime, cwd, eventBus);
		await factory(api);
		time(`${extensionPath} factory`, "extensions");

		return { extension, error: null };
	} catch (err) {
		const message = err instanceof Error ? err.message : String(err);
		return { extension: null, error: `Failed to load extension: ${message}` };
	}
}
```

关键在这里：`factory(api)` **就是在执行你写的那个** `export default function(pi)`，而传进去的 `pi` 就是 `createExtensionAPI` 造出来的 `api`。所以插件里调的 `pi.on(...)` / `pi.registerTool(...)`，本质是往这个 `api` 对象上挂东西。

看 `createExtensionAPI` 干了啥，就明白 `pi.on` / `pi.registerTool` 到底改了什么：

```ts
const api = {
  on(event: string, handler: HandlerFn): void {
    runtime.assertActive();
    const list = extension.handlers.get(event) ?? [];
    list.push(handler);
    extension.handlers.set(event, list);   // ← 把 handler 存进 extension.handlers
  },

  registerTool(tool: ToolDefinition): void {
    runtime.assertActive();
    extension.tools.set(tool.name, {         // ← 把 tool 存进 extension.tools
      definition: tool,
      sourceInfo: extension.sourceInfo,
    });
    runtime.refreshTools();
  },
  // ...
};
```

所以一个插件加载完，产物就是一个 `Extension` 对象，里面两个 Map 最关键：

- `handlers`：`事件名 → 回调列表`（`on` 往里塞）
- `tools`：`工具名 → 工具定义`（`registerTool` 往里塞）

注意 `registerTool` 里还调了 `runtime.refreshTools()`——这就接回了上一篇的工具装配链（`_refreshToolRegistry`），插件新注册的工具会被重新收拢进注册表。

当然这个插件api内部还有更多的方法，Extension对象内部还有更多的属性，不过这个这个后续我们再探索

# 中枢：ExtensionRunner （本章节大部分由ai生成，感觉写的很好）

插件加载完是一盘散沙，得有人托管它们、在合适的时刻挨个调用。这就是 `runner.ts` 的 `ExtensionRunner`——**插件系统跳动的心脏**。

它托管了所有 `Extension`，对外暴露的核心方法就三类。

## hasHandlers：有没有人关心这个事件

```ts
hasHandlers(eventType: string): boolean {
  // 遍历所有 extension，看有没有谁注册了这个事件
  const handlers = ext.handlers.get(eventType);
  // ...
}
```

这是个**性能短路**：宿主 emit 之前先问一句「有插件订阅吗」，没人订阅就直接跳过，不白跑。

## emit：把事件广播给所有订阅者

```ts
async emit<TEvent extends RunnerEmitEvent>(event: TEvent): Promise<RunnerEmitResult<TEvent>> {
  const ctx = this.createContext();
  let result: SessionBeforeEventResult | undefined;

  for (const ext of this.extensions) {
    const handlers = ext.handlers.get(event.type);
    if (!handlers || handlers.length === 0) continue;

    for (const handler of handlers) {
      try {
        const handlerResult = await handler(event, ctx);
        // ... 处理可取消的 before 事件
      } catch (err) {
        // 插件抛错不会搞崩宿主，只记录 error
        this.emitError({ extensionPath: ext.path, event: event.type, /* ... */ });
      }
    }
  }
  return result as RunnerEmitResult<TEvent>;
}
```

模式很清晰：**遍历所有插件 → 取出订阅了该事件的 handler → 依次** `await` **调用 → 传入** `event` **和** `ctx`。两个细节值得记：

- `createContext()` 造出的 `ctx` 就是插件回调里第二个参数（能调 `ctx.ui`、`ctx.actions` 反向操作宿主）。
- handler 抛错被 try/catch 兜住，只 `emitError` 记录——**一个插件挂掉不连累宿主和其他插件**。



## emitToolCall：能「拦截」的特殊 emit

普通 `emit` 是广播，但工具拦截需要「谁说了 block 就立刻停」，所以单独有个 `emitToolCall`：

```ts
async emitToolCall(event: ToolCallEvent): Promise<ToolCallEventResult | undefined> {
  const ctx = this.createContext();
  let result: ToolCallEventResult | undefined;

  for (const ext of this.extensions) {
    const handlers = ext.handlers.get("tool_call");
    if (!handlers || handlers.length === 0) continue;

    for (const handler of handlers) {
      const handlerResult = await handler(event, ctx);
      if (handlerResult) {
        result = handlerResult as ToolCallEventResult;
        if (result.block) {
          return result;   // ← 有人 block 就短路返回，后面的插件不再问
        }
      }
    }
  }
  return result;
}
```

这就是 `pi.on("tool_call")` 返回 `{ block: true }` 能拦住工具的底层实现——**任何一个插件说 block，立即返回，工具就不执行了。**

这里还很多的特殊emitXXX，这个逻辑与普通的emit逻辑有些不同，所以需要单独一个方法写逻辑

# AgentSession 如何把这一切接进主流程

回到宿主。`AgentSession` 和插件系统的关系，就三根骨架。

## 骨架一：插件的「诞生」

在 `_buildRuntime` 尾部创建 runner：

```ts
const extensionsResult = this._resourceLoader.getExtensions();  // 底层走 loader.ts

this._extensionRunner = new ExtensionRunner(
  extensionsResult.extensions,
  extensionsResult.runtime,
  this._cwd,
  this.sessionManager,
  new ModelRegistry(this._modelRuntime),
);
this._bindExtensionCore(this._extensionRunner);      // 注入宿主核心能力: 让插件的 ctx 能反向操作会话(发消息/改工具集/切模型/压缩/读上下文用量等)
this._applyExtensionBindings(this._extensionRunner);  // 绑定 UI 与外部回调: UI 上下文、命令上下文、错误监听器
```

`_resourceLoader.getExtensions()` 底层用 `loader.ts` 扫描加载，产物交给 `new ExtensionRunner(...)` 托管。`this._extensionRunner` 从此就是插件系统的中枢，后面所有 `emit` 都打给它。

## 骨架二：底层 loop 的回调 → 插件事件（最关键的桥）

`agent.ts` 只认 `beforeToolCall` / `afterToolCall` 两个回调，`AgentSession` 在 `_installAgentToolHooks` 里把它们**转发**成插件事件：

```ts
private _installAgentToolHooks(): void {
  this.agent.beforeToolCall = async ({ toolCall, args }) => {
    const runner = this._extensionRunner;
    if (!runner.hasHandlers("tool_call")) {
      return undefined;                    // 没人订阅，直接跳过
    }
    return await runner.emitToolCall({     // 转成插件事件
      type: "tool_call",
      toolName: toolCall.name,
      toolCallId: toolCall.id,
      input: args as Record<string, unknown>,
    });
  };

  this.agent.afterToolCall = async ({ toolCall, args, result, isError }) => {
    // 同理转成 tool_result 事件，插件还能改写结果
  };
}
```

这就是整条链的接缝：**模型要调工具 →** `agent.ts` **触发** `beforeToolCall` **→** `AgentSession` **转成** `emitToolCall` **→ 插件的** `pi.on("tool_call")` **拿到，可** `block`**。** `agent.ts` 全程不知道「插件」是什么，它只是在合适的时机回调了宿主注入的函数。

## 骨架三：生命周期事件的广播

除了工具钩子，`AgentSession` 在各个关键时刻散布着几十处 `this._extensionRunner.emit(...)`：`agent_start` / `agent_end`、`message_end`、`session_before_compact`……

你不用逐个记，模式统一：**宿主在某件事发生时** `emit` **一个事件，订阅了的插件就被回调。** 这正是「事件驱动」架构的样子——宿主在自己流程里埋下一堆「广播点」，插件按需订阅。

# 串起来看

用一次危险命令拦截，把整条链走一遍：

```
① 加载期
   loader 扫描 .pi/extensions/*.ts
   → factory(api) 执行插件, pi.on("tool_call", h) 把 h 存进 extension.handlers
   → ExtensionRunner 托管这个 extension

② 运行期
   模型返回一个 bash 的 toolCall (rm -rf ...)
   → agent-loop 执行前触发 agent.beforeToolCall
   → AgentSession._installAgentToolHooks 转成 runner.emitToolCall
   → runner 遍历插件, 调到我们的 handler
   → handler 弹 ctx.ui.confirm, 用户拒绝 → 返回 { block: true }
   → emitToolCall 短路返回 block
   → agent-loop 拿到 block, 这个工具不执行, 回一个错误结果给模型
```

一句话总结整个系统的分工：

- `loader.ts`：把插件从磁盘读进来，`factory(pi)` 执行后产出带 `handlers` / `tools` 两个 Map 的 `Extension`。
- `runner.ts`**（ExtensionRunner）**：托管所有插件，`emit` / `emitToolCall` 负责在正确的时刻把事件分发下去，还负责错误隔离。
- `types.ts`：定义 `ExtensionAPI`（插件作者用的 `pi`）、`ExtensionContext`（回调里的 `ctx`）和所有事件类型——即「插件能碰哪些东西、能听哪些时刻」。
- `AgentSession`：宿主。创建 runner、把底层 `agent.ts` 的回调转成插件事件、在几十个关键点广播生命周期事件。



# 写在最后

tool 是给模型用的，插件是给 pi 自己用的。区别就一点：工具被动等模型调用，插件能主动卡在流程的关键节点上，决定这件事要不要发生、结果要不要改。

而它的底子其实很朴素，就是一个事件总线：宿主在流程里埋广播点，插件用 `on` 订阅、用返回值介入，`ExtensionRunner` 居中调度、顺便兜住每个插件的异常。真正值得学的是三层分工——`agent.ts` 只管跑、`AgentSession` 负责翻译和广播、`ExtensionRunner` 负责托管和隔离。分层清楚了，「让第三方代码安全地嵌进核心流程」这件危险事才变得可控。

在写了几篇文章后，ai已经能基于现有数据很好的模仿了，感觉写的很不错，流程非常的清楚。
而且ai会自动截掉无关的代码，关注主要流程，让文章更加清晰易懂。