# MicroBridge: project guide for Claude

Small portfolio demo (Azure integration events) built to prepare for a Senior Full Stack (.NET / API / Azure) contract interview. Owner: Buddhika (senior .NET dev). Keep explanations short and simple.

## Where to look first

- Spec (source of truth): `Docs/mini-spec.md`. If code and spec disagree, stop and ask.

## Stack and decisions

- .NET 10, Azure Functions isolated worker, Flex Consumption plan with 512 memory and 0 Always Ready instances. 
- Azure Service Bus: reuse the existing DevPulse Standard namespace, with a new `integration-events` topic.
- Everything else in a new `MicroBridge` resource group, Australia East, in the DevPulse subscription: Function apps, Logic App (Consumption), storage, Application Insights (with a daily cap), Cosmos DB (serverless).
- Cost is the main constraint. Nothing always-on. Resources are deleted when the demo is done.
- Infrastructure as code with Bicep. Do not create, change or delete Azure resources unless asked.

## Code standards

- DTOs and message contracts are C# records. Validate input and return a clear 400 for bad input.
- Do not catch and hide exceptions. Log errors; let failures reach retry and the dead-letter queue.
- Service Bus consumers: auto-complete off, complete only after the work succeeds, idempotent (upsert by `messageId`, ignore older `occurredAtUtc`).
- Times in UTC. Configurable values (keys, names, limits) come from settings, never hardcoded.
- Do not add NuGet packages without asking; say which package and why.
- Small diffs. Do not refactor unrelated code. Do not rename files without asking.

## Testing

- xUnit v3, FluentAssertions 7.2.2 (never upgrade to 8 or later, it is paid), Moq 4.20.72 (not 4.20.0 or 4.20.1).
- Keep it small: a couple of tests per piece, happy path and the main error paths only. No comprehensive suites.
- Mock only boundaries (Service Bus sender, Cosmos client, clock). Pure logic (envelope builder, payload validation) is tested directly.
- Claude Code writes the FAILING tests first. Buddhika writes the implementation himself. Then Claude reviews it with a mark out of 10 (good, tighten, correct).
- Run `dotnet build` and `dotnet test` before saying you are done.

## Working rules for Claude

- **Stop and ask, never guess.** If information is missing, a command fails, the spec is unclear or anything is ambiguous, stop and ask Buddhika. Never work around a problem silently.
- **Check the branch before writing tests or files.** Run `git rev-parse --abbrev-ref HEAD`. If it is `main`, stop and suggest a feature branch name (`feature/<short-name>`); Buddhika creates it.
- **Check dependencies before using them.** Before using Azure resources, Docker or any local emulator, confirm they exist and are running. If not, stop and ask Buddhika. Never start or create them yourself.
- **Do not write production code** unless asked.
- **Git is Buddhika's.** Claude may only read git state (`status`, `diff`, `log`). Never run `add`, `commit`, `push`, `checkout`, `merge`, `reset`, `tag`, or use `gh`.
- **Secrets:** never read, print or commit keys, connection strings, `.env` files or user secrets.
- End each task with: what changed, how it was tested, anything you were unsure about.

## Commit style

Conventional Commits: `type(scope): subject`, for example `feat(intake): add webhook endpoint`. Types: feat, fix, docs, test, refactor, chore, build, ci, perf, style. Never add a "Generated with Claude Code" footer or a claude.ai session link to commits or PR text.
