---
title: PI分析-agent 内核
date: 2026-8-27 23:07:00
categories: 
- agent
---

我们都知道agent本质本就是一个loop，不断循环思考去解决下一步问题，经典的就是ReAct模式

这里我们重点看看pi agent内核是咋写的，依旧以prompt为入口开始看看

## prompt

```ts
/** Start a new prompt from text, a single message, or a batch of messages. */
async prompt(message: AgentMessage | AgentMessage[]): Promise<void>;
async prompt(input: string, images?: ImageContent[]): Promise<void>;
```
我们还需分别看看AgentMessage和ImageContent是什么结构
```ts
// CustomAgentMessages是拓展自定义Message类型，CustomAgentMessages[keyof CustomAgentMessages]这个语法是拆开所有类型，Message | AMessage | BMessage....
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
export type Message = UserMessage | AssistantMessage | ToolResultMessage;

// UserMessage
export interface UserMessage {
	role: "user";
	content: string | (TextContent | ImageContent)[];
	timestamp: number; // Unix timestamp in milliseconds
}
export interface TextContent {
	type: "text";
	text: string;
	textSignature?: string; // e.g., for OpenAI responses, message metadata (legacy id string or TextSignatureV1 JSON)
}
export interface ImageContent {
	type: "image";
	data: string; // base64 encoded image data
	mimeType: string; // e.g., "image/jpeg", "image/png"
}

// AssistantMessage
export interface AssistantMessage {
	role: "assistant";
	content: (TextContent | ThinkingContent | ToolCall)[];
	api: Api; // 协议类型
	provider: ProviderId; // 模型厂商
	model: string; //  请求时model
	responseModel?: string;// Concrete `chunk.model` when different from the requested `model` (e.g. OpenRouter `auto` -> `anthropic/...`)
	responseId?: string; // Provider-specific response/message identifier when the upstream API exposes one
	diagnostics?: AssistantMessageDiagnostic[]; // Redacted provider/runtime diagnostics for failures and recoveries.
	usage: Usage; // 用量，整体还是很好理解的，点进去看看就知道了
	stopReason: StopReason; // 停止原因
	deferred?: DeferredHandle; // 异步请求句柄，提示agent稍后再查询进度
	errorMessage?: string; 
	rawStopReason?: string; // provider 原始停止原因
	/**
	 * Provider indication of whether the model explicitly ended its turn.
	 * Preserved for debugging and does not currently affect agent control flow.
	 */
	endTurn?: boolean;
	timestamp: number; // Unix timestamp in milliseconds
}
export interface ThinkingContent {
	type: "thinking";
	thinking: string;
	thinkingSignature?: string; // e.g., for OpenAI responses, the reasoning item ID
	/** When true, the thinking content was redacted by safety filters. The opaque
	 *  encrypted payload is stored in `thinkingSignature` so it can be passed back
	 *  to the API for multi-turn continuity. */
	redacted?: boolean;
}
export interface ToolCall {
	type: "toolCall";
	id: string;
	name: string;
	arguments: Record<string, any>; // 一个object，本质可以理解为一个string -> any的映射
	thoughtSignature?: string; // Google-specific: opaque signature for reusing thought context
	/** OpenAI Responses namespace for calls to dynamically loaded or namespaced tools. */
	namespace?: string;
}
```

解析来重点看看run的过程, 上面两个prompt是重载函数，真真调用的是下面这个
```ts
async prompt(input: string | AgentMessage | AgentMessage[], images?: ImageContent[]): Promise<void> {
    // 
    if (this.activeRun) {
        throw new Error(
            "Agent is already processing a prompt. Use steer() or followUp() to queue messages, or wait for completion.",
        );
    }
    // 全部转为AgentMessage[]
    const messages = this.normalizePromptInput(input, images);
    await this.runPromptMessages(messages);
}
private normalizePromptInput(
		input: string | AgentMessage | AgentMessage[],
		images?: ImageContent[],
	): AgentMessage[] {
        // 转化为AgentMessage[]
		if (Array.isArray(input)) {
			return input;
		}
		if (typeof input !== "string") {
			return [input];
		}

        // 对应第二个重载方法string + ImageContent[]
		const content: Array<TextContent | ImageContent> = [{ type: "text", text: input }];
		if (images && images.length > 0) {
			content.push(...images);
		}
		return [{ role: "user", content, timestamp: Date.now() }];
	}
}
```

### activeRun
这个activeRun是一个好设计，单独拿出来看看怎么个事
整体还是基于把promise的resolve获取出来，完成了如下功能
- 并发控制(run只运行一个人跑)
- 等待（waitForIdle）
- 中断，通过AbortController控制
```ts
type ActiveRun = {
	promise: Promise<void>;
	resolve: () => void;
	abortController: AbortController;
};

class Agent {
	private activeRun?: ActiveRun;

	async run() {
		if (this.activeRun) {
			throw new Error("already running");
		}

		const abortController = new AbortController();
		let resolve = () => {};
		const promise = new Promise<void>((r) => (resolve = r));
		this.activeRun = { promise, resolve, abortController };

		try {
			await this.task(abortController.signal);
			console.log("done");
		} catch (e) {
			console.log("caught:", (e as Error).message);
		} finally {
			this.activeRun.resolve();
			this.activeRun = undefined;
		}
	}

	private task(signal: AbortSignal) {
		return new Promise<void>((resolve, reject) => {
            // 如果接受到中断信息，reject退出
			const tick = setInterval(() => {
				if (signal.aborted) {
					clearInterval(tick);
					reject(new Error("aborted"));
					return;
				}
				console.log("working...");
			}, 500);

			setTimeout(() => {
				clearInterval(tick);
				resolve();
			}, 3000);
		});
	}

	abort() {
		this.activeRun?.abortController.abort();
	}

	async waitForIdle() {
        // 等待this.activeRun?.promise完成
		await (this.activeRun?.promise ?? Promise.resolve());
	}
}

async function main() {
	const agent = new Agent();

	const run = agent.run();

    // 因为正在run，所以这里应该rejected: already running
	try {
		await agent.run();
	} catch (e) {
		console.log("rejected:", (e as Error).message);
	}

    // 沉睡1200，应该输出两次working...
    // 然后因为调用了abort，console.log("caught:", (e as Error).message);会catch这个error，caught: aborted
	setTimeout(() => agent.abort(), 1200);
	await run;

    // 等待run完成，所以这里应该输出idle
	await agent.waitForIdle();
	console.log("idle");
}

main();

// output
// rejected: already running
// working...
// working...
// caught: aborted
// idle
```

继续看主流程
```ts
private async runPromptMessages(
    messages: AgentMessage[],
    options: { skipInitialSteeringPoll?: boolean } = {},
): Promise<void> {
    await this.runWithLifecycle(async (signal) => {
        await runAgentLoop(
            messages,
            this.createContextSnapshot(),
            this.createLoopConfig(options),
            (event) => this.processEvents(event),
            signal,
            this.streamFunction,
        );
    });
}

// 有上面的activeRun demo，这个就非常清晰了
private async runWithLifecycle(executor: (signal: AbortSignal) => Promise<void>): Promise<void> {
    if (this.activeRun) {
        throw new Error("Agent is already processing.");
    }

    const abortController = new AbortController();
    let resolvePromise = () => {};
    const promise = new Promise<void>((resolve) => {
        resolvePromise = resolve;
    });
    this.activeRun = { promise, resolve: resolvePromise, abortController };

    this._state.isStreaming = true;
    this._state.streamingMessage = undefined;
    this._state.errorMessage = undefined;

    try {
        await executor(abortController.signal);
    } catch (error) {
        await this.handleRunFailure(error, abortController.signal.aborted);
    } finally {
        this.finishRun();
    }
}
```

## runAgentLoop
这个主要流程进入了agent-loop.ts
```ts
export async function runAgentLoop(
	prompts: AgentMessage[],
	context: AgentContext, // 包括message，tools，system prompt
	config: AgentLoopConfig,
	emit: AgentEventSink,
	signal: AbortSignal | undefined,
	streamFn: StreamFn,
): Promise<AgentMessage[]> {
	const newMessages: AgentMessage[] = [...prompts];
	const currentContext: AgentContext = {
		...context,
		messages: [...context.messages, ...prompts],
	};

    // 由Agent传递进来emit方法，emit通知Agent当前发生事件
	await emit({ type: "agent_start" });
	await emit({ type: "turn_start" });
	for (const prompt of prompts) {
		await emit({ type: "message_start", message: prompt });
		await emit({ type: "message_end", message: prompt });
	}

	await runLoop(currentContext, newMessages, config, signal, emit, streamFn ?? getDefaultStreamFn());
	return newMessages;
}

async function runLoop(
	initialContext: AgentContext, // 这是原始的context
	newMessages: AgentMessage[], // 这是新要注入的message 
	initialConfig: AgentLoopConfig, 
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
	streamFunction: StreamFn,
): Promise<void> {
	let currentContext = initialContext;
	let config = initialConfig;
	let firstTurn = true;
	// steer message是高优先级message， turn内立即传入进去
	let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];

	// Outer loop: continues when queued follow-up messages arrive after agent would stop
	while (true) {
		let hasMoreToolCalls = true;

		// Inner loop: process tool calls and steering messages
		while (hasMoreToolCalls || pendingMessages.length > 0) {
            // 这里因为外面已经发送过一次turn_start，所以这里不再发送
			if (!firstTurn) {
				await emit({ type: "turn_start" });
			} else {
				firstTurn = false;
			}

			// Process pending messages (inject before next assistant response)
			if (pendingMessages.length > 0) {
				for (const message of pendingMessages) {
					await emit({ type: "message_start", message });
					await emit({ type: "message_end", message });
					currentContext.messages.push(message);
					newMessages.push(message);
				}
				pendingMessages = [];
			}

			// Stream assistant response
			const message = await streamAssistantResponse(currentContext, config, signal, emit, streamFunction);
			newMessages.push(message);

			if (message.stopReason === "error" || message.stopReason === "aborted") {
				await emit({ type: "turn_end", message, toolResults: [] });
				await emit({ type: "agent_end", messages: newMessages });
				return;
			}

			// 处理tool call
			const toolCalls = message.content.filter((c) => c.type === "toolCall");

			const toolResults: ToolResultMessage[] = [];
			hasMoreToolCalls = false;
			if (toolCalls.length > 0) {
				// A "length" stop means the output was cut off by the token limit, so
				// every tool call in the message may carry truncated arguments. Fail
				// them all instead of executing potentially borked calls.
                // 注释已经很清楚了，对于被截断的llm output可能参数不全，agent直接返回err更合理
				const executedToolBatch =
					message.stopReason === "length"
						? await failToolCallsFromTruncatedMessage(toolCalls, emit)
						: await executeToolCalls(currentContext, message, config, signal, emit);
				toolResults.push(...executedToolBatch.messages);
                // 依靠这一状态决定是否将tool结果再喂给llm，从而实现react，思考 -> 调用tool -> 获取结果继续思考 -> 调用tool -> 思考 -> ...... -> 结束
				hasMoreToolCalls = !executedToolBatch.terminate;

				for (const result of toolResults) {
					currentContext.messages.push(result);
					newMessages.push(result);
				}
			}

            // 到这里，这个turn已经结束了，所以我们可以理解一个turn就是
            // agent发消息给llm，llm响应，执行完llm响应需要执行的操作
			await emit({ type: "turn_end", message, toolResults });

            // 允许外部钩子prepareNextTurn调整上下文，调整model和思考强度，这个在ai ide都能看到
			const nextTurnContext = {
				messag
				toolResults,
				context: currentContext,
				newMessages,
			};
			const nextTurnSnapshot = await config.prepareNextTurn?.(nextTurnContext);
			if (nextTurnSnapshot) {
				currentContext = nextTurnSnapshot.context ?? currentContext;
				config = {
					...config,
					model: nextTurnSnapshot.model ?? config.model,
					reasoning:
						nextTurnSnapshot.thinkingLevel === undefined
							? config.reasoning
							: nextTurnSnapshot.thinkingLevel === "off"
								? undefined
								: nextTurnSnapshot.thinkingLevel,
				};
			}

			if (
                // 调用钩子判断是否停止，不过这不是之前abort中断信号，需要区分开
				await config.shouldStopAfterTurn?.({
					message,
					toolResults,
					context: currentContext,
					newMessages,
				})
			) {
				await emit({ type: "agent_end", messages: newMessages });
				return;
			}

            // 继续处理steer message，如果有就继续走inner loop处理
			pendingMessages = (await config.getSteeringMessages?.()) || [];
		}

		// 外层循环才处理followUpMessages，这就像正常chat的每一轮用户输入
		const followUpMessages = (await config.getFollowUpMessages?.()) || [];
		if (followUpMessages.length > 0) {
			// Set as pending so inner loop processes them
			pendingMessages = followUpMessages;
			continue;
		}

		// No more messages, exit
		break;
	}

	await emit({ type: "agent_end", messages: newMessages });
}
```
这算是比较经典的react模式
llm输出结果 -> 调用tool -> 携带tool调用结果再去调用llm -> llm输出结果 -> 调用tool
形成这样一个循环


## streamAssistantResponse
这里是与大模型通信的方法，应该会有一些细节，具体看看
```ts
async function streamAssistantResponse(
	context: AgentContext,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
	streamFunction: StreamFn,
): Promise<AssistantMessage> {
	// Apply context transform if configured (AgentMessage[] → AgentMessage[])
    // transformContext的注释写了他要干啥，就是message发送前先干一些操作，比如超上下文窗口了，先干掉一些旧 message
	let messages = context.messages;
	if (config.transformContext) {
		messages = await config.transformContext(messages, signal);
	}

	// Convert to LLM-compatible messages (AgentMessage[] → Message[])
	const llmMessages = await config.convertToLlm(messages);

	// Build LLM context
    // 虽然代码已经可以看出来干啥了，不过我们还是大致知道发送给llm一些啥东西
    // 系统上下文 + 对话记录 + tools(mcp本质也就是tools + prompts)
	const llmContext: Context = {
		systemPrompt: context.systemPrompt,
		messages: llmMessages,
		tools: context.tools,
	};

	// Resolve API key (important for expiring tokens)
	const resolvedApiKey =
		(config.getApiKey ? await config.getApiKey(config.model.provider) : undefined) || config.apiKey;

	const response = await streamFunction(config.model, llmContext, {
		...config,
		apiKey: resolvedApiKey,
		signal,
	});

	let partialMessage: AssistantMessage | null = null;
	let addedPartial = false;

    // 读llm的回复，慢慢刷新context的最后一条message
	for await (const event of response) {
		switch (event.type) {
			case "start":
				partialMessage = event.partial;
				context.messages.push(partialMessage);
				addedPartial = true;
				await emit({ type: "message_start", message: { ...partialMessage } });
				break;

			case "text_start":
			case "text_delta":
			case "text_end":
			case "thinking_start":
			case "thinking_delta":
			case "thinking_end":
			case "toolcall_start":
			case "toolcall_delta":
			case "toolcall_end":
				if (partialMessage) {
					partialMessage = event.partial;
					context.messages[context.messages.length - 1] = partialMessage;
					await emit({
						type: "message_update",
						assistantMessageEvent: event,
						message: { ...partialMessage },
					});
				}
				break;

			case "done":
			case "error": {
				const finalMessage = await response.result();
				if (addedPartial) {
					context.messages[context.messages.length - 1] = finalMessage;
				} else {
					context.messages.push(finalMessage);
				}
				if (!addedPartial) {
					await emit({ type: "message_start", message: { ...finalMessage } });
				}
				await emit({ type: "message_end", message: finalMessage });
				return finalMessage;
			}
		}
	}

	const finalMessage = await response.result();
	if (addedPartial) {
		context.messages[context.messages.length - 1] = finalMessage;
	} else {
		context.messages.push(finalMessage);
		await emit({ type: "message_start", message: { ...finalMessage } });
	}
	await emit({ type: "message_end", message: finalMessage });
	return finalMessage;
}
```

这算是整个的agent最核心的流程，我们再从全局看看，是否遗漏了什么细节

## continue
```ts
await this.agent.prompt(messages);
while (await this._handlePostAgentRun()) {
    await this.agent.continue();
}
```
在prompt后还会有一个continue流程，我们来看看怎么个事
```ts
private async _handlePostAgentRun(): Promise<boolean> {
    // 获取最新的last assistant messgae
    const msg = this._lastAssistantMessage;
    this._lastAssistantMessage = undefined;
    if (!msg) {
        return false;
    }

    // 可重试的错误 and retry准备
    if (this._isRetryableError(msg) && (await this._prepareRetry(msg))) {
        return true;
    }

    if (msg.stopReason === "error" && this._retryAttempt > 0) {
        this._emit({
            type: "auto_retry_end",
            success: false,
            attempt: this._retryAttempt,
            finalError: msg.errorMessage,
        });
        this._retryAttempt = 0;
    }

    // 之前的上下文压缩起作用了，如果willRetry为true，这里会重试
    if (await this._checkCompaction(msg)) {
        return true;
    }

    // 这里就是两个队列steer和followup有消息也要continue
    // The agent loop drains both queues before emitting agent_end. Any messages
    // here were queued by agent_end extension handlers and need a continuation.
    return this.agent.hasQueuedMessages();
}
```

再看看agent的continue是怎么个事
```ts
async continue(): Promise<void> {
    if (this.activeRun) {
        throw new Error("Agent is already processing. Wait for completion before continuing.");
    }

    const lastMessage = this._state.messages[this._state.messages.length - 1];
    if (!lastMessage) {
        throw new Error("No messages to continue from");
    }

    // 两个message如果有消息依旧走runPromptMessages -> runAgentLoop
    if (lastMessage.role === "assistant") {
        const queuedSteering = this.steeringQueue.drain();
        if (queuedSteering.length > 0) {
            await this.runPromptMessages(queuedSteering, { skipInitialSteeringPoll: true });
            return;
        }

        const queuedFollowUps = this.followUpQueue.drain();
        if (queuedFollowUps.length > 0) {
            await this.runPromptMessages(queuedFollowUps);
            return;
        }

        throw new Error("Cannot continue from message role: assistant");
    }

    // 这里就会调用新的runAgentLoopContinue
    await this.runContinuation();
}
```
我们看看runAgentLoopContinue与之前分析的runAgentLoop流程有何不用
和之前流程差不多了，除了多一个需要check最后一条消息非llm message
```ts
export async function runAgentLoopContinue(
	context: AgentContext,
	config: AgentLoopConfig,
	emit: AgentEventSink,
	signal: AbortSignal | undefined,
	streamFn: StreamFn,
): Promise<AgentMessage[]> {
	if (context.messages.length === 0) {
		throw new Error("Cannot continue: no messages in context");
	}

	if (context.messages[context.messages.length - 1].role === "assistant") {
		throw new Error("Cannot continue from message role: assistant");
	}

	const newMessages: AgentMessage[] = [];
	const currentContext: AgentContext = { ...context };

	await emit({ type: "agent_start" });
	await emit({ type: "turn_start" });

	await runLoop(currentContext, newMessages, config, signal, emit, streamFn ?? getDefaultStreamFn());
	return newMessages;
}
```


## 工具调用
这也是比较重要的一块，看看咋写的，agent的调用tool的逻辑主要在这块
```ts
if (toolCalls.length > 0) {
    // A "length" stop means the output was cut off by the token limit, so
    // every tool call in the message may carry truncated arguments. Fail
    // them all instead of executing potentially borked calls.
    const executedToolBatch =
        message.stopReason === "length"
            ? await failToolCallsFromTruncatedMessage(toolCalls, emit)
            : await executeToolCalls(currentContext, message, config, signal, emit);
    toolResults.push(...executedToolBatch.messages);
    hasMoreToolCalls = !executedToolBatch.terminate;

    for (const result of toolResults) {
        currentContext.messages.push(result);
        newMessages.push(result);
    }
}
```
接着就是executeToolCalls，看看具体调用流程是咋样的
```ts
async function executeToolCalls(
	currentContext: AgentContext,
	assistantMessage: AssistantMessage,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
): Promise<ExecutedToolCallBatch> {
    // 过滤出调用toolCall的Content
	const toolCalls = assistantMessage.content.filter((c) => c.type === "toolCall");
    // 有一个工具要求串行，就串行
	const hasSequentialToolCall = toolCalls.some(
		(tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
	);
	if (config.toolExecution === "sequential" || hasSequentialToolCall) {
		return executeToolCallsSequential(currentContext, assistantMessage, toolCalls, config, signal, emit);
	}
	return executeToolCallsParallel(currentContext, assistantMessage, toolCalls, config, signal, emit);
}
```
这里串行并行估计区别不大，先具体看看串行
```ts
async function executeToolCallsSequential(
	currentContext: AgentContext,
	assistantMessage: AssistantMessage,
	toolCalls: AgentToolCall[],
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
): Promise<ExecutedToolCallBatch> {
	const finalizedCalls: FinalizedToolCallOutcome[] = [];
	const messages: ToolResultMessage[] = [];

	for (const toolCall of toolCalls) {
		await emit({
			type: "tool_execution_start",
			toolCallId: toolCall.id,
			toolName: toolCall.name,
			args: toolCall.arguments,
		});

		const preparation = await prepareToolCall(currentContext, assistantMessage, toolCall, config, signal);
		let finalized: FinalizedToolCallOutcome;
		if (preparation.kind === "immediate") {
			finalized = {
				toolCall,
				result: preparation.result,
				isError: preparation.isError,
			};
		} else {
			const executed = await executePreparedToolCall(preparation, signal, emit);
			finalized = await finalizeExecutedToolCall(
				currentContext,
				assistantMessage,
				preparation,
				executed,
				config,
				signal,
			);
		}

        // 发送tool_execution_end and message_start and message_end事件，发送的真多
		await emitToolExecutionEnd(finalized, emit);
		const toolResultMessage = createToolResultMessage(finalized);
		await emitToolResultMessage(toolResultMessage, emit);
		finalizedCalls.push(finalized);
		messages.push(toolResultMessage);

		if (signal?.aborted) {
			break;
		}
	}

	return {
		messages,
		terminate: shouldTerminateToolBatch(finalizedCalls),
	};
}
```
先看看prepareToolCall
```ts
async function prepareToolCall(
	currentContext: AgentContext,
	assistantMessage: AssistantMessage,
	toolCall: AgentToolCall,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
): Promise<PreparedToolCall | ImmediateToolCallOutcome> {
    // 没有这个工具返回错误
	const tool = currentContext.tools?.find((t) => t.name === toolCall.name);
	if (!tool) {
		return {
			kind: "immediate",
			result: createErrorToolResult(`Tool ${toolCall.name} not found`),
			isError: true,
		};
	}

	try {
        // 准备调用参数和校验参数
		const preparedToolCall = prepareToolCallArguments(tool, toolCall);
		const validatedArgs = validateToolArguments(tool, preparedToolCall);
        // 执行钩子
		if (config.beforeToolCall) {
			const beforeResult = await config.beforeToolCall(
				{
					assistantMessage,
					toolCall,
					args: validatedArgs,
					context: currentContext,
				},
				signal,
			);
			if (signal?.aborted) {
				return {
					kind: "immediate",
					result: createErrorToolResult("Operation aborted"),
					isError: true,
				};
			}
			if (beforeResult?.block) {
				const result = createErrorToolResult(beforeResult.reason || "Tool execution was blocked");
                // 这里有机会设置result.terminate
				if (beforeResult.terminate === true) {
					result.terminate = true;
				}
				return {
					kind: "immediate",
					result,
					isError: true,
				};
			}
		}
        // 中断了？返回err
		if (signal?.aborted) {
			return {
				kind: "immediate",
				result: createErrorToolResult("Operation aborted"),
				isError: true,
			};
		}
		return {
			kind: "prepared",
			toolCall,
			tool,
			args: validatedArgs,
		};
	} catch (error) {
		return {
			kind: "immediate",
			result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
			isError: true,
		};
	}
}

// 这里并未设置result.terminate
function createErrorToolResult(message: string): AgentToolResult<any> {
	return {
		content: [{ type: "text", text: message }],
		details: {},
	};
}
```
prepare后就可以开始调用了
```ts
async function executePreparedToolCall(
	prepared: PreparedToolCall,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
): Promise<ExecutedToolCallOutcome> {
	const updateEvents: Promise<void>[] = [];
	let acceptingUpdates = true;

	try {
        // 调用的工具本身的execute方法
		const result = await prepared.tool.execute(
			prepared.toolCall.id,
			prepared.args as never,
			signal,
            // 提供一个部分更新结果的事件发射方法
			(partialResult) => {
				if (!acceptingUpdates) return;
				updateEvents.push(
					Promise.resolve(
						emit({
							type: "tool_execution_update",
							toolCallId: prepared.toolCall.id,
							toolName: prepared.toolCall.name,
							args: prepared.toolCall.arguments,
							partialResult,
						}),
					),
				);
			},
		);
		acceptingUpdates = false;
		await Promise.all(updateEvents);
		return { result, isError: false };
	} catch (error) {
		acceptingUpdates = false;
		await Promise.all(updateEvents);
		return {
			result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
			isError: true,
		};
	} finally {
		acceptingUpdates = false;
	}
}
```
并行是类似的，只有使用了一个Promise.all用来等待所有的tool调用完成
具体的tool怎么实现，设计，我觉得需要单开一章来讲讲

## 全局细节
### listeners
agent这个设计还算巧妙，因为pi的agent只是作为一个核心包，并不提供多余的功能，但是也需要通知外部，内部正在发生什么事情
这就设置了一个简单的发布订阅模式
```ts
private readonly listeners = new Set<(event: AgentEvent, signal: AbortSignal) => Promise<void> | void>();

// subscribe，同时将删除方法丢给外部，有点意思，
subscribe(listener: (event: AgentEvent, signal: AbortSignal) => Promise<void> | void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
}

private async processEvents(event: AgentEvent): Promise<void> {
    switch (event.type) {
        case "message_start":
            this._state.streamingMessage = event.message;
            break;

        case "message_update":
            this._state.streamingMessage = event.message;
            break;

        case "message_end":
            this._state.streamingMessage = undefined;
            this._state.messages.push(event.message);
            break;

        case "tool_execution_start": {
            const pendingToolCalls = new Set(this._state.pendingToolCalls);
            pendingToolCalls.add(event.toolCallId);
            this._state.pendingToolCalls = pendingToolCalls;
            break;
        }

        case "tool_execution_end": {
            const pendingToolCalls = new Set(this._state.pendingToolCalls);
            pendingToolCalls.delete(event.toolCallId);
            this._state.pendingToolCalls = pendingToolCalls;
            break;
        }

        case "turn_end":
            if (event.message.role === "assistant" && event.message.errorMessage) {
                this._state.errorMessage = event.message.errorMessage;
            }
            break;

        case "agent_end":
            this._state.streamingMessage = undefined;
            break;
    }

    const signal = this.activeRun?.abortController.signal;
    if (!signal) {
        throw new Error("Agent listener invoked outside active run");
    }
    // 接收到各类事件还需同步到外侧
    for (const listener of this.listeners) {
        await listener(event, signal);
    }
}
```

整体来讲这个agent就是loop + tool + event通知外部，设计非常简洁