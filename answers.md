# Written Answers

## 1. What do `@`, `[N+]`, `[O-]`, and `.` mean in SMILES?

- `@` → represents **stereochemistry**. It tells us about the 3D arrangement of atoms around a chiral atom. `@` and `@@` represent different configurations, but they don't directly mean R and S.
- `[N+]` → means the nitrogen has a **formal positive charge of +1**.
- `[O-]` → means the oxygen has a **formal negative charge of -1**.
- `.` → means there are **separate molecular components** which are not covalently bonded.

For example, CMP-013 is `[Na+].[O-]C(=O)c1ccccc1`. Here, `[Na+]` is the sodium ion and `[O-]...` is the negatively charged benzoate part. The `.` shows that they are separate components.

These symbols are important because they preserve information about the actual chemical structure. If we lose the charge, stereochemistry, or component information, we may not be representing the same molecule correctly.

---

## 2. Names of CMP-003, CMP-007 and CMP-011

**CMP-003:** `CC(=O)Oc1ccccc1C(=O)O`  
→ **2-acetoxybenzoic acid** (commonly aspirin)

It has a benzene ring with a carboxylic acid group and an acetoxy group next to it.

**CMP-007:** `O=[N+]([O-])c1ccc(Cl)cc1`  
→ **1-chloro-4-nitrobenzene**

It has a chlorine and a nitro group attached to the benzene ring in the para position.

**CMP-011:** `C[C@H](O)C(=O)O`  
→ **(2S)-2-hydroxypropanoic acid** (S-lactic acid)

Here the `@` is important because the molecule has a chiral carbon, so the stereochemistry needs to be included in the name.

---

## 3. What if the database gives a common name instead of an IUPAC name?

I would **not treat the common name as an IUPAC name** or try to guess the systematic name.

I would store them separately, for example:

```text
iupac_name  → systematic name, if available
common_name → name returned by the database
name_source → PubChem
