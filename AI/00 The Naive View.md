# 00 — The Naive View

> My side of the story first. What I learnt, what I did, what broke. The research comes after.

---

## The task

Was given a task to build a chat agent for BMS using local LLMs.

## What I knew

I had worked with LLM APIs before, hitting an endpoint like `/chat/completions`, and had done tool calling through it. That's all. Beyond calling the endpoint, I didn't know shit about what the flow was from there, or the inner workings. All I knew was they predict the next token for the given prompt, until they choose there's nothing more to produce or hit some stop signal.

Didn't know what an inference engine was. Didn't even know what the word "inference" meant when people used it in videos.

```
what I thought it was:

  my code ──► /chat/completions ──► the model
```

## What was already there

There was already an inference engine installed on the hardware I was working on. Didn't know that. There was already a model on it too. Didn't know that either.

All I "inferred" from the requirement was: connect to that model, build an agent.

## Harness

I knew what a harness was, but I thought it was only for coding agents. Didn't know it could be used for other stuff as well. It's just the plumbing around the LLM call. So all the context management, tool calls and business rules are the code that surrounds the LLM call.

## Built a subpar agent, then the issues started

Built a subpar agent with help from Claude and then started facing issues.

- **Tool calls were prompted, not native.** That's how I learnt there are two versions. Prompted is when there are no tools in the chat template, so you send the tools as part of the prompt from the harness. The harness has to do the job of the inference engine. And "chat template" was a new word. Didn't know what that was, so started looking into what it meant.
- **Chat template and the endpoint.** Got to know about the chat template and the endpoint the inference engine provides. That endpoint is the bridge we use to talk to the model.
- **Structured output wasn't there.** Learnt about that. First thought the model had to support it, but it's mostly the inference engine's job. The engine takes the schema from the request and while generating, only lets the model pick tokens that keep the output valid. The model just has to be decent at JSON, otherwise you get valid but dumb output.
- **Inference engine bugs.** A lot of them surfaced. The issue was I didn't know if they were inference engine bugs, my code issues or the model's. I had no prior knowledge of this.

So did more research to understand how things actually work.

## It's the combination

Got to know that it's the combination of inference engine + model that has to support your use case. Ours was:

- native tool calling
- structured output
- skills, so prompt caching was needed if latency had to be less
- hardware that supports all of the above for that combination

```
what it actually is:

  harness ──► endpoint ──► inference engine ──► model
                                 │
                             hardware
```

## Fixed our end first

Before changing models, I built out the harness with skills support. Whatever the model was going to be, at least our side of the code had to be clean.

The rule from the start: the harness stays model-agnostic. Swap the model or the inference engine, and the code shouldn't have to bend around it. Same with evals, they test the agent, not a specific model. If we ever moved to cloud models, the harness had to handle that too.

Streaming was part of it as well.

## Testing, the shit part of the journey

- **Speed.** The old combination generated tokens painfully slow. That's what pushed me into time to first token, prefill and decode. The new combination was way better.
- **Laptop testing.** Also ran things locally on an M4 Mac with a small Gemma 4 model.
- **Evals.** Learnt you need something like golden tests, a fixed set to check every change against.
- **Prompt versioning.** Every prompt going to the LLM is basically a template literal, and each one is a knob you'll end up tuning.
- **Knobs everywhere.** Harness changes, prompt changes, engine settings. You have to know which knobs exist before you can turn any of them.
- **Model quirks.** Even after all of this, the latest model started looping over tools.
- **Latency.** Still something to keep chasing.
- **Internals.** Got into how an LLM works inside. Not full depth, but more than surface.
- **Decisions doc.** Wrote down the decisions made in an md, for the agent and for whoever reads it next.

## Looking back

I initially thought hit the endpoint and that itself is the full model. I didn't know jack, because I'd never run one locally.

Even if the model is the same, if the inference engine changes there's a lot of difference. Platforms too. So many things.
