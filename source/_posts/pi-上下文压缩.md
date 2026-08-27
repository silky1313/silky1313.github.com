---

title: PI分析-上下文压缩
date: 2026-8-26 23:07:00
categories: 

- agent

---

对一些逻辑较为复杂的点进行记录

执行压缩总共就两种情况

1. LLM返回上下文超出长度 或者 llm输出长度低于输出maxtoken（我理解为取决于LLM的响应决定是否压缩）

```ts
// 还没撞到desiredMaxOutput上限，但是报错了，压缩后重试
export function isRecoverableLength(message: AssistantMessage, desiredMaxOutput: number): boolean {	return message.stopReason === "length" && desiredMaxOutput > 0 && message.usage.output < desiredMaxOutput;
}

// 超出上下文长度
export function isContextOverflow(message: AssistantMessage, contextWindow?: number): boolean {
	// Case 1: Check error message patterns
	if (message.stopReason === "error" && message.errorMessage) {
		// Skip messages matching known non-overflow patterns (e.g. throttling / rate-limit)
		const isNonOverflow = NON_OVERFLOW_PATTERNS.some((p) => p.test(message.errorMessage!));
		if (!isNonOverflow && OVERFLOW_PATTERNS.some((p) => p.test(message.errorMessage!))) {
			return true;
		}
	}

	// Case 2: Silent overflow (z.ai style) - successful but usage exceeds context
	if (contextWindow && message.stopReason === "stop") {
		const inputTokens = message.usage.input + message.usage.cacheRead;
		if (inputTokens > contextWindow) {
			return true;
		}
	}

	// Case 3: Length-stop overflow (Xiaomi MiMo style) - server truncates oversized input
	// to fit the context window, leaving no room for output. Returns stopReason "length"
	// with output=0 and input+cacheRead filling the context window.
	if (contextWindow && message.stopReason === "length" && message.usage.output === 0) {
		const inputTokens = message.usage.input + message.usage.cacheRead;
		if (inputTokens >= contextWindow * 0.99) {
			return true;
		}
	}

	return false;
}
```

1. 框架计算超出上下文超出长度

```ts
let contextTokens: number;
const directContextTokens = assistantMessage.usage ? calculateContextTokens(assistantMessage.usage) : 0;
if (assistantMessage.stopReason === "error" || directContextTokens === 0) {
	const messages = this.agent.state.messages;
	const estimate = estimateContextTokens(messages);
	if (estimate.lastUsageIndex === null) return false; // No usage data at all
	// Verify the usage source is post-compaction. Kept pre-compaction messages
	// have stale usage reflecting the old (larger) context and would falsely
	// trigger compaction right after one just finished.
	const usageMsg = messages[estimate.lastUsageIndex];
	if (
		compactionEntry &&
		usageMsg.role === "assistant" &&
		(usageMsg as AssistantMessage).timestamp <= new Date(compactionEntry.timestamp).getTime()
	) {
		return false;
	}
	contextTokens = estimate.tokens;
} else {
	contextTokens = directContextTokens;
}
if (shouldCompact(contextTokens, contextWindow, settings)) {
	return await this._runAutoCompaction("threshold", false);
}
return false;
```



## _runAutoCompaction

压缩的核心方法，具体看看

### 准备阶段

```ts
// pi组织一次回话是一颗树妆的，这是为了支持回退，pathEntries你可以理解为当前的回话记录
export function prepareCompaction(
	pathEntries: SessionEntry[],
	settings: CompactionSettings,
): CompactionPreparation | undefined {...}
```

寻找压缩范围，从上次压缩点往后压缩

```ts
// 找到上一个压缩点
let prevCompactionIndex = -1;
for (let i = pathEntries.length - 1; i >= 0; i--) {
	if (pathEntries[i].type === "compaction") {
		prevCompactionIndex = i;
		break;
	}
}

// 压缩从上一压缩点的firstKeptEntryIndex -> pathEntries.length;
let previousSummary: string | undefined;
let boundaryStart = 0;
if (prevCompactionIndex >= 0) {
	const prevCompaction = pathEntries[prevCompactionIndex] as CompactionEntry;
	previousSummary = prevCompaction.summary;
	// 将id -> index, id是uuid
	const firstKeptEntryIndex = pathEntries.findIndex((entry) => entry.id === prevCompaction.firstKeptEntryId);
	boundaryStart = firstKeptEntryIndex >= 0 ? firstKeptEntryIndex : prevCompactionIndex + 1;
}
const boundaryEnd = pathEntries.length;
```

寻找有效cutpoints，因为不能把tool result割开，这是不完整的

```ts
// 保留最新的keepRecentTokens个token
const cutPoint = findCutPoint(pathEntries, boundaryStart, boundaryEnd, settings.keepRecentTokens);

export function findCutPoint(
	entries: SessionEntry[],
	startIndex: number,
	endIndex: number,
	keepRecentTokens: number,
): CutPointResult {
	const cutPoints = findValidCutPoints(entries, startIndex, endIndex);
	// ...
}

function findValidCutPoints(entries: SessionEntry[], startIndex: number, endIndex: number): number[] {
	const cutPoints: number[] = [];
	for (let i = startIndex; i < endIndex; i++) {
		const entry = entries[i];
		if (entry.type === "compaction") {
			continue;
		}
		if (sessionEntryToContextMessages(entry).some(isCutPointMessage)) {
			cutPoints.push(i);
		}
	}
	return cutPoints;
}

function isCutPointMessage(message: AgentMessage): boolean {
	switch (message.role) {
		case "user":
		case "assistant":
		case "bashExecution":
		case "custom":
		case "branchSummary":
		case "compactionSummary":
			return true;
		case "toolResult":
		// tool results are not cut points
			return false;
	}
	return false;
}
```

接下来开始寻找真正的cutpoints

```ts
export function findCutPoint(
	entries: SessionEntry[],
	startIndex: number,
	endIndex: number,
	keepRecentTokens: number,
): CutPointResult {
	const cutPoints = findValidCutPoints(entries, startIndex, endIndex);

	if (cutPoints.length === 0) {
		return { firstKeptEntryIndex: startIndex, turnStartIndex: -1, isSplitTurn: false };
	}

	// Walk backwards from newest, accumulating estimated message sizes
	let accumulatedTokens = 0;
	let cutIndex = cutPoints[0]; // Default: keep from first message (not header)

	// 保留最近的keepRecentTokens个token，获取最近的切分点
	for (let i = endIndex - 1; i >= startIndex; i--) {
		const entry = entries[i];
		const messageTokens = sessionEntryToContextMessages(entry).reduce(
			(sum, message) => sum + estimateTokens(message),
			0,
		);
		if (messageTokens === 0) continue;
		accumulatedTokens += messageTokens;

		// Check if we've exceeded the budget
		if (accumulatedTokens >= keepRecentTokens) {
			// Find the closest valid cut point at or after this entry
			for (let c = 0; c < cutPoints.length; c++) {
				if (cutPoints[c] >= i) {
					cutIndex = cutPoints[c];
					break;
				}
			}
			break;
		}
	}

	// 过滤掉一些不占上下文的entry，没啥影响，所以不参与压缩
	while (cutIndex > startIndex) {
		const prevEntry = entries[cutIndex - 1];
		// Stop at compaction boundaries or context-visible entries.
		// 元数据对
		if (prevEntry.type === "compaction" || sessionEntryToContextMessages(prevEntry).length > 0) {
			break;
		}
		cutIndex--;
	}

	// 如果cutpoint不是turn start，则压缩会从压缩一个不完整的turn
	const cutEntry = entries[cutIndex];
	const startsTurn = isTurnStartEntry(cutEntry);
	const turnStartIndex = startsTurn ? -1 : findTurnStartIndex(entries, cutIndex, startIndex);

	return {
		firstKeptEntryIndex: cutIndex,
		turnStartIndex, // turn 开始的index
		isSplitTurn: !startsTurn && turnStartIndex !== -1, // 是否会压缩一个split turn
	};
}
```

寻找完cutpoint继续回到prepareCompaction

```ts
// 找到分割点
const cutPoint = findCutPoint(pathEntries, boundaryStart, boundaryEnd, settings.keepRecentTokens);

const firstKeptEntry = pathEntries[cutPoint.firstKeptEntryIndex];
if (!firstKeptEntry?.id) {
	return undefined; // Session needs migration
}
const firstKeptEntryId = firstKeptEntry.id;

const historyEnd = cutPoint.isSplitTurn ? cutPoint.turnStartIndex : cutPoint.firstKeptEntryIndex;

// 完整的turn
const messagesToSummarize: AgentMessage[] = [];
for (let i = boundaryStart; i < historyEnd; i++) {
	const msg = getMessageFromEntryForCompaction(pathEntries[i]);
	if (msg) messagesToSummarize.push(msg);
}

// split turn
const turnPrefixMessages: AgentMessage[] = [];
if (cutPoint.isSplitTurn) {
	for (let i = cutPoint.turnStartIndex; i < cutPoint.firstKeptEntryIndex; i++) {
		const msg = getMessageFromEntryForCompaction(pathEntries[i]);
		if (msg) turnPrefixMessages.push(msg);
	}
}

if (messagesToSummarize.length === 0 && turnPrefixMessages.length === 0) {
	return undefined;
}

// 这两步是收集文件操作，read/write/edit，对于系统来说，这是精确消息
const fileOps = extractFileOperations(messagesToSummarize, pathEntries, prevCompactionIndex);
if (cutPoint.isSplitTurn) {
	for (const msg of turnPrefixMessages) {
		extractFileOpsFromMessage(msg, fileOps);
	}
}

return {
	firstKeptEntryId,
	messagesToSummarize,
	turnPrefixMessages,
	isSplitTurn: cutPoint.isSplitTurn,
	tokensBefore,
	previousSummary,
	fileOps,
	settings,
};
```



### 正式开始

```ts
let extensionCompaction: CompactionResult | undefined;
let fromExtension = false;

// 存在拓展处理compact，交给拓展处理
if (this._extensionRunner.hasHandlers("session_before_compact")) {
	const result = (await this._extensionRunner.emit({
		type: "session_before_compact",
		preparation,
		branchEntries: pathEntries,
		customInstructions,
		reason: "manual",
		willRetry: false,
		signal: this._compactionAbortController.signal,
	})) as SessionBeforeCompactResult | undefined;

	if (result?.cancel) {
		throw new Error("Compaction cancelled");
	}

	if (result?.compaction) {
		extensionCompaction = result.compaction;
		fromExtension = true;
	}
}

let summary: string;
let firstKeptEntryId: string;
let tokensBefore: number;
let usage: Usage | undefined;
let details: unknown;

if (extensionCompaction) {
	// Extension provided compaction content
	summary = extensionCompaction.summary;
	firstKeptEntryId = extensionCompaction.firstKeptEntryId;
	tokensBefore = extensionCompaction.tokensBefore;
	usage = extensionCompaction.usage;
	details = extensionCompaction.details;
} else {
	// 内置压缩流程
	const result = await compact(
		preparation,
		requestModel,
		apiKey,
		headers,
		customInstructions,
		this._compactionAbortController.signal,
		this.thinkingLevel,
		this.agent.streamFunction,
		env,
		this.settingsManager.getRetrySettings(),
		this._summarizationRetryCallbacks({ source: "compaction", reason: "manual" }),
	);
	summary = result.summary;
	firstKeptEntryId = result.firstKeptEntryId;
	tokensBefore = result.tokensBefore;
	usage = result.usage;
	details = result.details;
}
```

接下来我们看看内置的压缩流程

```ts
export async function compact(
	preparation: CompactionPreparation,
	model: Model<any>,
	apiKey: string | undefined,
	headers?: Record<string, string>,
	customInstructions?: string,
	signal?: AbortSignal,
	thinkingLevel?: ThinkingLevel,
	streamFn?: StreamFn,
	env?: Record<string, string>,
	retry?: RetryPolicy,
	callbacks?: RetryCallbacks,
): Promise<CompactionResult> {
	const {
		firstKeptEntryId,
		messagesToSummarize,
		turnPrefixMessages,
		isSplitTurn,
		tokensBefore, 	// 
		previousSummary, // 上一次的压缩概览
		fileOps,
		settings,
	} = preparation;


	let summary: string;
	let summaryUsage: Usage;

	// 需要压缩一个split turn，两种处理情况不同
	if (isSplitTurn && turnPrefixMessages.length > 0) {
		let historyText = "No prior history.";
		let historyUsage: Usage | undefined;
		if (messagesToSummarize.length > 0) {
			const historyResult = await generateSummaryWithUsage(
				messagesToSummarize,
				model,
				settings.reserveTokens,
				apiKey,
				headers,
				signal,
				customInstructions,
				previousSummary,
				thinkingLevel,
				streamFn,
				env,
				retry,
				callbacks,
			);
			historyText = historyResult.text;
			historyUsage = historyResult.usage;
		}
		const turnPrefixResult = await generateTurnPrefixSummary(
			turnPrefixMessages,
			model,
			settings.reserveTokens,
			apiKey,
			headers,
			env,
			signal,
			thinkingLevel,
			streamFn,
			retry,
			callbacks,
		);
		// Merge into single summary
		summary = `${historyText}\n\n---\n\n**Turn Context (split turn):**\n\n${turnPrefixResult.text}`;
		summaryUsage = historyUsage ? combineUsage(historyUsage, turnPrefixResult.usage) : turnPrefixResult.usage;
	} else {
		// Just generate history summary
		const result = await generateSummaryWithUsage(
			messagesToSummarize,
			model,
			settings.reserveTokens,
			apiKey,
			headers,
			signal,
			customInstructions,
			previousSummary,
			thinkingLevel,
			streamFn,
			env,
			retry,
			callbacks,
		);
		summary = result.text;
		summaryUsage = result.usage;
	}

	// 文件操作精确计算后直接合并到summary中
	const { readFiles, modifiedFiles } = computeFileLists(fileOps);
	summary += formatFileOperations(readFiles, modifiedFiles);

	if (!firstKeptEntryId) {
		throw new Error("First kept entry has no UUID - session may need migration");
	}

	return {
		summary,
		firstKeptEntryId,
		tokensBefore,
		usage: summaryUsage,
		details: { readFiles, modifiedFiles } as CompactionDetails,
	};
}
```

两个压缩方法

```ts
/** Generate or update a conversation summary and return its provider usage. */
export async function generateSummaryWithUsage(
	currentMessages: AgentMessage[],
	model: Model<any>,
	reserveTokens: number,
	apiKey: string | undefined,
	headers?: Record<string, string>,
	signal?: AbortSignal,
	customInstructions?: string,
	previousSummary?: string,
	thinkingLevel?: ThinkingLevel,
	streamFn?: StreamFn,
	env?: Record<string, string>,
	retry?: RetryPolicy,
	callbacks?: RetryCallbacks,
): Promise<{ text: string; usage: Usage }> {
	const maxTokens = Math.min(
		Math.floor(0.8 * reserveTokens), // reserveTokens是保留input token，为 下一次用户输入和system prompt准备
		model.maxTokens > 0 ? model.maxTokens : Number.POSITIVE_INFINITY, // 这个maxToken是llm输出的maxtoken
	);

	// UPDATE_SUMMARIZATION_PROMPT是会基于previousSummary去生成summary，SUMMARIZATION_PROMPT则是生成一份新的summary
	let basePrompt = previousSummary ? UPDATE_SUMMARIZATION_PROMPT : SUMMARIZATION_PROMPT;
	if (customInstructions) {
		basePrompt = `${basePrompt}\n\nAdditional focus: ${customInstructions}`;
	}

	// 将内部流转的AgentMessage -> LLM Message -> string
	const llmMessages = convertToLlm(currentMessages);
	const conversationText = serializeConversation(llmMessages);

	// 整理整个上下文 basePrompt + previousSummary + conversationText
	let promptText = `<conversation>\n${conversationText}\n</conversation>\n\n`;
	if (previousSummary) {
		promptText += `<previous-summary>\n${previousSummary}\n</previous-summary>\n\n`;
	}
	promptText += basePrompt;

	const summarizationMessages = [
		{
			role: "user" as const,
			content: [{ type: "text" as const, text: promptText }],
			timestamp: Date.now(),
		},
	];

	// 调用llm压缩
	const completionOptions = createSummarizationOptions(model, maxTokens, apiKey, headers, env, signal, thinkingLevel);

	const response = await completeSummarization(
		model,
		{ systemPrompt: SUMMARIZATION_SYSTEM_PROMPT, messages: summarizationMessages },
		completionOptions,
		streamFn,
		retry,
		callbacks,
	);

	if (response.stopReason === "error") {
		throw new Error(`Summarization failed: ${response.errorMessage || "Unknown error"}`);
	}

	const textContent = contentText(response.content);

	return { text: textContent, usage: response.usage };
}
```

```ts
/**
 * Generate a summary for a turn prefix (when splitting a turn).
 */
async function generateTurnPrefixSummary(
	messages: AgentMessage[],
	model: Model<any>,
	reserveTokens: number,
	apiKey: string | undefined,
	headers?: Record<string, string>,
	env?: Record<string, string>,
	signal?: AbortSignal,
	thinkingLevel?: ThinkingLevel,
	streamFn?: StreamFn,
	retry?: RetryPolicy,
	callbacks?: RetryCallbacks,
): Promise<{ text: string; usage: Usage }> {
	// 这几乎是唯一区别，对这个的maxTokens的计算方式不同，这个maxtokens给的更小
	// 原因是因为这里只有半个turn，没必要给这么大的输出上限
	const maxTokens = Math.min(
		Math.floor(0.5 * reserveTokens),
		model.maxTokens > 0 ? model.maxTokens : Number.POSITIVE_INFINITY,
	); // Smaller budget for turn prefix
	const llmMessages = convertToLlm(messages);
	const conversationText = serializeConversation(llmMessages);
	const promptText = `<conversation>\n${conversationText}\n</conversation>\n\n${TURN_PREFIX_SUMMARIZATION_PROMPT}`;
	const summarizationMessages = [
		{
			role: "user" as const,
			content: [{ type: "text" as const, text: promptText }],
			timestamp: Date.now(),
		},
	];

	const response = await completeSummarization(
		model,
		{ systemPrompt: SUMMARIZATION_SYSTEM_PROMPT, messages: summarizationMessages },
		createSummarizationOptions(model, maxTokens, apiKey, headers, env, signal, thinkingLevel),
		streamFn,
		retry,
		callbacks,
	);

	if (response.stopReason === "error") {
		throw new Error(`Turn prefix summarization failed: ${response.errorMessage || "Unknown error"}`);
	}

	return {
		text: contentText(response.content),
		usage: response.usage,
	};
}
```



### 压缩后处理

```ts
// append compact entry到会话树中
this.sessionManager.appendCompaction(summary, firstKeptEntryId, tokensBefore, details, fromExtension, usage);
const newEntries = this.sessionManager.getEntries();
const sessionContext = this.sessionManager.buildSessionContext();
this.agent.state.messages = sessionContext.messages;
const estimatedTokensAfter = estimateMessagesTokens(sessionContext.messages);

// 获取到最新的压缩Entry
const savedCompactionEntry = newEntries.find((e) => e.type === "compaction" && e.summary === summary) as
	| CompactionEntry
	| undefined;

// 拓展处理，发送压缩event看是否有插件关注
if (this._extensionRunner && savedCompactionEntry) {
	await this._extensionRunner.emit({
		type: "session_compact",
		compactionEntry: savedCompactionEntry,
		fromExtension,
		reason,
		willRetry,
	});
}

const result: CompactionResult = {
	summary,
	firstKeptEntryId,
	tokensBefore, // 压缩前的token有多少？
	estimatedTokensAfter, // 压缩后的token有多少？
	usage,
	details,
};
this._emit({ type: "compaction_end", reason, result, aborted: false, willRetry });

// 上轮对话是否重试？如果上轮对话来自于llm且值得重试，就干掉最后一条消息并重试
// 返回true代表让agent继续与llm通信
if (willRetry) {
	const messages = this.agent.state.messages;
	const lastMsg = messages[messages.length - 1];
	// The overflow response was persisted on message_end before _checkCompaction() removed it
	// from agent state. Rebuilding state from the new compaction can restore that kept entry,
	// leaving an assistant as the final message. agent.continue() rejects that state, so remove
	// the retriable error or truncated-length response again before continuing the interrupted turn.
	if (lastMsg?.role === "assistant" && (lastMsg.stopReason === "error" || lastMsg.stopReason === "length")) {
		this.agent.state.messages = messages.slice(0, -1);
	}
	return true;
}

// 否责压缩完成后看是否有queue message，没有就停止等待
return this.agent.hasQueuedMessages();
```

