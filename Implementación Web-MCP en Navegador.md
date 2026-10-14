# **WebMCP Integration and Architecture Skills Protocol for the Webtyp Framework**

## **Architectural Paradigm and Framework Objectives**

The Web Model Context Protocol (WebMCP) represents a fundamental shift in how browser-native AI agents interact with web applications. Developed within the W3C Web Machine Learning Community Group, WebMCP transitions AI agents away from non-deterministic interactions, such as visual pixel parsing and DOM scraping, by allowing websites to expose structured, callable tools directly through the browser1. By relying on explicit tool contracts defined via JSON Schema rather than heuristic visual analysis, WebMCP yields a task accuracy rate of approximately 98% and reduces inference token consumption by up to 89%2.  
Unlike Anthropic's backend-focused Model Context Protocol (MCP), which requires separate server infrastructure, API key management, and JSON-RPC transports, WebMCP operates entirely on the client side1. It executes within the user's active browsing context, securely inheriting the existing session state, cookies, and authentication layers1. For the development of the "webtyp" framework, implementing WebMCP enables the seamless creation of agent-ready web applications without duplicating backend logic. This document outlines the technical specifications, namespace evolution, integration patterns, and security constraints required to align the framework with the WebMCP standard.

## **Namespace Evolution and Polyfill Integration**

The WebMCP specification is currently an experimental draft undergoing rapid iteration. Initial implementations, including the Chrome 146 early preview, exposed the API via navigator.modelContext. However, subsequent revisions to the W3C draft and Chrome 150 origin trials deprecated this namespace in favor of document.modelContext6. Frameworks must implement robust feature detection to bridge these versions, as relying solely on the older namespace results in silent registration failures in newer browser builds6.  
Because native support is currently restricted to experimental browser flags and origin trials, production-grade frameworks must implement a polyfill strategy. The @mcp-b/global and @mcp-b/webmcp-polyfill packages inject document.modelContext into environments lacking native support, bridging the WebMCP interface to the underlying agent transports9. The framework initialization logic should detect native support and seamlessly fall back to these polyfills to guarantee uninterrupted agent compatibility across all browsers.

## **The Imperative API: Dynamic JavaScript Tool Registration**

The imperative API affords programmatic control for Single Page Applications (SPAs) and highly dynamic framework components. By invoking document.modelContext.registerTool(), the framework can expose existing application logic as agent-callable tools11.

### **Structural Interfaces and Type Definitions**

To ensure strict adherence to the WebMCP standard, the framework's internal architecture must map to specific JSON Schema requirements. The core configuration objects determine how the agent interprets and invokes the tool14.

| Interface Property | Data Type | Requirement | Technical Description |
| :---- | :---- | :---- | :---- |
| name | String | Required | A unique identifier for the tool per document context, formatted typically in snake\_case (e.g., update\_preferences). |
| description | String | Required | Natural language instructions detailing the tool's purpose, usage conditions, and precise behavioral outcomes to guide the LLM. |
| inputSchema | Object | Optional | A JSON Schema (draft 2020-12) definition of the required parameters and types. |
| annotations | Object | Optional | Security and execution hints defining the tool's operational risks, including properties like readOnlyHint and consequentialHint. |
| execute | Function | Required | An asynchronous callback invoked by the agent. Accepts an arguments object and a context object containing an AbortSignal. |

### **Lifecycle Management and Execution Handling**

In dynamic frameworks, tools must not outlive the UI components they represent. WebMCP handles tool lifecycles through the AbortSignal primitive. The framework must instantiate an AbortController when a component mounts, pass its signal to registerTool(), and invoke abort() during the component's teardown phase11. This guarantees that agents cannot invoke tools tied to unmounted views.  
Execution handlers must prioritize structured error reporting over raw exceptions. If a tool encounters a validation failure or semantic error, throwing an unhandled exception will crash the execution context. Instead, the framework must normalize errors by returning a standard WebMCP result object containing the isError: true flag alongside descriptive text10. This allows the agent to read the error context and self-correct its parameters for a subsequent attempt.  
For high-stakes or irreversible operations—such as financial transactions or account deletion—the execution handler must enforce a human-in-the-loop workflow. The framework can invoke client.requestUserInteraction() within the execute block, pausing the agent's process to surface a confirmation dialog to the active user5. The agent execution only resumes once explicit human consent is captured.

## **The Declarative API: HTML Form Annotations**

For server-rendered views or static interfaces, WebMCP provides a Declarative API that transforms standard HTML forms into agent-callable tools via specific DOM attributes13. The framework should map internal form components to these data attributes to automate tool discovery without requiring client-side JavaScript execution for registration.  
Forms must include both toolname and tooldescription attributes to be successfully indexed by the browser's model context18. The toolautosubmit boolean attribute controls the execution flow; its presence permits the agent to dispatch the form immediately upon filling it, while its absence forces the browser to pause, visually populating the fields and awaiting human submission18. Individual input elements dictate the generated JSON Schema properties, which can be semantically overridden using toolparamtitle and toolparamdescription18.  
When a declarative form is submitted, the framework's event delegation system must differentiate between human users and AI agents. The browser injects a SubmitEvent.agentInvoked boolean property into the event payload18. If this value is true, the framework must intercept the standard navigation behavior using e.preventDefault() and supply a structured JSON response back to the model via the SubmitEvent.respondWith(Promise) method18. This pattern ensures the agent receives machine-readable data while human users continue to see standard HTML responses.

## **Security Constraints and Privilege Boundaries**

WebMCP establishes a rigorous security perimeter mediated entirely by the browser, operating under the principle that the agent inherits only the active user's current session privileges5.

### **Risk Annotations and Prompt Injection Defense**

The framework must intelligently apply WebMCP's annotation hints to safeguard the agent's reasoning engine. The readOnlyHint: true flag should be automatically appended to query operations, informing the agent that the tool is safe to execute without triggering side effects25. Conversely, consequentialHint: true marks state-mutating actions, signaling to the browser and agent that explicit confirmation boundaries must be respected25. To mitigate prompt injection attacks, any tool returning user-generated text, scraped HTML, or unverified database strings must include the untrustedContentHint: true annotation, isolating the payload from the agent's core instruction set25.

### **Cross-Origin Isolation and Permissions Policies**

By default, WebMCP restricts tool registration and discovery to the top-level document and same-origin iframes28. If the framework utilizes cross-origin iframes for widget isolation or micro-frontends, it must explicitly navigate the Permissions-Policy architecture. The parent document must grant access by embedding the iframe with the allow="tools" attribute11. Simultaneously, the child iframe must explicitly declare which parent origins are permitted to discover its tools by supplying the exposedTo array in the registerTool() options29. The browser enforces this double-gated opt-in mechanism to prevent malicious third-party scripts from hijacking the agent's context.

## **Integration Directives for the Webtyp Framework**

To synthesize these protocols into the architecture of the "webtyp" framework, the resulting API design must prioritize progressive enhancement and lifecycle safety. The framework must never assume the presence of document.modelContext; all WebMCP logic must gracefully degrade, ensuring that human interfaces remain fully operational on unsupported browsers15.  
State synchronization forms the backbone of stable agent interactions. The framework's internal routing and component unmounting processes must automatically trigger the abort() method on registered tools, preventing the agent from hallucinating interactions with deprecated DOM nodes11. Furthermore, the framework's schema definition layer—whether relying on TypeScript interfaces or property types—should automatically compile into valid draft 2020-12 JSON Schemas for tool registration, abstracting the boilerplate away from the end developer. Finally, for debugging and automated testing within the framework ecosystem, developers should leverage navigator.modelContextTesting and the Model Context Tool Inspector extension, which allow for headless tool inspection, execution simulation, and schema validation without invoking a live LLM instance11.

#### **Obras citadas**

> 1. WebMCP: Making Your Web App Agent-Ready \- Towards AI, [https\://pub.towardsai.net/webmcp-making-your-web-app-agent-ready-d7d4d9cb790d](https://pub.towardsai.net/webmcp-making-your-web-app-agent-ready-d7d4d9cb790d)  
> 2. WebMCP: Basic Info \+ Hands-On Guide | Digital Loop, [https\://digital-loop.com/en/blog/webmcp-google-and-microsoft-agentic-website](https://digital-loop.com/en/blog/webmcp-google-and-microsoft-agentic-website)  
> 3. Why Browser Agents Waste 89% of Their Tokens | Chanl Blog, [https\://chanl.ai/blog/webmcp-every-website-tool-ai-agents](https://chanl.ai/blog/webmcp-every-website-tool-ai-agents)  
> 4. WebMCP Technical Notes \- GitHub Pages, [https\://w3c-cg.github.io/aikr/webMCP/webmcp-technical-notes.html](https://w3c-cg.github.io/aikr/webMCP/webmcp-technical-notes.html)  
> 5. WebMCP Explained: What It Is and How It Works \- browsers.to, [https\://browsers.to/articles/webmcp-explained](https://browsers.to/articles/webmcp-explained)  
> 6. WebMCP: when your website hands tools to the agent, [https\://www\.neuralpartners.ai/resources/webmcp-explained/](https://www.neuralpartners.ai/resources/webmcp-explained/)  
> 7. What Is WebMCP? Your Website's API for AI Agents \- No Hacks, [https\://nohacks.co/blog/what-is-webmcp](https://nohacks.co/blog/what-is-webmcp)  
> 8. From DOM Scraping to Contracts: Understanding WebMCP (Part 1), [https\://daily.dev/posts/from-dom-scraping-to-contracts-understanding-webmcp-part-1--nevl1du6v](https://daily.dev/posts/from-dom-scraping-to-contracts-understanding-webmcp-part-1--nevl1du6v)  
> 9. @mcp-b/webmcp-polyfill \- npm, [https\://www\.npmjs.com/package/@mcp-b/webmcp-polyfill](https://www.npmjs.com/package/@mcp-b/webmcp-polyfill)  
> 10. @mcp-b/global \- npm, [https\://www\.npmjs.com/package/@mcp-b/global](https://www.npmjs.com/package/@mcp-b/global)  
> 11. WebMCP: Qué es y cómo convierte tu web en agéntica, [https\://www\.webreactiva.com/blog/webmcp](https://www.webreactiva.com/blog/webmcp)  
> 12. Google WebMCP Explained: The Complete Guide to Web Model, [https\://discoveredlabs.com/blog/google-webmcp-explained-the-complete-guide-to-web-model-context-protocol-for-ai-agents](https://discoveredlabs.com/blog/google-webmcp-explained-the-complete-guide-to-web-model-context-protocol-for-ai-agents)  
> 13. A Guide to Web MCP: How Websites Can Expose Tools to AI Agents, [https\://composio.dev/content/a-guide-to-web-mcp-how-websites-can-expose-tools-to-ai-agents](https://composio.dev/content/a-guide-to-web-mcp-how-websites-can-expose-tools-to-ai-agents)  
> 14. WebMCP is new browser-native execution model for AI Agents : r/mcp, [https\://www\.reddit.com/r/mcp/comments/1rdcvkd/webmcp\_is\_new\_browsernative\_execution\_model\_for/](https://www.reddit.com/r/mcp/comments/1rdcvkd/webmcp_is_new_browsernative_execution_model_for/)  
> 15. WebMCP Cheat Sheet \- W3C Browser AI Tool API Quick Reference, [https\://www\.webfuse.com/webmcp-cheat-sheet](https://www.webfuse.com/webmcp-cheat-sheet)  
> 16. Error Handling in MCP Servers: isError vs JSON-RPC ... \- AgentCat, [https\://agentcat.com/guides/error-handling-custom-mcp-servers/](https://agentcat.com/guides/error-handling-custom-mcp-servers/)  
> 17. Official WebScraping AI MCP Server, [https\://mcpservers.org/servers/webscraping-ai/webscraping-ai-mcp-server](https://mcpservers.org/servers/webscraping-ai/webscraping-ai-mcp-server)  
> 18. WebMCP Declarative API: HTML Form Tools \- AgentCat, [https\://agentcat.com/guides/declarative-webmcp-tools-html-form-attributes/](https://agentcat.com/guides/declarative-webmcp-tools-html-form-attributes/)  
> 19. What is WebMCP and how to use it \- Blog ﹤ ﹥ Codely, [https\://codely.com/en/blog/what-is-webmcp-and-how-to-use-it](https://codely.com/en/blog/what-is-webmcp-and-how-to-use-it)  
> 20. WebMCP Explained: How Chrome Turns Websites Into MCP Tools, [https\://www\.alekseialeinikov.com/en/blog/topics/ai/webmcp-explained-chrome-ai-agents-mcp-tools](https://www.alekseialeinikov.com/en/blog/topics/ai/webmcp-explained-chrome-ai-agents-mcp-tools)  
> 21. Chrome's WebMCP Early Preview: the end of “AI agents clicking, [https\://dev.to/axrisi/chromes-webmcp-early-preview-the-end-of-ai-agents-clicking-buttons-b6e](https://dev.to/axrisi/chromes-webmcp-early-preview-the-end-of-ai-agents-clicking-buttons-b6e)  
> 22. Chrome's WebMCP makes AI agents stop pretending \- AI @ Sulat.com, [https\://ai.sulat.com/chromes-webmcp-makes-ai-agents-stop-pretending-e8c7da1ba650](https://ai.sulat.com/chromes-webmcp-makes-ai-agents-stop-pretending-e8c7da1ba650)  
> 23. WebMCP Explained: Make Your Website Usable by AI Agents (2026), [https\://letstartdesign.com/webmcp-explained-how-to-make-your-website-usable-by-ai-agents](https://letstartdesign.com/webmcp-explained-how-to-make-your-website-usable-by-ai-agents)  
> 24. What Is WebMCP? When to Expose Browser Tools to AI Agents, [https\://ekline.io/blog/what-is-webmcp-browser-tools-ai-agents](https://ekline.io/blog/what-is-webmcp-browser-tools-ai-agents)  
> 25. WebMCP: Give AI agents a real API for your Angular app, [https\://dev.to/marianocodes/webmcp-give-ai-agents-a-real-api-for-your-angular-app-ngo](https://dev.to/marianocodes/webmcp-give-ai-agents-a-real-api-for-your-angular-app-ngo)  
> 26. Before You Add WebMCP: the Security Problem Nobody's Selling You, [https\://blog.agentailor.com/posts/before-you-add-webmcp](https://blog.agentailor.com/posts/before-you-add-webmcp)  
> 27. Make Your Site Usable by AI Agents with WebMCP | Dennis Morello, [https\://morello.dev/blog/webmcp-making-your-site-usable-by-ai-agents](https://morello.dev/blog/webmcp-making-your-site-usable-by-ai-agents)  
> 28. webmachinelearning/webmcp \- GitHub, [https\://github.com/webmachinelearning/webmcp](https://github.com/webmachinelearning/webmcp)  
> 29. README.md \- webmachinelearning/webmcp \- GitHub, [https\://github.com/webmachinelearning/webmcp/blob/main/README.md](https://github.com/webmachinelearning/webmcp/blob/main/README.md)  
> 30. WebMCP | AI in Chrome \- Chrome for Developers, [https\://developer.chrome.com/docs/ai/webmcp](https://developer.chrome.com/docs/ai/webmcp)  
> 31. WebMCP \- Web Machine Learning, [https\://webmachinelearning.github.io/webmcp/](https://webmachinelearning.github.io/webmcp/)  
> 32. Transports and bridges \- WebMCP documentation and developer tools, [https\://docs.mcp-b.ai/explanation/architecture/transports-and-bridges](https://docs.mcp-b.ai/explanation/architecture/transports-and-bridges)  
> 33. WebMCP agent tools \- Meta Wearables Developer Center, [https\://wearables.developer.meta.com/docs/develop/webapps/agent-tools/](https://wearables.developer.meta.com/docs/develop/webapps/agent-tools/)