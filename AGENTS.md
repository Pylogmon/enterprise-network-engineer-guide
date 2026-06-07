# Repository Guidelines

## Project Structure & Module Organization

This repository contains a Markdown-based enterprise network engineering study guide. The main content lives under `enterprise-network-engineer-guide/`.

- `enterprise-network-engineer-guide/README.md`: entry point and reading guidance.
- `enterprise-network-engineer-guide/DIRECTORY.md`: standalone table of contents.
- `enterprise-network-engineer-guide/WRITING_STATUS.md`: writing progress and next planned chapters.
- `enterprise-network-engineer-guide/chapters/`: numbered chapter files, e.g. `06-switching-basics.md`.
- `enterprise-network-engineer-guide/appendices/`: appendix materials and quick references.
- `enterprise-network-engineer-guide/templates/`: reusable writing templates.

Keep each chapter in its own file. Update `DIRECTORY.md` whenever a new chapter becomes available.

## Build, Test, and Development Commands

There is no build system or application runtime. Use shell checks for document maintenance:

```sh
find enterprise-network-engineer-guide -maxdepth 3 -type f | sort
wc -l enterprise-network-engineer-guide/chapters/*.md
rg "TODO|待补充|FIXME" enterprise-network-engineer-guide
```

Use `find` to verify structure, `wc -l` to estimate chapter size, and `rg` to locate unfinished sections.

## Coding Style & Naming Conventions

Write documentation in Markdown using concise headings and clear technical explanations. Prefer ASCII punctuation in filenames and paths. Chapter filenames should use this pattern:

```text
NN-english-topic.md
```

Examples: `07-vlan.md`, `10-layer3-switching.md`. Keep titles in Chinese to match the guide, and keep cross-references as relative Markdown links.

## Chapter Writing Standard

Write all new and expanded chapters to the same level of detail as the current first four chapters. The target reader is a zero-foundation learner, so do not treat important concepts as obvious.

Each chapter should generally include:

- A clear `本章学习目标` section near the beginning.
- Step-by-step concept explanations before introducing commands or advanced design.
- Enterprise scenarios that explain where the technology appears in real networks.
- Tables for comparisons, planning values, address ranges, protocol roles, or troubleshooting matrices.
- Concrete examples with internally consistent VLAN IDs, IP ranges, gateways, DNS, DHCP, routing, and security policy assumptions.
- Verification and troubleshooting sections that teach how to narrow the fault boundary.
- A short self-check list or exercises when the topic involves calculation, design judgment, or operational workflow.
- A concise `本章小结` that connects the topic to later chapters.

Avoid one-sentence explanations for core topics. For example, when introducing LAN, WAN, VLAN, routing, NAT, DNS, DHCP, STP, OSPF, or firewall policy, explain the definition, why it exists, common enterprise usage, typical topology, related terms, and common failure symptoms.

Use subchapters freely when they make the material easier for beginners to follow. Prefer a tutorial-book flow:

```text
概念 -> 为什么需要 -> 工作原理 -> 企业场景 -> 示例 -> 验证 -> 常见故障 -> 小结
```

Use Mermaid diagrams when topology, packet flow, layering, DHCP exchange, routing path, redundancy, or troubleshooting flow would be clearer visually. Keep Mermaid diagrams simple enough to render in common Markdown viewers, and prefer quoted labels when labels include HTML line breaks or special characters.

When expanding existing chapters, preserve the established style and chapter intent, but deepen the explanation rather than only adding lists. Do not rewrite unrelated completed chapters wholesale just to match wording.

## Testing Guidelines

No automated tests are configured. Before submitting changes, manually verify:

- Markdown links point to existing files.
- `DIRECTORY.md` reflects new or renamed chapters.
- Technical examples are internally consistent, especially VLAN IDs, IP ranges, gateways, and routing examples.
- New chapters follow the structure in `templates/chapter-template.md`.
- New or expanded chapters meet the chapter writing standard above and are understandable to zero-foundation readers.
- Mermaid code fences, if used, are balanced and use syntax that common Markdown renderers can parse.

## Commit & Pull Request Guidelines

No Git history is available in this workspace. Use clear, imperative commit messages such as:

```text
Add VLAN troubleshooting chapter
Update routing section directory links
```

Pull requests should describe the changed chapters, list any new files, and call out technical assumptions or examples that need review. For large additions, include a short checklist confirming directory links and writing status were updated.

## Agent-Specific Instructions

Keep edits scoped to documentation unless explicitly asked otherwise. Do not rewrite completed chapters wholesale when adding new content. Preserve the existing tutorial-book style: concept, principle, enterprise scenario, example, verification, troubleshooting, and summary. For future chapter work, treat chapters 1-4 as the quality and depth benchmark.
