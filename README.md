# Neoantigen Target Discovery in Multiple Myeloma

A exploratory bioinformatics pipeline for identifying candidate neoantigen 
targets for cancer vaccine development, built as a proof-of-concept using 
publicly available data.

---

## Why I Built This

Multiple Myeloma is almost always preceded by a pre-cancerous state called 
MGUS (Monoclonal Gammopathy of Undetermined Significance), which is detectable 
in blood tests years before cancer develops. The idea behind a pre-cancer 
vaccine is simple: if you can identify the molecular fingerprints of abnormal 
cells early, you can train the immune system to recognize and eliminate them 
before full disease develops.

This pipeline is my exploration of the computational side of that problem. 
Specifically: given somatic mutation data from myeloma patients, which 
mutations produce peptides that the immune system could potentially recognize?

This is not a production pipeline. It is a learning project that demonstrates 
the core architecture of neoantigen target discovery and my ability to move 
from biological question to working code quickly.

---

## The Biological Question

When a cell acquires a somatic mutation, it changes the protein sequences 
inside that cell. The cell's machinery constantly chops proteins into small 
fragments called peptides and displays them on its surface via MHC-I molecules. 
This is how the immune system monitors cells for abnormalities.

If a mutant peptide binds to an MHC-I molecule and gets displayed on the cell 
surface, a T-cell could potentially recognize it as foreign and kill the cell. 
This is the basis of neoantigen-based cancer vaccines.

The computational question is: which somatic mutations produce peptides that 
actually bind to MHC-I molecules well enough to be displayed?

---

## The Data

**MMRF CoMMpass Study** (Multiple Myeloma Research Foundation)

- 1,091 newly diagnosed Multiple Myeloma patients
- Whole exome sequencing of tumor and matched normal tissue
- Open access masked somatic mutation MAF files downloaded from GDC portal
- Project ID: MMRF-COMMPASS

This is one of the largest and most well-characterized myeloma genomic 
datasets in the world.

---

## What the Pipeline Does

### Step 1: Load and filter somatic mutations
All 1,091 MAF files are loaded and merged. Variants are filtered to keep 
only missense mutations: single amino acid substitutions that change the 
protein sequence. Silent, intronic, UTR, and frameshift variants are excluded 
at this stage.

**Result**: 51,005 missense mutations across 1,091 patients.

### Step 2: Remove immunoglobulin gene artifacts
Plasma cells (the cell type that becomes malignant in myeloma) naturally 
undergo somatic hypermutation in immunoglobulin genes as part of normal 
B-cell biology. This process deliberately introduces mutations into IG genes 
to improve antibody affinity. These mutations are not tumor drivers and are 
removed before analysis.

**Result**: 2,935 IG gene variants removed. 48,070 remaining.

### Step 3: Extract mutant peptide sequences
For each missense mutation, the mutant amino acid is extracted from the 
HGVSp_Short notation (e.g. p.S251L means Serine at position 251 changed 
to Leucine). A synthetic 9-mer peptide is constructed centered on the mutant 
residue.

**Note**: This is the key limitation of this proof-of-concept. A production 
pipeline would fetch the true protein sequence from UniProt or Ensembl and 
extract real flanking context. The synthetic Alanine flanks used here do not 
reflect true biological sequence and therefore limit the utility of binding 
predictions.

### Step 4: MHC-I binding prediction with netMHCpan 4.2
Each peptide is run through netMHCpan 4.2c, a neural network trained on 
mass spectrometry data of peptides eluted from real MHC molecules. Predictions 
are made against HLA-A*02:01, the most common HLA allele in European 
populations (~45% frequency).

**Key output metric**: %Rank_EL (Eluted Ligand Rank)
- < 0.5%: Strong Binder — high priority candidate
- 0.5–2%: Weak Binder — moderate priority
- > 2%: Non-binder

### Step 5: Recurrence analysis
Mutations are grouped by gene and amino acid change to identify those 
appearing in multiple patients. Recurrent mutations are stronger vaccine 
candidates because they suggest clonal selection and functional relevance 
to tumor growth.

---

## Key Findings

After filtering, 3,157 mutations are recurrent across more than one patient.

### RAS/MAPK pathway dominance
The most recurrent mutations cluster in a single oncogenic signaling pathway:

| Gene | Mutation | Patients |
|------|----------|----------|
| NRAS | p.Q61R | 74 |
| KRAS | p.Q61H | 73 |
| NRAS | p.Q61K | 57 |
| BRAF | p.V640E | 34 |
| KRAS | p.G13D | 33 |
| KRAS | p.G12D | 31 |

NRAS, KRAS, and BRAF are sequential nodes in the RAS/MAPK cascade. 
Activating mutations here constitutively drive cell proliferation. Their 
dominance in this dataset is consistent with published myeloma genomics 
literature, which validates the pipeline output.

### Myeloma-specific drivers

**IRF4 p.K123R** (12 patients): A gain-of-function hotspot specific to 
myeloma. IRF4 is a transcription factor essential for plasma cell survival. 
Myeloma cells are often dependent on IRF4 activity for survival.

**DIS3 p.D488N** (11 patients) and **p.R780K** (8 patients): DIS3 is an 
RNA exonuclease among the most frequently mutated genes in myeloma 
specifically. Its myeloma enrichment makes it a disease-specific target.

---

## What the Results Mean for Vaccine Design

Recurrent mutations in functionally important genes are priority candidates 
because:

1. They are likely clonal (present in all tumor cells, not just a subset)
2. They are under positive selection (the tumor needs them to survive)
3. They are shared across patients (a single vaccine formulation could 
   cover multiple patients)

The RAS/MAPK hotspot mutations (NRAS Q61, KRAS G12/G13/Q61, BRAF V600E) 
are particularly attractive because they are well-validated oncogenic drivers 
with existing clinical evidence of functional importance.

---

## Limitations

**Synthetic peptides**: The current implementation uses Alanine as flanking 
context. HLA-A*02:01 has strict anchor residue preferences at positions 2 
and 9. Alanine flanks do not satisfy these requirements, which is why no 
strong binders were identified in the binding prediction step. True protein 
context from Ensembl or UniProt would resolve this.

**Single HLA allele**: Only HLA-A*02:01 is tested. A real pipeline requires 
patient-specific HLA typing and predictions across HLA-A, B, and C alleles.

**No expression filter**: Mutations in unexpressed genes cannot produce 
peptides. RNA-seq integration would filter for expressed mutations only.

**No clonality filter**: Variant allele frequency (VAF) is not used to 
filter for clonal mutations. Subclonal mutations present in only a fraction 
of tumor cells are weaker vaccine targets.

**MHC binding is necessary but not sufficient**: A peptide that binds MHC 
must also be recognized by a T-cell receptor (TCR) to activate an immune 
response. TCR recognition is not predicted here.

---

## Next Steps for a Production Pipeline

1. Replace synthetic peptides with true protein context via Ensembl REST API
2. Add patient-specific HLA typing from WXS data using OptiType or HLA-HD
3. Filter for expressed mutations using CoMMpass RNA-seq data
4. Filter for clonal mutations using VAF thresholds
5. Run predictions across HLA-A, B, C for each patient
6. Add proteasomal cleavage prediction with NetChop
7. Cross-reference against self-proteome to exclude peptides similar to 
   normal human proteins

---

## Tools and Data

- **netMHCpan 4.2c** (DTU Health Tech) — MHC-I binding prediction
- **MMRF CoMMpass** open access MAF files via GDC portal
- Python: pandas, subprocess, pathlib
- Jupyter notebook

---

## Repository Structure
neoantigen-target-discovery/
├── notebooks/
│   └── myeloma_neoantigen_discovery.ipynb
├── tools/
│   └── netMHCpan-4.2/          # not tracked in git
├── data/
│   ├── raw_mafs/                # not tracked in git
│   └── processed/               # not tracked in git
└── README.md

---

*Built as a proof-of-concept to demonstrate pipeline architecture for 
neoantigen target discovery. Data is publicly available through GDC. 
netMHCpan requires a free academic license from DTU Health Tech.*
