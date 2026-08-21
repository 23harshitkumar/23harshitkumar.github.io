---
layout: post
title: "GSoC 2026: Template Logic Support for Playground"
date: 2026-08-21 00:00:00 +0530
categories: update final
---

<div style="text-align: center; margin: 40px 0;">
  <img src="/assets/images/gsoc-banner.png" alt="Accord Project & GSoC 2026 Banner" style="width: 100%; max-width: 500px; height: auto; border-radius: 8px;">
</div>

Author: [Harshit Kumar](https://github.com/23harshitkumar), a contributor to [Google Summer of Code 2026](https://summerofcode.withgoogle.com/programs/2026/projects/TODO_PROJECT_ID).

### Table of Contents

1. [About the Project](#about-the-project)
2. [Problem Statement](#problem-statement)
3. [What Was Built](#what-was-built)
4. [Future Direction](#future-direction)
5. [Learnings](#learnings)
6. [Experience](#experience)

<br>

***

### About the Project

Accord Project templates are composed of three fundamental layers: a **Concerto** data model, a **TemplateMark** natural language template, and **TypeScript logic** that dictates how the contract responds to transactions. While the [Template Playground](https://playground.accordproject.org) has historically supported the first two layers, it lacked native support for executing contract logic. This project bridges that gap by introducing full Template Logic support directly into the playground, empowering users to author, compile, and execute contract logic entirely within the browser.

### Problem Statement

Previously, the absence of in-browser logic execution meant users were forced to rely on local CLI tools or external backend services to test their contracts. This disconnect fragmented the developer experience, introducing unnecessary friction and making it challenging for newcomers to visualize the complete lifecycle of a template. My objective was to eliminate this friction by building a cohesive, end-to-end client-side pipeline featuring a dedicated TypeScript editor, an in-browser compilation step, and a secure, sandboxed runtime environment.

### What Was Built

#### Pipeline: TypeScript Compilation

The [`compileLogic()`](https://github.com/accordproject/template-playground/blob/main/src/store/store.ts) method in the Zustand store uses `TemplateArchiveProcessor.compileLogic()` from `@accordproject/template-engine` to compile the user's TypeScript into JavaScript. The compiled output is then post-processed to make it evaluable via `new Function()`:

```js
// Strip export keywords so new Function() can evaluate the code
code = code.replace(/^export\s+class/gm, "class");
code = code.replace(/^export\s+default/gm, "");

// Confirm a TemplateLogic subclass exists and append a return statement
const match = code.match(/class\s+(\w+)\s+extends\s+TemplateLogic/);
code += `\nreturn ${match[1]};\n`;
```

If compilation fails, diagnostic errors with line numbers are mapped back to the Monaco editor for inline display.

#### Pipeline: Sandboxed Execution

The core design challenge was running arbitrary user code safely without freezing the UI. The solution uses a three-layer isolation boundary:

**React UI → Sandboxed iframe → Blob Web Worker**

[`executeInSandbox(code, method, args)`](https://github.com/accordproject/template-playground/blob/main/src/store/store.ts) generates a monotonic `executionId`, registers a resolver callback in a module-scoped [`sandboxResolvers`](https://github.com/accordproject/template-playground/blob/main/src/store/sandboxResolvers.ts) Map, and posts a JSON message to the iframe via `postMessage`. The [`SandboxFrame`](https://github.com/accordproject/template-playground/blob/main/src/components/SandboxFrame.tsx) component renders a hidden `<iframe sandbox="allow-scripts">` (no `allow-same-origin`, so the browser assigns it a null origin). Inside, [`logic-handler.html`](https://github.com/accordproject/template-playground/blob/main/public/logic-handler.html) spawns a Blob Web Worker per execution with a 5-second kill-switch via `Worker.terminate()`. Even an infinite loop in user code cannot freeze the browser.

#### Pipeline: Contract State Lifecycle

Two store methods compose the full lifecycle:

- [`initContract()`](https://github.com/accordproject/template-playground/blob/main/src/store/store.ts) calls `executeInSandbox(compiledLogicJs, 'init', [parsedData])` and stores the returned state.
- [`triggerContract()`](https://github.com/accordproject/template-playground/blob/main/src/store/store.ts) passes data, a request payload, and the current state to the sandbox's `trigger` method, then extracts the result, updated state, and emitted events (such as `PaymentObligation`) for rendering:

```js
const { result, state, emit } = await executeInSandbox(
  compiledLogicJs,
  'trigger',
  [parsedData, requestPayload, currentState]
);
```

#### Supporting Features

Beyond the core pipeline, several features were built to complete the experience:

- **Adaptive Layout**: the panel layout detects whether a template contains logic and automatically shows or hides the Logic editor and Contract Runner.
- **Shareable Links**: logic-enabled templates can be shared via URL with the logic code, request payload, and state encoded.
- **Sample Templates**: logic-enabled samples (Late Payment, Service Agreement, Counter) ship out-of-the-box so users can explore immediately.
- **Guided Tour**: an interactive Shepherd.js walkthrough introduces the logic workflow to new users.
- **Obligations Tracker**: a visual component renders `PaymentObligation` events emitted by contract logic.

### Future Direction

Future contributors can extend this work by:

- **User Experience & Intuitiveness**: streamlining the contract execution lifecycle to make the playground more intuitive for newcomers and more efficient for power users.
- **Error Reporting**: improving execution error messages to be more human-readable, prioritising root-cause errors first, and providing clearer diagnostics across the compile → init → trigger lifecycle.
- **Template Library Integration**: improving integration so users can dynamically load and execute any sample from the Accord Project Template Library (Cicero) directly within the playground.

### Learnings

This project was a massive learning opportunity that went beyond just writing code. I gained deep experience in building secure, cross-origin `postMessage` bridges to safely execute untrusted code, managing complex async state with Zustand, and integrating a TypeScript compilation toolchain into a browser build. More importantly, I learned the rigors of writing production-level code, anticipating edge cases, ensuring robust error handling, and writing software that is secure and maintainable. The weekly Technology Working Group calls also gave me incredible insight into making high-level architectural decisions in an open-source setting.

Beyond the technical skills, this experience significantly improved my professional communication. Working closely with mentors across different time zones taught me how to articulate technical challenges clearly, ask the right questions, and collaborate effectively within a professional, globally distributed team.

### Experience

I owe a massive debt of gratitude to the entire Accord Project community for fostering such a welcoming environment. 

To my mentors, Matt Roberts, Dan Selman, Sanket Shevkar, and Priyanshu Singh, thank you for your patience, meticulous code reviews, and for pointing me in the right architectural direction. A specific shoutout goes to Sanket for our regular syncs that helped maintain the project's momentum.

I also want to specifically acknowledge Diana Lease for her incredible availability. Her timely heads-ups, constant support, and willingness to help navigate organizational hurdles made a massive difference during development.

Lastly, I am deeply thankful to Google for hosting the Google Summer of Code program. It has been a transformative summer, and I am excited to stick around and continue building within the Accord Project ecosystem!
