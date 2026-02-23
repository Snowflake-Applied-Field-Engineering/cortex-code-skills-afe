# Cortex Code Skills - Snowflake Applied Field Engineering

A collection of skills developed by members of the Snowflake Applied Field Engineering team for [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code), Snowflake's AI coding assistant.

[Skills](https://docs.snowflake.com/en/user-guide/cortex-code/extensibility#skills) are structured workflows that extend Cortex Code's capabilities with domain-specific guidance, decision trees, SQL templates, and reference material.

## Available Skills

### UDF Debugging (`udf-debugging/`)

Debug Snowpark User-Defined Functions using Snowflake Event Tables. This skill provides guided workflows for:

- Setting up event tables and configuring telemetry levels (LOG_LEVEL, TRACE_LEVEL, METRIC_LEVEL)
- Querying UDF logs, metrics, and traces from event tables
- Analyzing common Snowpark error patterns and suggesting fixes
- Adding logging and OpenTelemetry instrumentation to Python UDFs

The skill routes between two sub-workflows based on user intent:

| Sub-Skill | Purpose |
|-----------|---------|
| `setup/` | Event table detection and telemetry configuration |
| `debug/` | Log querying, error analysis, and fix suggestions |

Reference documents in `references/` cover event table schema, observability parameters, error patterns, and Python logging examples.

## Skill Structure

Each skill follows a standard layout:

```tree
<skill-name>/
  SKILL.md              # Entry point: intent detection, routing, key concepts
  <sub-skill>/
    SKILL.md            # Sub-workflow with step-by-step guidance
  references/
    *.md                # Supporting reference material
```

`SKILL.md` files use YAML frontmatter (`name`, `description`, `parent_skill`) and define structured workflows with mandatory stopping points where user confirmation is required before executing mutations.

## Contributing

To add a new skill, create a directory at the repository root with a `SKILL.md` entry point following the structure above. Skills should:

- Detect user intent and route to focused sub-skills
- Include stopping points before any mutations (ALTER, CREATE, DROP statements)
- Provide reference documents for complex domain knowledge
- Use YAML frontmatter for metadata

## License

Apache License 2.0 -- see [LICENSE.txt](LICENSE.txt) for details.

## Support

This repository and its contents are NOT supported by Snowflake.

The resources, scripts, and documentation provided here are created and maintained independently. They are intended for internal use, educational purposes, or to streamline specific workflow tasks at your own risk.

For questions, issues, or contributions, please open an issue on GitHub. Support is provided on a best-effort basis, and Snowflake Support is unable to assist with content in this repository.