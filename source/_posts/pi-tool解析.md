title: PI分析-tool解析
date: 2026-9-2 23:07:00
categories: 

- agent

---

这里我们具体看看一个工具咋定义以及如何被使用
我们以ls为例子

# 创建一个工具

```ts
export function createLsTool(cwd: string, options?: LsToolOptions): AgentTool<typeof lsSchema> {
	return wrapToolDefinition(createLsToolDefinition(cwd, options));
}
```

这里我们从内层开始往外解剖，依次看看每层需要返回一个什么格式

```ts
// 需要返回一个ToolDefinition
export function createLsToolDefinition(
	cwd: string,
	options?: LsToolOptions,
): ToolDefinition<typeof lsSchema, LsToolDetails | undefined> {...}

export interface ToolDefinition<TParams extends TSchema = TSchema, TDetails = unknown, TState = any> {
	/** Tool name (used in LLM tool calls) */
	name: string;
	/** Human-readable label for UI */
	label: string;
	/** Description for LLM */
	description: string;
	/** Optional one-line snippet for the Available tools section in the default system prompt. Custom tools are omitted from that section when this is not provided. */
	promptSnippet?: string;
	/** Optional guideline bullets appended to the default system prompt Guidelines section when this tool is active. */
	promptGuidelines?: string[];
	/** Parameter schema (TypeBox) */
	parameters: TParams;
	/** Optional provider-side constrained sampling request for this tool. Set false to explicitly disable it, equivalent to leaving it undefined. */
	// llm生成参数时受限采样，保证模型输出符合预期的结果
	constrainedSampling?: false | ConstrainedSamplingConfig;
	/** Controls whether ToolExecutionComponent renders the standard colored shell or the tool renders its own framing. */
	// 这个字段是控制工具执行组件是渲染标准彩色 shell 还是工具自己渲染框架的开关。
	renderShell?: "default" | "self";

	/** Optional compatibility shim to prepare raw tool call arguments before schema validation. Must return an object conforming to TParams. */
	prepareArguments?: (args: unknown) => Static<TParams>;

	/**
	 * Per-tool execution mode override.
	 * - "sequential": this tool must execute one at a time with other tool calls.
	 * - "parallel": this tool can execute concurrently with other tool calls.
	 *
	 * If omitted, the default execution mode applies.
	 */
	executionMode?: ToolExecutionMode;

	/** Execute the tool. */
	execute(
		toolCallId: string,
		params: Static<TParams>,
		signal: AbortSignal | undefined,
		onUpdate: AgentToolUpdateCallback<TDetails> | undefined,
		ctx: ExtensionContext,
	): Promise<AgentToolResult<TDetails>>;

	/** Custom rendering for tool call display */
	renderCall?: (args: Static<TParams>, theme: Theme, context: ToolRenderContext<TState, Static<TParams>>) => Component;

	/** Custom rendering for tool result display */
	renderResult?: (
		result: AgentToolResult<TDetails>,
		options: ToolRenderResultOptions,
		theme: Theme,
		context: ToolRenderContext<TState, Static<TParams>>,
	) => Component;
}
```

那么createLsToolDefinition方法的内容应该就是实现这个interface了

```ts
export function createLsToolDefinition(
	cwd: string,
	options?: LsToolOptions,
): ToolDefinition<typeof lsSchema, LsToolDetails | undefined> {
	const ops = options?.operations ?? defaultLsOperations;
	return {
		name: "ls",
		label: "ls",
		description: `List directory contents. Returns entries sorted alphabetically, with '/' suffix for directories. Includes dotfiles. Output is truncated to ${DEFAULT_LIMIT} entries or ${DEFAULT_MAX_BYTES / 1024}KB (whichever is hit first).`,
		promptSnippet: "List directory contents",
		parameters: lsSchema,
		async execute(
			_toolCallId,
			{ path, limit }: { path?: string; limit?: number },
			signal?: AbortSignal,
			_onUpdate?,
			_ctx?,
		) {
			return new Promise((resolve, reject) => {
				if (signal?.aborted) {
					reject(new Error("Operation aborted"));
					return;
				}

				const onAbort = () => reject(new Error("Operation aborted"));
				signal?.addEventListener("abort", onAbort, { once: true });

				(async () => {
					try {
						const dirPath = resolveToCwd(path || ".", cwd);
						const effectiveLimit = limit ?? DEFAULT_LIMIT;

						// Check if path exists.
						if (!(await ops.exists(dirPath))) {
							reject(new Error(`Path not found: ${dirPath}`));
							return;
						}

						// Check if path is a directory.
						const stat = await ops.stat(dirPath);
						if (!stat.isDirectory()) {
							reject(new Error(`Not a directory: ${dirPath}`));
							return;
						}

						// Read directory entries.
						let entries: string[];
						try {
							entries = await ops.readdir(dirPath);
						} catch (e: any) {
							reject(new Error(`Cannot read directory: ${e.message}`));
							return;
						}

						// Sort alphabetically, case-insensitive.
						entries.sort((a, b) => a.toLowerCase().localeCompare(b.toLowerCase()));

						// Format entries with directory indicators.
						const results: string[] = [];
						let entryLimitReached = false;
						for (const entry of entries) {
							if (results.length >= effectiveLimit) {
								entryLimitReached = true;
								break;
							}

							const fullPath = nodePath.join(dirPath, entry);
							let suffix = "";
							try {
								const entryStat = await ops.stat(fullPath);
								if (entryStat.isDirectory()) suffix = "/";
							} catch {
								// Skip entries we cannot stat.
								continue;
							}
							results.push(entry + suffix);
						}

						signal?.removeEventListener("abort", onAbort);

						if (results.length === 0) {
							resolve({ content: [{ type: "text", text: "(empty directory)" }], details: undefined });
							return;
						}

						const rawOutput = results.join("\n");
						// 截断输出，避免达到行数上限 or 字节上限，不过之前已经限制了行数上限，所以这里只限制字节上限
						const truncation = truncateHead(rawOutput, { maxLines: Number.MAX_SAFE_INTEGER });
						let output = truncation.content;
						const details: LsToolDetails = {};
						// Build actionable notices for truncation and entry limits.
						const notices: string[] = [];
						if (entryLimitReached) {
							notices.push(`${effectiveLimit} entries limit reached. Use limit=${effectiveLimit * 2} for more`);
							details.entryLimitReached = effectiveLimit;
						}
						if (truncation.truncated) {
							notices.push(`${formatSize(DEFAULT_MAX_BYTES)} limit reached`);
							details.truncation = truncation;
						}
						if (notices.length > 0) {
							output += `\n\n[${notices.join(". ")}]`;
						}

						resolve({
							content: [{ type: "text", text: output }],
							details: Object.keys(details).length > 0 ? details : undefined,
						});
					} catch (e: any) {
						signal?.removeEventListener("abort", onAbort);
						reject(e);
					}
				})();
			});
		},
		renderCall(args, theme, context) {
			const text = (context.lastComponent as Text | undefined) ?? new Text("", 0, 0);
			text.setText(formatLsCall(args, theme, context.cwd));
			return text;
		},
		renderResult(result, options, theme, context) {
			const text = (context.lastComponent as Text | undefined) ?? new Text("", 0, 0);
			text.setText(formatLsResult(result as any, options, theme, context.showImages));
			return text;
		},
	};
}
```

往外再看看wrapToolDefinition，这里稍微封装了一下，多余的方法就没有传导到AgentTool

```ts
export function wrapToolDefinition<TDetails = unknown>(
	definition: ToolDefinition<any, TDetails>,
	ctxFactory?: () => ExtensionContext,
): AgentTool<any, TDetails> {
	return {
		name: definition.name,
		label: definition.label,
		description: definition.description,
		parameters: definition.parameters,
		constrainedSampling: definition.constrainedSampling,
		prepareArguments: definition.prepareArguments,
		executionMode: definition.executionMode,
		execute: (toolCallId, params, signal, onUpdate, ctx?: ExtensionContext) =>
			definition.execute(toolCallId, params, signal, onUpdate, ctx ?? (ctxFactory?.() as ExtensionContext)),
	};
}
```



# 使用一个工具

这里我们coding-agent为例子

```ts
const baseToolDefinitions = this._baseToolsOverride
	? Object.fromEntries(
			Object.entries(this._baseToolsOverride).map(([name, tool]) => [
				name,
				createToolDefinitionFromAgentTool(tool),
			]),
		)
	: createAllToolDefinitions(this._cwd, {
			read: { autoResizeImages },
			bash: { commandPrefix: shellCommandPrefix, shellPath },
		});

this._baseToolDefinitions = new Map(
	Object.entries(baseToolDefinitions).map(([name, tool]) => [name, tool as ToolDefinition]),
);
```

这里会调用createAllToolDefinitions创建默认的所有tools

```ts
export function createAllToolDefinitions(cwd: string, options?: ToolsOptions): Record<ToolName, ToolDef> {
	return {
		read: createReadToolDefinition(cwd, options?.read),
		bash: createBashToolDefinition(cwd, options?.bash),
		edit: createEditToolDefinition(cwd, options?.edit),
		write: createWriteToolDefinition(cwd, options?.write),
		grep: createGrepToolDefinition(cwd, options?.grep),
		find: createFindToolDefinition(cwd, options?.find),
		ls: createLsToolDefinition(cwd, options?.ls),
	};
}

export interface ToolsOptions {
	read?: ReadToolOptions;
	bash?: BashToolOptions;
	write?: WriteToolOptions;
	edit?: EditToolOptions;
	grep?: GrepToolOptions;
	find?: FindToolOptions;
	ls?: LsToolOptions;
}

export type ToolName = "read" | "bash" | "edit" | "write" | "grep" | "find" | "ls";
export type ToolDef = ToolDefinition<any, any>;
```

这里实际上就返回所有所有可用工具来，同时满足ToolDefinition接口的定义

tool与大模型的交互，可以引申出下面的三个问题

- 如何让模型知道有哪些tool？
- 模型如何调用tool？
- 调用tool后的结果如何塞会context？



## 如何让llm知道有哪些tool？

```ts
set tools(tools: AgentTool<any>[]);
get tools(): AgentTool<any>[];

// 这里传入tool作为context的一部分
private createContextSnapshot(): AgentContext {
	return {
		systemPrompt: this._state.systemPrompt,
		messages: this._state.messages.slice(),
		tools: this._state.tools.slice(),
	};
}

// runAgentLoop的传入createContextSnapshot创建的context
await runAgentLoop(
	messages,
	this.createContextSnapshot(),
	this.createLoopConfig(options),
	(event) => this.processEvents(event),
	signal,
	this.streamFunction,
);

// 这个方法与大模型交互
async function streamAssistantResponse(
	context: AgentContext,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
	streamFunction: StreamFn,
): Promise<AssistantMessage> {
	// ....

	// context -> llmcontext, tool也带进去了
	const llmContext: Context = {
		systemPrompt: context.systemPrompt,
		messages: llmMessages,
		tools: context.tools,
	};

	const response = await streamFunction(config.model, llmContext, {
		...config,
		apiKey: resolvedApiKey,
		signal,
	});
	
	//...
}
```

这里就得看看具体的协议决定这个tool是怎么被序列化传递的了
这里我们以Anthropic 协议为例子，具体在packages/ai/src/api/anthropic-messages.ts
```ts
export interface Tool<TParameters extends TSchema = TSchema> {
	name: string;
	description: string;
	parameters: TParameters;
	constrainedSampling?: false | ConstrainedSamplingConfig;
}

function convertTools(
	tools: Tool[],
	isOAuthToken: boolean,
	supportsEagerToolInputStreaming: boolean,
	supportsStrictTools: boolean,
	cacheControl?: CacheControlEphemeral,
	deferLoading = false,
): Anthropic.Messages.Tool[] {
	if (!tools) return [];

	return tools.map((tool, index) => {
		// 把tool 入参洗为claude 要求的格式，具体细节在此处不看了
		const strict = resolveJsonSchemaStrictSampling(tool, supportsStrictTools);
		const parameters = getJsonSchemaToolParameters(tool, strict);
		const schema = parameters as { properties?: unknown; required?: string[] };
		const legacyInputSchema = {
			type: "object" as const,
			properties: schema.properties ?? {},
			required: schema.required ?? [],
		};
		const inputSchema =
			strict === true
				? {
						...(parameters as Record<string, unknown>),
						...legacyInputSchema,
					}
				: legacyInputSchema;

		// 这里有一些其他参数，不过三个必须参数分别是name，description和input_schema
		return {
			name: isOAuthToken ? toClaudeCodeName(tool.name) : tool.name,
			description: tool.description,
			...(supportsEagerToolInputStreaming ? { eager_input_streaming: true } : {}),
			...(strict === true ? { strict: true } : {}),
			input_schema: inputSchema,
			...(deferLoading ? { defer_loading: true } : {}),
			...(cacheControl && index === tools.length - 1 ? { cache_control: cacheControl } : {}),
		};
	});
}
```

其实我们前面看到`ToolDefinition`的含义明显更加丰富，肯定还有有一些注入提示词的地方
从AgentSession来看这个链条
`constructor -> _buildRuntime -> _refreshToolRegistry -> setActiveToolsByName -> _rebuildSystemPrompt`

```ts
private _rebuildSystemPrompt(toolNames: string[]): string {
	// 获取到snippet和guideline
	const validToolNames = toolNames.filter((name) => this._toolRegistry.has(name));
	const toolSnippets: Record<string, string> = {};
	const promptGuidelines: string[] = [];
	for (const name of validToolNames) {
		const snippet = this._toolPromptSnippets.get(name);
		if (snippet) {
			toolSnippets[name] = snippet;
		}

		const toolGuidelines = this._toolPromptGuidelines.get(name);
		if (toolGuidelines) {
			promptGuidelines.push(...toolGuidelines);
		}
	}

	const loaderSystemPrompt = this._resourceLoader.getSystemPrompt();
	const loaderAppendSystemPrompt = this._resourceLoader.getAppendSystemPrompt();
	const appendSystemPrompt =
		loaderAppendSystemPrompt.length > 0 ? loaderAppendSystemPrompt.join("\n\n") : undefined;
	const loadedSkills = this._resourceLoader.getSkills().skills;
	const loadedContextFiles = this._resourceLoader.getAgentsFiles().agentsFiles;

	this._baseSystemPromptOptions = {
		cwd: this._cwd,
		skills: loadedSkills,
		contextFiles: loadedContextFiles,
		customPrompt: loaderSystemPrompt,
		appendSystemPrompt,
		selectedTools: validToolNames,
		toolSnippets,
		promptGuidelines,
	};
	// 这里重写整个提示词
	return buildSystemPrompt(this._baseSystemPromptOptions);
}

/** Build the system prompt with tools, guidelines, and context */
export function buildSystemPrompt(options: BuildSystemPromptOptions): string {
	const {
		customPrompt,
		selectedTools,
		toolSnippets,
		promptGuidelines,
		appendSystemPrompt,
		cwd,
		contextFiles: providedContextFiles,
		skills: providedSkills,
	} = options;
	const promptCwd = cwd.replace(/\\/g, "/");

	const appendSection = appendSystemPrompt ? `\n\n${appendSystemPrompt}` : "";

	const contextFiles = providedContextFiles ?? [];
	const skills = providedSkills ?? [];

	if (customPrompt) {
		let prompt = customPrompt;

		if (appendSection) {
			prompt += appendSection;
		}

		// Append project context files
		if (contextFiles.length > 0) {
			prompt += "\n\n<project_context>\n\n";
			prompt += "Project-specific instructions and guidelines:\n\n";
			for (const { path: filePath, content } of contextFiles) {
				prompt += `<project_instructions path="${filePath}">\n${content}\n</project_instructions>\n\n`;
			}
			prompt += "</project_context>\n";
		}

		// Append skills section (only if read tool is available)
		const customPromptHasRead = !selectedTools || selectedTools.includes("read");
		if (customPromptHasRead && skills.length > 0) {
			prompt += formatSkillsForPrompt(skills);
		}

		prompt += `\nCurrent working directory: ${promptCwd}`;

		return prompt;
	}

	// Get absolute paths to documentation and examples
	const readmePath = getReadmePath(); // 获取到readme.md的绝对路径
	const docsPath = getDocsPath(); // 获取到docs的绝对路径
	const examplesPath = getExamplesPath(); // 获取到examples的绝对路径

	// 使用tool的snippets构建toolsList
	const tools = selectedTools || ["read", "bash", "edit", "write"];
	const visibleTools = tools.filter((name) => !!toolSnippets?.[name]);
	const toolsList =
		visibleTools.length > 0 ? visibleTools.map((name) => `- ${name}: ${toolSnippets![name]}`).join("\n") : "(none)";

	// 构建guideline
	const guidelinesList: string[] = [];
	const guidelinesSet = new Set<string>();
	const addGuideline = (guideline: string): void => {
		if (guidelinesSet.has(guideline)) {
			return;
		}
		guidelinesSet.add(guideline);
		guidelinesList.push(guideline);
	};

	const hasBash = tools.includes("bash");
	const hasGrep = tools.includes("grep");
	const hasFind = tools.includes("find");
	const hasLs = tools.includes("ls");
	const hasRead = tools.includes("read");

	// 如果tool有bash，但是没有grep、find、ls，则添加guideline
	// 让ai使用bash去操作文件
	if (hasBash && !hasGrep && !hasFind && !hasLs) {
		addGuideline("Use bash for file operations like ls, rg, find");
	}

	// 增加原始tools的guideline
	for (const guideline of promptGuidelines ?? []) {
		const normalized = guideline.trim();
		if (normalized.length > 0) {
			addGuideline(normalized);
		}
	}

	// Always include these
	addGuideline("Be concise in your responses");
	addGuideline("Show file paths clearly when working with files");

	const guidelines = guidelinesList.map((g) => `- ${g}`).join("\n");

	let prompt = `You are an expert coding assistant operating inside pi, a coding agent harness. You help users by reading files, executing commands, editing code, and writing new files.

Available tools:
${toolsList}

In addition to the tools above, you may have access to other custom tools depending on the project.

Guidelines:
${guidelines}

Pi documentation (read only when the user asks about pi itself, its SDK, extensions, themes, skills, or TUI):
- Main documentation: ${readmePath}
- Additional docs: ${docsPath}
- Examples: ${examplesPath} (extensions, custom tools, SDK)
- When reading pi docs or examples, resolve docs/... under Additional docs and examples/... under Examples, not the current working directory
- When asked about: extensions (docs/extensions.md, examples/extensions/), themes (docs/themes.md), skills (docs/skills.md), prompt templates (docs/prompt-templates.md), TUI components (docs/tui.md), keybindings (docs/keybindings.md), SDK integrations (docs/sdk.md), custom providers (docs/custom-provider.md), adding models (docs/models.md), pi packages (docs/packages.md), environment variables (docs/environment-variables.md)
- When working on pi topics, read the docs and examples, and follow .md cross-references before implementing
- Always read pi .md files completely and follow links to related docs (e.g., tui.md for TUI API details)`;

	if (appendSection) {
		prompt += appendSection;
	}

	// Append project context files
	if (contextFiles.length > 0) {
		prompt += "\n\n<project_context>\n\n";
		prompt += "Project-specific instructions and guidelines:\n\n";
		for (const { path: filePath, content } of contextFiles) {
			prompt += `<project_instructions path="${filePath}">\n${content}\n</project_instructions>\n\n`;
		}
		prompt += "</project_context>\n";
	}

	// Append skills section (only if read tool is available)
	if (hasRead && skills.length > 0) {
		prompt += formatSkillsForPrompt(skills);
	}

	prompt += `\nCurrent working directory: ${promptCwd}`;

	return prompt;
}
```
这里是系统默认的提示此会把toolsList和guidelines带进去

## 模型如何调用tool？&& 调用tool后的结果如何塞会context？
这个我们知道是基于模型的返回信息去调用本地的工具的，我们看看这个具体的流程
这个其实之前在看agent内核的时候已经仔细看过了，这里我们再把代码贴出来看看怎么个事
```ts
async function runLoop(
	initialContext: AgentContext,
	newMessages: AgentMessage[],
	initialConfig: AgentLoopConfig,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
	streamFunction: StreamFn,
): Promise<void> {
	let currentContext = initialContext;
	//....

	// 从大模型的返回message过滤出toolCall messgae
	const toolCalls = message.content.filter((c) => c.type === "toolCall");

	const toolResults: ToolResultMessage[] = [];
	hasMoreToolCalls = false;
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

		// 加入到message的上下文中
		for (const result of toolResults) {
			currentContext.messages.push(result);
			newMessages.push(result);
		}
	}
}
```
但是这里我们还需要仔细看看toolResults的格式
```ts
export interface ToolResultMessage<TDetails = any> {
	role: "toolResult";
	toolCallId: string;
	toolName: string;
	content: (TextContent | ImageContent)[]; // Supports text and images
	details?: TDetails;
	/** Usage from the tool execution itself, if available. Not part of main LLM context accounting. */
	usage?: Usage;
	/**
	 * Names from `Context.tools` that became available after this result.
	 * Providers with native deferred tool loading use this as the load point;
	 * other providers ignore it and use `Context.tools` normally.
	 */
	addedToolNames?: string[];
	isError: boolean;
	timestamp: number; // Unix timestamp in milliseconds
}
```

整体而言，tool的逻辑非常简单，可是作为一个工程落地，需要考虑如何构建整个tool模块，让其更好的镶嵌到项目中去，还是需要考虑很多东西的