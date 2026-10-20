# Spyglass for Claude

The official [Spyglass](https://spyglass.so) plugin for Claude. Search brands and creators across Meta, Instagram, and TikTok; inspect paid ad libraries and viral organic creatives; cluster hooks, personas, angles, and campaigns; analyze destination landing pages and creator collaborations; fetch public post comments; and index missing accounts on demand.

## Install

Once the directory listing is approved, install **Spyglass** from Claude's plugin directory, then authenticate when Claude opens the Spyglass OAuth flow. No API key needs to be copied into Claude.

For local development:

Clone this repository using GitHub’s normal repository workflow. Then validate the local folder with `claude plugin validate --strict ./claude-plugin` and load it with `claude --plugin-dir ./claude-plugin`.

In Claude Code, open `/mcp`, approve the `spyglass` server, and complete the Spyglass OAuth flow in your browser. Then run one of the prompts below. Claude should call a `spyglass` MCP tool and return competitive creative intelligence. Loading the plugin with `--plugin-dir` keeps the test isolated to that session.

## Try it

Ask Claude:

```text
Use Spyglass to find AG1's longest-running active Meta video ads and break down their top hooks and core angles.
```

```text
Use Spyglass to discover the top DTC electrolyte and hydration powder brands, then compare their creative hooks over the last 30 days.
```

```text
Use Spyglass to find female K-beauty skincare creators in the US with 20k–150k followers ranked by highest engagement.
```

## Included

- The production Spyglass MCP server at `https://mcp.spyglass.so/mcp`
- OAuth 2.1 authentication through Spyglass
- Two specialized skills (`spyglass-accounts` and `spyglass-media`) that guide account resolution, on-demand indexing (`addAccounts` + `getAccountsIndexStatus`), semantic vs. brand-library media retrieval, taxonomy filtering, and performance sorting
- Brand & creator discovery, paid Meta ad & organic Instagram/TikTok creative search, creative insights (hooks, personas, angles, media mix), campaigns & offers, creator collaborations, landing pages, and public post comments

## Links

- [Spyglass documentation](https://docs.spyglass.so)
- [Spyglass MCP guide](https://docs.spyglass.so/documentation/get-started/mcp)
- [Spyglass API reference](https://docs.spyglass.so/llms.txt)
- [Support](mailto:support@spyglass.so)

## License

MIT — see [LICENSE](LICENSE).
