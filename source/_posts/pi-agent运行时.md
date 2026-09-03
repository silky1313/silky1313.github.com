---
title: PI分析-agent 运行时
date: 2026-9-1 23:07:00
categories: 
- agent
---

# AgentSessionRuntime
我们先从代码开始看看，后续再总结整个模块的功能
```ts
export class AgentSessionRuntime {
	// 提供了两个钩子
	// 一个用于重新绑Session
	// 一个用于在Session失效前执行清理
	private rebindSession?: (session: AgentSession) => Promise<void>;
	private beforeSessionInvalidate?: () => void;
	// 管理两个核心组件，Session和Services
	private _session: AgentSession;
	private _services: AgentSessionServices;
	// 创建runtime的工厂函数
	private readonly createRuntime: CreateAgentSessionRuntimeFactory;
	// 一些错误信息的收集，用于错误处理
	private _diagnostics: AgentSessionRuntimeDiagnostic[];
	private _modelFallbackMessage?: string;
}
```
管理的主体就是AgentSession和AgentSessionServices，然后提供了一些钩子
四个核心方法
- switchSession
- newSession
- fork
- importFromJsonl

## switchSession
```ts
async switchSession(
    sessionPath: string,
    options?: {
        cwdOverride?: string;
        withSession?: (ctx: ReplacedSessionContext) => Promise<void>;
        projectTrustContextFactory?: (cwd: string) => ProjectTrustContext;
    },
): Promise<{ cancelled: boolean }> {
    // 发送事件给插件
    const beforeResult = await this.emitBeforeSwitch("resume", sessionPath);
    if (beforeResult.cancelled) {
        return beforeResult;
    }

    // sessionFile是原会话path， sessionPath是目标会话path
    const previousSessionFile = this.session.sessionFile;
    const sessionManager = SessionManager.open(sessionPath, undefined, options?.cwdOverride);
    // check session的cwd是否存在
    assertSessionCwdExists(sessionManager, this.cwd);
    // 处理当前正在执行的活动，abort他们
    await this.teardownCurrent("resume", sessionManager.getSessionFile()); 
    // 调用工厂函数更新各个属性
    this.apply(
        await this.createRuntime({
            // 新的cwd
            cwd: sessionManager.getCwd(),
            agentDir: this.services.agentDir,
            // 新的sessionManager
            sessionManager,
            sessionStartEvent: { type: "session_start", reason: "resume", previousSessionFile },
            projectTrustContext: options?.projectTrustContextFactory?.(sessionManager.getCwd()),
        }),
    ); 
    // 调用外部钩子，一个是rebindSession，一个是如果option有withSession也要调用
    await this.finishSessionReplacement(options?.withSession);
    return { cancelled: false };
}

private async teardownCurrent(reason: SessionShutdownEvent["reason"], targetSessionFile?: string): Promise<void> {
    // Settle any active response first so the aborted turn (including tool
    // results) is persisted to the outgoing session before it is replaced.
    // 中断当前session
    await this.session.abort(); 
    // 插件处理session_shutdown事件
    await emitSessionShutdownEvent(this.session.extensionRunner, {
        type: "session_shutdown",
        reason,
        targetSessionFile,
    });
    // 外部钩子，在session有效前的一些操作，比如UI清理啥的，具体可以看看方法的注释
    this.beforeSessionInvalidate?.();
    // session的一些清理操作，abort正在执行的操作等等
    this.session.dispose(); 
}
```

这里看看工厂函数和apply函数具体干了啥
```ts
export type CreateAgentSessionRuntimeFactory = (options: {
	cwd: string;
	agentDir: string;
	sessionManager: SessionManager;
	sessionStartEvent?: SessionStartEvent;
	projectTrustContext?: ProjectTrustContext;
}) => Promise<CreateAgentSessionRuntimeResult>;

/** Result from createAgentSession */
export interface CreateAgentSessionResult {
	/** The created session */
	session: AgentSession;
	/** Extensions result (for UI context setup in interactive mode) */
	extensionsResult: LoadExtensionsResult;
	/** Warning if session was restored with a different model than saved */
	modelFallbackMessage?: string;
}

// 两个返回值都有了modelFallbackMessage和diagnostics
export interface CreateAgentSessionRuntimeResult extends CreateAgentSessionResult {
	services: AgentSessionServices;
	diagnostics: AgentSessionRuntimeDiagnostic[];
}

// 这个再把重要信息重新赋值回去
private apply(result: CreateAgentSessionRuntimeResult): void {
    this._session = result.session;
    this._services = result.services;
    this._diagnostics = result.diagnostics;
    this._modelFallbackMessage = result.modelFallbackMessage;
}
```

## newSession
再看这个逻辑旧非常清晰了
```ts
async newSession(options?: {
    parentSession?: string;
    setup?: (sessionManager: SessionManager) => Promise<void>;
    withSession?: (ctx: ReplacedSessionContext) => Promise<void>;
}): Promise<{ cancelled: boolean }> {
    // 插件处理事件
    const beforeResult = await this.emitBeforeSwitch("new");
    if (beforeResult.cancelled) {
        return beforeResult;
    }

    // 创建新session
    const previousSessionFile = this.session.sessionFile;
    const sessionDir = this.session.sessionManager.getSessionDir();
    const sessionManager = this.session.sessionManager.isPersisted()
        ? SessionManager.create(this.cwd, sessionDir)
        : SessionManager.inMemory(this.cwd);
    if (options?.parentSession) {
        sessionManager.newSession({ parentSession: options.parentSession });
    }

    // 与switchSession相同的操作
    await this.teardownCurrent("new", sessionManager.getSessionFile());
    this.apply(
        await this.createRuntime({
            cwd: this.cwd,
            agentDir: this.services.agentDir,
            sessionManager,
            sessionStartEvent: { type: "session_start", reason: "new", previousSessionFile },
        }),
    );
    // 提供了一个外部钩子，看起来是构造了一些初始的message
    if (options?.setup) {
        await options.setup(this.session.sessionManager);
        this.session.agent.state.messages = this.session.sessionManager.buildSessionContext().messages;
    }
    // 调用外部钩子，一个是rebindSession，一个是如果option有withSession也要调用
    await this.finishSessionReplacement(options?.withSession);
    return { cancelled: false };
}
```

## fork
```ts
async fork(
    entryId: string, // 之前已经讲过pi的session是树形的，这个节点id
    options?: { position?: "before" | "at"; withSession?: (ctx: ReplacedSessionContext) => Promise<void> }, // 一个fork位置，一个seesion构造完成后的钩子
): Promise<{ cancelled: boolean; selectedText?: string }> {
    const position = options?.position ?? "before";
    // 拓展处理事件
    const beforeResult = await this.emitBeforeFork(entryId, { position });
    if (beforeResult.cancelled) {
        return { cancelled: true };
    }
    let targetLeafId: string | null;
    let selectedText: string | undefined;

    const selectedEntry = this.session.sessionManager.getEntry(entryId);
    if (!selectedEntry) {
        throw new Error("Invalid entry ID for forking");
    }

    if (position === "at") {
        targetLeafId = selectedEntry.id;
    } else {
        // 如果是before，会要求从上一个turn的结尾开始，避免从turn的中间分叉
        if (selectedEntry.type !== "message" || selectedEntry.message.role !== "user") {
            throw new Error("Invalid entry ID for forking");
        }
        targetLeafId = selectedEntry.parentId;
        // 获取当前selectedEntry的一个msg，可能是为了供用户重新编辑
        selectedText = extractUserMessageText(selectedEntry.message.content);
    }

    const previousSessionFile = this.session.sessionFile;
    // 如果session需要持久话的话
    if (this.session.sessionManager.isPersisted()) {
        const currentSessionFile = this.session.sessionFile;
        if (!currentSessionFile) {
            throw new Error("Persisted session is missing a session file");
        }
        const sessionDir = this.session.sessionManager.getSessionDir();
        // 代表在一个root节点执行fork操作
        if (!targetLeafId) {
            const sessionManager = SessionManager.create(this.cwd, sessionDir);
            sessionManager.newSession({ parentSession: currentSessionFile });
            await this.teardownCurrent("fork", sessionManager.getSessionFile());
            this.apply(
                await this.createRuntime({
                    cwd: this.cwd,
                    agentDir: this.services.agentDir,
                    sessionManager,
                    sessionStartEvent: { type: "session_start", reason: "fork", previousSessionFile },
                }),
            );
            await this.finishSessionReplacement(options?.withSession);
            return { cancelled: false, selectedText };
        }

        //  非root节点，说明session已经存在了
        if (!existsSync(currentSessionFile)) {
            throw new Error(
                "This session has not been saved yet. Wait for the first assistant response before cloning or forking it.",
            );
        }
        // 创建fork session
        const sessionManager = SessionManager.open(currentSessionFile, sessionDir);
        const forkedSessionPath = sessionManager.createBranchedSession(targetLeafId);
        if (!forkedSessionPath) {
            throw new Error("Failed to create forked session");
        }
        await this.teardownCurrent("fork", sessionManager.getSessionFile());
        this.apply(
            await this.createRuntime({
                cwd: sessionManager.getCwd(),
                agentDir: this.services.agentDir,
                sessionManager,
                sessionStartEvent: { type: "session_start", reason: "fork", previousSessionFile },
            }),
        );
        await this.finishSessionReplacement(options?.withSession);
        return { cancelled: false, selectedText };
    }

    // session不需要持久化了，缺失了create session 和 open session文件的操作
    const sessionManager = this.session.sessionManager;
    if (!targetLeafId) {
        sessionManager.newSession({ parentSession: this.session.sessionFile });
    } else {
        sessionManager.createBranchedSession(targetLeafId);
    }
    await this.teardownCurrent("fork", sessionManager.getSessionFile());
    this.apply(
        await this.createRuntime({
            cwd: this.cwd,
            agentDir: this.services.agentDir,
            sessionManager,
            sessionStartEvent: { type: "session_start", reason: "fork", previousSessionFile },
        }),
    );
    await this.finishSessionReplacement(options?.withSession);
    return { cancelled: false, selectedText };
}
```

## importFromJsonl
这个应该就是从文件中恢复session，不再赘述
```ts
/**
* Import a session JSONL file and switch runtime state to the imported session.
*
* @returns `{ cancelled: true }` when cancelled by `session_before_switch`, otherwise `{ cancelled: false }`.
* @throws {SessionImportFileNotFoundError} When the input path does not exist.
* @throws {MissingSessionCwdError} When the imported session cwd cannot be resolved and no override is provided.
*/
async importFromJsonl(inputPath: string, cwdOverride?: string): Promise<{ cancelled: boolean }> {
    const resolvedPath = resolvePath(inputPath);
    if (!existsSync(resolvedPath)) {
        throw new SessionImportFileNotFoundError(resolvedPath);
    }

    const sessionDir = this.session.sessionManager.getSessionDir();
    if (!existsSync(sessionDir)) {
        mkdirSync(sessionDir, { recursive: true });
    }

    const destinationPath = join(sessionDir, basename(resolvedPath));
    const beforeResult = await this.emitBeforeSwitch("resume", destinationPath);
    if (beforeResult.cancelled) {
        return beforeResult;
    }

    const previousSessionFile = this.session.sessionFile;
    if (resolve(destinationPath) !== resolvedPath) {
        copyFileSync(resolvedPath, destinationPath);
    }

    const sessionManager = SessionManager.open(destinationPath, sessionDir, cwdOverride);
    assertSessionCwdExists(sessionManager, this.cwd);
    await this.teardownCurrent("resume", sessionManager.getSessionFile());
    this.apply(
        await this.createRuntime({
            cwd: sessionManager.getCwd(),
            agentDir: this.services.agentDir,
            sessionManager,
            sessionStartEvent: { type: "session_start", reason: "resume", previousSessionFile },
        }),
    );
    await this.finishSessionReplacement();
    return { cancelled: false };
}
```
整体而言，`AgentSessionRuntime` 扮演的是「会话生命周期的稳定门面」这一角色。它对外暴露一个恒定不变的句柄，把「会话与工作目录会随时被替换」这件事彻底封装在内部：无论是切换、新建、fork 还是从 JSONL 导入，上层拿到的始终是同一个 `runtime` 引用，而其背后的 `AgentSessionServices`（目录绑定的基础设施）与 `AgentSession`（会话主体）则被原子地整体替换。

这种设计的价值在于**引用稳定性与状态可变性的解耦**：调用方无需关心底层对象何时被重建，也不必在每次切换后重新持有引用、重新接线；同时通过 `rebindSession` / `beforeSessionInvalidate` 以及一系列 `session_before_switch`、`session_shutdown` 扩展事件，把「旧会话如何优雅退场、新会话如何平滑接管」这一过程完全暴露出来，让 UI 层与扩展能在恰当的时机介入清理与重绑。可以说，它用一层轻薄的间接，换来了整个上层交互对会话切换的无感知。

这里使用最近热门的archify生成了一个架构图，虽然文章本身线条已经很清晰了，但是架构图有利于全局理解
![](images/pi-agent运行时-架构.png)

# AgentSessionServices
```ts
/**
 * Coherent cwd-bound runtime services for one effective session cwd.
 *
 * This is infrastructure only. The AgentSession itself is created separately so
 * session options can be resolved against these services first.
 */
export interface AgentSessionServices {
	cwd: string; // 当前有效工作目录（这套服务绑定的目录）
	agentDir: string; // agent 自身目录（配置、扩展、skill 等）
	modelRuntime: ModelRuntime; // 模型运行时：模型提供方、选择、鉴权
	settingsManager: SettingsManager; // 设置管理器：读写用户配置
	resourceLoader: ResourceLoader; // 资源加载器：主题、扩展资源
	diagnostics: AgentSessionRuntimeDiagnostic[]; // 创建服务时产生的诊断信息
}
```
主要是提供session与目录的绑定，便于切换session

# AgentSession
这则是一个超级大类了，主要提供会话功能。先看看它持有的核心字段，按功能分组如下：

| 分组 | 关键字段 | 作用 |
|------|---------|------|
| 核心依赖 | `agent` / `sessionManager` / `settingsManager` / `_modelRuntime` | 底层 agent、会话持久化、设置、模型鉴权 |
| 事件订阅 | `_unsubscribeAgent` / `_eventListeners` / `_isAgentRunActive` | 订阅 agent 事件并转发给模式层 |
| 消息队列 | `_steeringMessages` / `_followUpMessages` / `_pendingNextTurnMessages` | 流式中插队(steer)、排队(followUp)、下一轮上下文 |
| 压缩 | `_compactionAbortController` / `_autoCompactionAbortController` / `_overflowRecoveryAttempted` | 手动 + 自动 compaction，上下文溢出恢复 |
| 重试 | `_retryAbortController` / `_retryAttempt` | 助手错误后的自动重试 |
| Bash | `_bashAbortControllers` / `_pendingBashMessages` | 独立于 LLM 的 bash 执行 |
| 扩展/工具 | `_extensionRunner` / `_toolRegistry` / `_toolDefinitions` / `_baseSystemPrompt` | 扩展系统、动态工具注册表、系统提示词构建 |

总体而言就是这些模块构成，我们再看看它对外提供了什么方法。我让ai总结了一些对外提供的方法，总体分为4个部分

| 分组 | 方法名 | 作用 |
|------|--------|------|
| 会话交互 | `prompt` / `steer` / `followUp` / `sendUserMessage` / `sendCustomMessage` / `clearQueue` / `getSteeringMessages` / `getFollowUpMessages` / `setSteeringMode` / `setFollowUpMode` / `abort` / `waitForIdle` | 输入驱动、插队/排队及其模式、中断与等待空闲 |
| 模型与上下文 | `setModel` / `cycleModel` / `setScopedModels` / `setThinkingLevel` / `cycleThinkingLevel` / `getAvailableThinkingLevels` / `supportsThinking` / `compact` / `abortCompaction` / `abortBranchSummary` / `setAutoCompactionEnabled` / `abortRetry` / `setAutoRetryEnabled` | 模型切换、思考等级、压缩与重试 |
| 工具与扩展 | `getActiveToolNames` / `getAllTools` / `getToolDefinition` / `setActiveToolsByName` / `subscribe` / `bindExtensions` / `reload` / `hasExtensionHandlers` / `executeBash` / `recordBashResult` / `abortBash` | 工具查询与启用、扩展绑定与事件、bash 执行 |
| 会话管理 | `setSessionName` / `navigateTree` / `getUserMessagesForForking` / `getSessionStats` / `getContextUsage` / `getLastAssistantText` / `exportToHtml` / `exportToJsonl` / `dispose` / `createReplacedSessionContext` | 命名、树形分支/fork、统计与导出、资源释放 |

也就是4个核心部分，会话交互这个部分再之前的agent内核中涉及到一些，这里再具体看看

## 会话交互

### prompt
这里的核心入口是prompt方法
```ts
async prompt(text: string, options?: PromptOptions): Promise<void> {
    const expandPromptTemplates = options?.expandPromptTemplates ?? true;
    const preflightResult = options?.preflightResult;
    let messages: AgentMessage[] | undefined;

    try {
        // /xxx args 看是否有插件需要处理
        if (expandPromptTemplates && text.startsWith("/")) {
            const handled = await this._tryExecuteExtensionCommand(text);
            if (handled) {
                // Extension command executed, no prompt to send
                preflightResult?.(true);
                return;
            }
        }

        // 插件处理 input
        let currentText = text;
        let currentImages = options?.images;
        if (this._extensionRunner.hasHandlers("input")) {
            const inputResult = await this._extensionRunner.emitInput(
                currentText,
                currentImages,
                options?.source ?? "interactive",
                this.isStreaming ? options?.streamingBehavior : undefined,
            );
            if (inputResult.action === "handled") {
                preflightResult?.(true);
                return;
            }
            if (inputResult.action === "transform") {
                currentText = inputResult.text;
                currentImages = inputResult.images ?? currentImages;
            }
        }

        // 展开skill 和 prompt template
        let expandedText = currentText;
        if (expandPromptTemplates) {
            expandedText = this._expandSkillCommand(expandedText);
            expandedText = expandPromptTemplate(expandedText, [...this.promptTemplates]);
        }

        // 判断当前agent是否正在运行 
        if (this.isStreaming) {
            if (!options?.streamingBehavior) {
                throw new Error(
                    "Agent is already processing. Specify streamingBehavior ('steer' or 'followUp') to queue the message.",
                );
            }
            // 两种队列，steer是turn中间即时插入，followUp是turn结束再插入
            if (options.streamingBehavior === "followUp") {
                await this._queueFollowUp(expandedText, currentImages);
            } else {
                await this._queueSteer(expandedText, currentImages);
            }
            preflightResult?.(true);
            return;
        }

        // 把bash队列刷到message中
        this._flushPendingBashMessages();

        // Validate model
        if (!this.model) {
            throw new Error(formatNoModelSelectedMessage());
        }

        // 鉴权
        const hasConfiguredAuth =
            this._modelRuntime.hasConfiguredAuth(this.model.provider) ||
            (await this._modelRuntime.checkAuth(this.model.provider)) !== undefined;
        if (!hasConfiguredAuth) {
            const isOAuth = this._modelRuntime.isUsingOAuth(this.model.provider);
            if (isOAuth) {
                throw new Error(
                    `Authentication failed for "${this.model.provider}". ` +
                        `Credentials may have expired or network is unavailable. ` +
                        `Run '/login ${this.model.provider}' to re-authenticate.`,
                );
            }
            throw new Error(formatNoApiKeyFoundMessage(this.model.provider));
        }

        // 发送用户消息前先check是否需要压缩
        const lastAssistant = this._findLastAssistantMessage();
        if (lastAssistant) {
            await this._checkCompaction(lastAssistant, false);
        }

        // Build messages array (custom message if any, then user message)
        messages = [];

        // 添加用户message
        const userContent: (TextContent | ImageContent)[] = [{ type: "text", text: expandedText }];
        if (currentImages) {
            userContent.push(...currentImages);
        }
        messages.push({
            role: "user",
            content: userContent,
            timestamp: Date.now(),
        });

        // 添加nextTurn消息
        for (const msg of this._pendingNextTurnMessages) {
            messages.push(msg);
        }
        this._pendingNextTurnMessages = [];

        // 发送拓展消息看是否需要修改系统提示词
        const result = await this._extensionRunner.emitBeforeAgentStart(
            expandedText,
            currentImages,
            this._baseSystemPrompt,
            this._baseSystemPromptOptions,
        );
        if (result?.messages) {
            for (const msg of result.messages) {
                messages.push({
                    role: "custom",
                    customType: msg.customType,
                    // Untyped extensions can pass null/missing content; normalize at ingestion.
                    content: msg.content ?? [],
                    display: msg.display,
                    details: msg.details,
                    timestamp: Date.now(),
                });
            }
        }
        // Apply extension-modified system prompt, or reset to base
        if (result?.systemPrompt !== undefined) {
            this._systemPromptOverride = result.systemPrompt;
            this.agent.state.systemPrompt = result.systemPrompt;
        } else {
            // Ensure we're using the base prompt (in case previous turn had modifications)
            this._systemPromptOverride = undefined;
            this.agent.state.systemPrompt = this._baseSystemPrompt;
        }
    } catch (error) {
        preflightResult?.(false);
        throw error;
    }

    if (!messages) {
        return;
    }

    preflightResult?.(true);
    await this._runAgentPrompt(messages);
}
```
这里再调用了`await this._runAgentPrompt(messages)`去处理发送message的流程
```ts
private async _runAgentPrompt(messages: AgentMessage | AgentMessage[]): Promise<void> {
    // 标识当前agent正在运行
    this._isAgentRunActive = true;
    try {
        // 调用agent核的prompt的方法
        await this.agent.prompt(messages);
        // 判断是否需要continue
        while (await this._handlePostAgentRun()) {
            await this.agent.continue();
        }
    } finally {
        // 系统提示词被覆盖这里会设置为true，这里重置他
        this._systemPromptOverride = undefined;
        this._flushPendingBashMessages();
        // 
        await this._emitAgentSettled();
    }
}
```

这里我们具体看看是怎么判断是否需要continue的
```ts
private async _handlePostAgentRun(): Promise<boolean> {
    const msg = this._lastAssistantMessage;
    this._lastAssistantMessage = undefined;
    if (!msg) {
        return false;
    }
    
    // 错误可重试才会去准备重试，_prepareRetry是比较具体的细节了，我们这里不在赘述，可以进方法看看
    if (this._isRetryableError(msg) && (await this._prepareRetry(msg))) {
        return true;
    }

    // 到这里可以判断重试已经结束了，因为最后一条消息不是错误消息了
    if (msg.stopReason === "error" && this._retryAttempt > 0) {
        this._emit({
            type: "auto_retry_end",
            success: false,
            attempt: this._retryAttempt,
            finalError: msg.errorMessage,
        });
        this._retryAttempt = 0;
    }

    // 压缩后是否需要continue？因为有些是因为上下文溢出，所以压缩后需要重试
    if (await this._checkCompaction(msg)) {
        return true;
    }

    // 判断是否还有排队的消息
    return this.agent.hasQueuedMessages();
}
```

这里还有一个收尾工作，主要是设置非活跃和空闲状态
```ts
private async _emitAgentSettled(): Promise<void> {
    // agent已不在活跃
    this._isAgentRunActive = false;
    try {
        await this._extensionRunner.emit({ type: "agent_settled" });
        this._emit({ type: "agent_settled" });
    } finally {
        this._resolveIdleWaitIfIdle();
    }
}

// 这个方法是调用了一个_resolveIdleWait这个resolve，来自于_idleWaitPromise
// 表示当前agent是否处于空闲状态
private _resolveIdleWaitIfIdle(): void {
    if (this._isAgentRunActive || !this._resolveIdleWait) {
        return;
    }
    const resolve = this._resolveIdleWait;
    this._idleWaitPromise = undefined;
    this._resolveIdleWait = undefined;
    resolve();
}

private _getIdleWaitPromise(): Promise<void> {
    if (!this._idleWaitPromise) {
        this._idleWaitPromise = new Promise((resolve) => {
            this._resolveIdleWait = resolve;
        });
    }
    return this._idleWaitPromise;
}

async waitForIdle(): Promise<void> {
    if (this.isIdle) {
        return;
    }
    await this._getIdleWaitPromise();
}
```

### steer 和 followUp 的区别
- steer在turn中间能插入就插入
- followup则是在一个turn结束再插入

## 模型与上下文
这一块比较重要的就是上下文压缩，具体可以看 {% post_link pi-上下文压缩 %}

## 工具与拓展
这里应该与主流程耦合很轻，算是独立模块，所以准备后续开个新文章写写

## 会话管理
这块其实主要就是SessionManager在管理，AgentSession也就是组合SessionManager的各个方法
这块其实并不算核心，其实本质就是将对话组织成树形形式，从而支持各种操作
