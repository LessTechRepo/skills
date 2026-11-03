# Less Skills

This repository packages the `less` plugin for Codex, Claude Code, and Claude Desktop. It adds agent skills on top of the Less MCP server so agents know *when and how* to combine the server's tools, and how to write custom Python scripts on Less.

## Features

- **Discover**: Find sources, models, tables, orchestrations, and destinations you can access.
- **Analyze**: Answer data questions with aggregate SQL over verified tables.
- **Understand pipelines**: Trace upstream provenance and downstream impact, walk a model step by step, and find where a metric is computed.
- **Operate**: Check job health, diagnose failures from logs, review schedules, and refresh an asset when you explicitly ask.
- **Write Python scripts**: Write, test, and deploy custom Python scripts that read and write Less tables and call external APIs, using the `less-sdk` package.



## Installation

### Codex

```
codex plugin marketplace add <org>/skills
codex plugin add less@less
```



### Claude Code

```
claude plugin marketplace add <org>/skills
claude plugin install less@less
```



### Claude Desktop

Install the Less plugin from **Settings > Plugins**.

### Skills Only

```
npx skills add <org>/skills
```

Skills-only installs do not configure the MCP server; add it to your client yourself (see below).

## Connect and Authenticate

The plugin connects to the Less MCP server over HTTP:

```
https://app.less.tech/api/mcp
```

`app.less.tech` is the default Less workspace host. If your workspace uses a different host, replace it in `less/.mcp.json` (Codex, Claude Code, Claude Desktop) or in your user-level MCP config. Re-apply the edit after upgrading the plugin, since an upgrade replaces the bundled file.

See the [Less documentation](https://docs.less.tech) for more about the platform and its [Terms of Use](https://www.less.tech/terms-of-use).

Every call runs as the signed-in user and is subject to that user's workspace permissions. Tools only see assets the user may read, and `executeAsset` only runs assets the user may execute. Authentication is handled by your client's MCP connection; this repository does not contain or issue tokens.

## Usage

The plugin contains two skills: `less-workspace`, covering discovery, analysis, pipelines, and operations, and `less-python-sdk`, covering custom Python scripts. Agents select them automatically. Example prompts:

```
What tables in my workspace hold monthly order history?
```

```
Which five customers had the highest revenue last quarter?
```

```
What feeds the revenue_daily model, and what breaks if I change it?
```

```
Where in our models do we compute gross margin?
```

```
What failed in the last 7 days, and why?
```

```
Refresh the orders source.
```

```
Write a Less Python script that pulls orders from our billing API into a table.
```

Invoke a skill explicitly with `/less:less-workspace <request>` in Claude Code, or `$less:less-workspace <request>` in Codex.

## Safety

The MCP is read-mostly. `executeAsset` is the only tool that changes anything: it starts a job. The skill only calls it when you explicitly ask to run, refresh, or re-execute an asset, and only after checking `canExecute` (and `hasPublishedVersion` for models).

These skills do not bypass Less authentication, authorization, or permission checks.

## License

The materials in this repository (skill text, manifests, and configuration) are licensed under the Apache License, Version 2.0. See `LICENSE`.

The license applies only to the materials in this repository. It does not grant rights to the Less product or hosted service, its APIs, customer data, or Less trademarks and branding.

## Layout

```
.claude-plugin/marketplace.json     Claude marketplace
.agents/plugins/marketplace.json    Codex marketplace
less/
  .claude-plugin/plugin.json
  .codex-plugin/plugin.json
  .mcp.json                         MCP server (Codex, Claude)
  mcp_config.json                   MCP server (user-level config format)
  skills/less-workspace/
    SKILL.md                        Triggers, workflows, rules
    references/                     analysis, pipelines, operations
  skills/less-python-sdk/
    SKILL.md                        Triggers, workflow, rules
    references/                     variables, tables, runtime
```

