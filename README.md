# DNAHarness-Skills

Portable [Agent Skills](https://opencode.ai/docs/skills/) (`SKILL.md` — YAML frontmatter +
markdown procedure) for the Forward Biolabs DNA harness. Skills are the orchestration layer:
they sequence calls to the tools/MCP server in
[DNAHarness-MCP](https://github.com/tejvirmann/DNAHarness-MCP) into the multi-step workflows a
scientist actually asks for, so a bare tool-calling loop doesn't skip or misorder steps.

Each `SKILL.md` uses the standard frontmatter shape (`name`, `description`) that opencode,
Claude Code, and other `SKILL.md`-compatible agent clients all discover the same way — nothing
in this repo is opencode-specific.

## Use with opencode

Point `skills.paths` at this repo's `skills/` folder in the harness project's `opencode.jsonc`:

```jsonc
{
  "skills": {
    "paths": ["/absolute/path/to/DNAHarness-Skills/skills"]
  }
}
```

Verify discovery with `opencode debug skill` — look for the skill's `name` with a `location`
pointing into this repo.

## Use with Claude Code / other `SKILL.md`-compatible clients

Symlink or copy the skill folder(s) you want into whichever discovery path your client scans
(e.g. Claude Code: `.claude/skills/<name>/SKILL.md` at the project or user level). The
`SKILL.md` files themselves don't need to change — only where they're placed.

## Skills

| Skill | Status | Composes (tools from DNAHarness-MCP) | Serves |
|---|---|---|---|
| [`golden-gate-assembly-planning`](skills/golden-gate-assembly-planning/SKILL.md) | **Built** | `parse_sequence_file` → `optimize_overhangs`/`find_split_positions` → `assess_internal_sites` → `find_codon_substitutions` → `design_primer_pair` → `check_primer_specificity` → `find_compatible_buffers` → `build_assembly_file` | Acceptance-test prompts 1 & 2: "assemble gene x into vector y," "create a golden gate plan for my construct" |
| `genbank-file-io` | Half done (read only) | `parse_sequence_file` (built); edit half needs a new `edit_genbank_file` tool | Reading/understanding GenBank files — read half doesn't need a skill, tool access is enough; edit half will |
| `genomic-database-lookup` | Not started — blocked on adopting a DB MCP | NCBI Datasets/Entrez MCP (or Claude Science's Genomes connector) | Public genomic database access |
| `mutagenesis-primer-library` | Not started — blocked on new tool | New `design_mutagenesis_primer_library` tool | Acceptance-test prompt 3: "create a library of mutagenesis primers for my enzyme/protein" |
| `benchling-sync` | Not started — blocked on wiring the Benchling MCP | Benchling native MCP (read) + new write-back tool | Push/pull DNA sequences to/from Benchling projects and LIMS |
| `sequencing-verification` | Not started — blocked on a new `sequencing-mcp` (largest single build in the roadmap) | New `sequencing-mcp` + `build_assembly_file` output as reference | Analyze Sanger/.ab1, Nanopore, Illumina, PacBio data against an expected construct |
| `twist-order` | Not started — blocked on a new `twist-mcp` | Manufacturability check → `twist-mcp` feasibility/quote/order, mandatory user-confirmation gate | Create and place synthetic DNA orders via Twist TAPI |
| `strain-engineering-check` | Not started — blocked on Evo2 access path | `evo2-mcp` or Claude Science's Evo 2 skill | Use Evo2 to check/improve strain engineering |
| `visual-reporting` | Not started (long-term) | New `construct-visualizer` / `chart-generator` tools | Wrap other skills' output with a plasmid map or chart when the result is visual in nature |

Full sourcing, priority order, and the explicitly-deprioritized backlog (rich DNA-representation
tracking, experiment-database analysis, CRISPR guide design, gene circuit builder) live in
[DNAHarness-MCP/research/skills.md](https://github.com/tejvirmann/DNAHarness-MCP/blob/main/research/skills.md)
and `research/plan-long-term.md` in that repo.

## Repo layout

```
skills/
  golden-gate-assembly-planning/
    SKILL.md
```
