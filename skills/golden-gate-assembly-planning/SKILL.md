---
name: golden-gate-assembly-planning
description: Plan a Golden Gate (Type IIS) DNA assembly end-to-end - from raw sequences or files through to a downloadable GenBank construct. Use when the user asks to "assemble gene X into vector Y," "create a Golden Gate plan for my construct," design GGA primers, pick assembly overhangs, or check a sequence for internal Type IIS sites before cloning. Requires the dnakit-mcp MCP server (parse_sequence_file, assess_internal_sites, find_codon_substitutions, find_split_positions, optimize_overhangs, score_overhang_set, design_primer_pair, check_primer_specificity, find_compatible_buffers, build_assembly_file).
---

# Golden Gate Assembly Planning

Sequences a chain of `dnakit-mcp` tool calls into a complete Golden Gate assembly plan. A bare
tool-calling loop tends to skip steps or call them out of order (e.g. designing primers before
resolving an internal-site conflict, or forgetting to check primer specificity) - follow this
procedure instead of improvising the order.

## Which entry point applies

There are two shapes of request. Identify which one you're in before doing anything else.

**Entry A - "assemble gene X into vector Y."** Two named parts, no fragment split needed. The
assembly has exactly as many fragments as the user gives you (usually 2: insert + backbone,
sometimes more if multiple parts are being combined in one pot). You are choosing brand-new
junction overhangs, not reading them off an existing sequence.

**Entry B - "create a Golden Gate plan for my construct" (user supplies one long designed
sequence).** One long sequence that needs to be *split* into synthesizable/orderable fragments.
Junction overhangs are determined by where you cut the sequence, not freely chosen.

If it's ambiguous which shape applies, ask the user rather than guessing - the two shapes call
different tools (`optimize_overhangs` vs. `find_split_positions`) and produce different plans.

## Step-by-step procedure

### 0. Get sequences in hand

If the user references a file (GenBank/FASTA/SnapGene) rather than pasting raw sequence, call
`parse_sequence_file` first. Don't ask the user to paste a sequence that's already sitting in a
file on disk.

If the user hasn't named an enzyme, default to `BsaI-HFv2` (the standard MoClo/Golden Gate Type
IIS enzyme) and say you're defaulting to it. Confirm instead of defaulting if the user has clearly
implied a specific enzyme system (e.g. mentions BsmBI/Esp3I or a specific MoClo level).

### 1. Determine fragments and overhangs

- **Entry A (assemble X into Y):** call `optimize_overhangs` with `n` = number of fragments
  (2 for a simple insert-into-vector; circular topology). This gives you a high-fidelity overhang
  set from Potapov ligation data - don't hand-pick overhangs yourself.
- **Entry B (split one long sequence):** call `find_split_positions` on the full sequence. This
  returns fragment boundaries *and* their overhangs together - the split and the overhang choice
  are not separate decisions here.

### 2. Check for internal site conflicts

Call `assess_internal_sites` on every fragment (pass the chosen `enzyme_name`, and
`assembly_overhangs` from step 1 if you have them, so overhang conflicts are flagged too). A
fragment with an internal site of the same enzyme will get cut in the wrong place during assembly.

If a flagged site falls inside a coding region (CDS), call `find_codon_substitutions` to remove it
via a silent (synonymous) mutation - this preserves the protein sequence while eliminating the
site. If it falls outside any CDS (e.g. in a spacer or UTR), tell the user directly: they may need
to either accept two-step assembly for that fragment or manually edit the region, since silent
substitution doesn't apply to non-coding sequence.

If internal sites can't be removed, say so explicitly and switch the plan to two-step assembly
rather than silently producing a plan that will fail in one pot.

### 3. Design primers per fragment

Call `design_primer_pair` once per fragment, using that fragment's assigned overhangs from step 1
and the chosen enzyme. This is a per-fragment call, not a batch call - do it once for each
fragment in the assembly.

### 4. Validate specificity

Call `check_primer_specificity` with *all* designed primers against *all* template sequences at
once (not one at a time) - mispriming is a property of the whole primer set against the whole
template pool, not a single primer in isolation. If a primer flags a low-Tm off-target binding
site, redesign that primer (adjust `fwd_start`/`rev_start` or `target_tm` in `design_primer_pair`)
before moving on.

### 5. Buffer check (multi-enzyme reactions only)

If more than one enzyme is involved (e.g. combining the Golden Gate enzyme with a separate
diagnostic digest), call `find_compatible_buffers`. Skip this step for a single-enzyme GGA - it's
not needed there.

### 6. Build the final construct file

Call `build_assembly_file` last, once the fragment order, full sequences, and trims are settled.
Pay close attention to the tool's own coordinate rules (they're easy to get subtly wrong):

- Each fragment's `sequence` must be the **complete** fragment (hundreds-thousands of bp), never
  just the 4nt overhang.
- Every fragment after the first needs `trim_start` set to the overhang length, so the shared
  junction isn't duplicated in the output.
- The backbone fragment must be linearized starting/ending at the insertion site, not at position
  0 of the original plasmid - think of it as cutting the circular plasmid open at the insertion
  point.
- Feature coordinates are relative to each fragment's own (possibly reordered) sequence, not the
  original plasmid.

Hand the resulting GenBank file back to the user as the deliverable.

## What "done" looks like

A complete run of this skill ends with: a resolved fragment/overhang set, zero unresolved internal
site conflicts (or an explicit two-step-assembly call-out), primers with a clean specificity
check, and a `build_assembly_file` GenBank output. If any of those didn't happen, the plan isn't
finished yet - don't stop early just because a plausible-looking answer appeared after a couple of
tool calls.
