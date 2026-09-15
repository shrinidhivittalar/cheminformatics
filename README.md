# Chemistry DB Intern Task — SMILES to IUPAC Conversion

Converts `input.csv` (compound_id, smiles) into `output.csv` with canonical SMILES,
IUPAC name, molecular formula, exact mass, InChIKey, status, and notes.

## Tools / libraries used

- **RDKit** — all local cheminformatics: SMILES parsing & validation, canonical
  SMILES, molecular formula, exact (monoisotopic) mass, InChIKey, and detecting
  multi-component structures. Everything except naming is computed offline,
  deterministically, from the input alone.
- **requests** — the one external HTTP call (PubChem lookup).

## API / data source used

**PubChem PUG REST**, queried by InChIKey:
`https://pubchem.ncbi.nlm.nih.gov/rest/pug/compound/inchikey/{inchikey}/property/IUPACName/JSON`

This is a *lookup* against PubChem's existing `IUPACName` property, not a name
generator. If PubChem confirms no compound matches the InChIKey, no name is
fabricated — nothing is guessed or filled in.

A short delay is added between retry attempts on the same request; this is a
politeness measure, not an enforced rate limit, and is not relied on for
correctness.

## Why RDKit can't produce the IUPAC name itself

RDKit (and open-source cheminformatics generally) has no reliable systematic
IUPAC nomenclature engine. Anything that "generates" a name locally would be
a heuristic guess, not a systematic/preferred name — which the task explicitly
rules out ("do not invent a name"). PubChem is used only as an external lookup
for the name; if no matching PubChem record is found, the name is left blank.

## How failures are handled

- **Empty/whitespace-only input**: rejected before RDKit ever sees it. This is
  handled as a special case because `Chem.MolFromSmiles("")` does not return
  `None` — it returns a technically-valid empty molecule (0 atoms), which
  would otherwise slip through as a spurious "compound" with `exact_mass 0.0`.
  Status `INVALID_SMILES`.
- **Syntax errors** (unbalanced rings/branches, bad tokens, e.g. `CMP-014`):
  RDKit's own parser diagnostic is captured and placed in `notes` — not a
  fixed, guessed explanation. Status `INVALID_SMILES`.
- **Chemically invalid but syntactically parseable** (e.g. a valence
  violation): caught via a separate sanitization step so RDKit's specific
  exception message (e.g. *"Explicit valence for atom # 0 C, 5, is greater
  than permitted"*) ends up in `notes`. Status `INVALID_SMILES`.
- **InChIKey generation failure** on an otherwise-valid molecule (rare): the
  exception is not silently discarded — its message is captured and appended
  to `notes` as *"InChIKey generation failed: ..."*. No InChIKey is invented,
  and no PubChem lookup is attempted without one (status becomes
  `LOOKUP_ERROR`, since a lookup can't proceed with nothing to look up).
- **PubChem request/network failures** (timeout, connection error, DNS
  failure): caught, and the loop continues to the next compound.
- **Unexpected PubChem HTTP status** (anything other than 200 or 404): caught,
  and the loop continues.
- **Malformed or non-JSON PubChem response body** (a 200 response whose body
  isn't valid JSON): caught, and the loop continues.
- **Unexpected JSON structure** (a 200, valid-JSON response missing the
  expected `PropertyTable`/`Properties`/`IUPACName` fields — e.g. a
  different response shape than the API normally returns): caught, and the
  loop continues.

All of the above are reported as status `LOOKUP_ERROR`, **not** `NOT_FOUND` —
a failed or uninterpretable lookup means we don't know whether a record
exists, which is a different state from PubChem affirmatively confirming
there isn't one. No exception from a PubChem request or from parsing its
response is allowed to propagate out of the lookup step and stop the batch;
every one of the cases above results in a normal `(None, "ERROR", note)`
result, not a crash, and processing moves on to the next compound.

## How missing results are handled

If PubChem is successfully queried and confirms no match for a valid
structure's InChIKey, `iupac_name` is left blank and status is `NOT_FOUND`.
If the lookup itself could not be completed, status is `LOOKUP_ERROR` instead
(see above) — the two are never conflated.

## Multi-component handling

Salts/mixtures (SMILES containing `.`, e.g. `CMP-013`) are preserved as a single
structure — not split into fragments. Local properties (formula, mass, InChIKey)
are computed for the whole structure, and an IUPAC name lookup is still
attempted. Status stays `MULTI_COMPONENT` regardless of the lookup outcome —
including if the lookup fails — because the structural property the task asks
to have surfaced is that the record contains multiple components.

If PubChem *does* return a name for a multi-component InChIKey, that name is
still recorded in `iupac_name` as-is (nothing invented), but `notes` explicitly
flags that a name returned for a disconnected/multi-component structure may
not represent the entire combined structure and should be reviewed — it is
not assumed to name the whole record.

## Duplicate handling (CMP-015)

PubChem lookups are cached in-memory by InChIKey for the duration of a run.
`CMP-015` has the same SMILES as `CMP-001`, so it produces the same InChIKey and
reuses the cached result (including a cached error, if that's what occurred) —
no second API call is made. `notes` says so explicitly on the reused row.

## Status values

| Status | Meaning |
|---|---|
| `SUCCESS` | Valid, single-component structure; IUPAC name found |
| `INVALID_SMILES` | Empty input, or RDKit could not parse/sanitize the input |
| `MULTI_COMPONENT` | Valid structure contains multiple components (e.g. a salt) |
| `NOT_FOUND` | PubChem lookup completed and confirmed no matching name |
| `LOOKUP_ERROR` | The PubChem lookup could not be completed (network/API failure) — existence of a name is unknown |

## Setup & run instructions

1. Open `chemistry_db_intern_task.ipynb` in Google Colab or Jupyter.
2. Run all cells top to bottom (Cell 1 installs `rdkit` and `requests`).
3. `output.csv` is (re)generated from `input.csv` — no manual edits are made to
   any row after generation. Delete `output.csv` and re-run the notebook at any
   time to regenerate it from scratch.
