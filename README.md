# COP1–ETV4 AlphaFold/Docking Workflow

## Project overview

This project focuses on the predicted COP1–ETV4 protein complex and an FDA-drug docking screen against COP1. The main goals were:

- visualize the AlphaFold-predicted COP1–ETV4 interaction;
- identify a COP1 pocket/hotspot for docking;
- test GNINA docking on a small FDA-drug pilot set;
- scale the workflow to the full 2,162-compound FDA library;
- create a poster-ready figure showing the COP1–ETV4 complex and predicted interface.

Generative AI assistance was used during this project. OpenAI Codex was used to help streamline command-line workflow development, troubleshoot DCC/GNINA/PyMOL issues, and organize analysis steps. Scientific decisions, data interpretation, and final review were performed by the researcher.

Most computational work was performed on Duke DCC in:

```bash
/hpc/group/woodlab/Michael/colabfold_test/cop1_etv4/docking/fda_screen
```

The local visualization/output files were saved on the Mac under:

```bash
~/Desktop/COP1_docking
```

and local slide/README outputs are in:

```bash
/Users/mariahdrexler/Documents/Codex/2026-06-22/pic/outputs
```

---

## 1. Starting files and model

The workflow used an AlphaFold/ColabFold COP1–ETV4 multimer prediction.

Key files included:

- `cop1_receptor.pdb` — COP1 receptor structure used for docking
- `fda_10.sdf` — initial 10-compound FDA pilot set
- `e-Drug3D_2162.sdf` — full FDA-like 2,162-compound ligand library

In PyMOL, the COP1–ETV4 model included objects such as:

- `fold_cop1_etv4_multimer_model_0_A_A`
- `fold_cop1_etv4_multimer_model_0_B_B`
- `COP1`
- `dock_center`
- `hotspot_center`

The two AlphaFold chains were later copied into shorter PyMOL aliases:

```pymol
create cop1_chain, fold_cop1_etv4_multimer_model_0_A_A
create etv4_chain, fold_cop1_etv4_multimer_model_0_B_B
```

---

## 2. Initial GNINA docking pilot

The initial docking command used GNINA with COP1 as receptor and the 10-drug SDF as ligand input:

```bash
gnina -r ../../cop1_receptor.pdb \
  -l fda_10.sdf \
  --center_x 9.785 --center_y 11.044 --center_z 7.827 \
  --size_x 25 --size_y 25 --size_z 25 \
  --cnn_scoring=none \
  --exhaustiveness 4 \
  --num_modes 3 \
  -o fda_10_slurm.sdf
```

GNINA produced output, but it became clear that using one multi-molecule SDF directly was not ideal for this workflow. Only the first molecule was effectively being handled the way we wanted.

The pilot library was therefore split into individual SDF files.

---

## 3. Splitting the 10-drug pilot SDF

Open Babel was not available on the login node, so the SDF was split using `awk`:

```bash
mkdir -p fda_10_split

awk '{
  file=sprintf("fda_10_split/drug_%02d.sdf", n+1)
  print >> file
}
(/^\$\$\$\$$/) {
  close(file)
  n++
}' fda_10.sdf
```

This created:

```text
fda_10_split/drug_01.sdf
fda_10_split/drug_02.sdf
...
fda_10_split/drug_10.sdf
```

---

## 4. Pilot Slurm array

A Slurm array was created to dock each pilot ligand separately:

```bash
#!/bin/bash
#SBATCH --job-name=gnina_pilot
#SBATCH --partition=gpu-common
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=01:00:00
#SBATCH --array=1-10%2
#SBATCH --output=pilot_logs/gnina_%A_%a.out

module load GNINA/1.0

i=$(printf "%02d" "$SLURM_ARRAY_TASK_ID")

gnina \
  -r ../../cop1_receptor.pdb \
  -l "fda_10_split/drug_${i}.sdf" \
  --center_x 9.785 \
  --center_y 11.044 \
  --center_z 7.827 \
  --size_x 35 \
  --size_y 35 \
  --size_z 35 \
  --cnn_scoring=none \
  --exhaustiveness 4 \
  --num_modes 3 \
  -o "pilot_docked/drug_${i}_docked.sdf"
```

The pilot was submitted with:

```bash
sbatch gnina_pilot_array.sh
```

All 10 pilot jobs completed and produced 3 poses each.

---

## 5. Pilot results

The pilot scores were extracted from the GNINA logs. The top pilot hits were:

| Drug number | Compound | Best score |
| --- | --- | --- |
| `drug_06` | Desoxycorticosterone pivalate / DOC pivalate | `-29.29` |
| `drug_05` | Desoxycorticosterone acetate / DOC acetate | `-27.49` |
| `drug_10` | Ergocalciferol | `-25.02` |
| `drug_01` | Sulfapyridine | `-22.01` |

`drug_02`, triptorelin, gave a strong score but also showed `ligand outside box` warnings, so it was treated cautiously/excluded from the strongest pilot interpretation.

---

## 6. Downloading pilot outputs to Mac

The top pilot docking files and receptor were copied from DCC to the Mac.

The correct DCC host used was:

```bash
dcc-login.oit.duke.edu
```

Local destination:

```bash
~/Desktop/COP1_docking
```

Commands:

```bash
mkdir -p ~/Desktop/COP1_docking

scp mtd45@dcc-login.oit.duke.edu:/hpc/group/woodlab/Michael/colabfold_test/cop1_etv4/docking/fda_screen/pilot_docked/drug_{01,05,06,10}_docked.sdf ~/Desktop/COP1_docking/

scp mtd45@dcc-login.oit.duke.edu:/hpc/group/woodlab/Michael/colabfold_test/cop1_etv4/cop1_receptor.pdb ~/Desktop/COP1_docking/
```

Downloaded files included:

```text
cop1_receptor.pdb
drug_01_docked.sdf
drug_05_docked.sdf
drug_06_docked.sdf
drug_10_docked.sdf
```

---

## 7. PyMOL pilot visualization

The top pilot outputs were loaded into PyMOL:

```pymol
load ~/Desktop/COP1_docking/cop1_receptor.pdb, COP1
load ~/Desktop/COP1_docking/drug_01_docked.sdf, sulfapyridine
load ~/Desktop/COP1_docking/drug_05_docked.sdf, DOC_acetate
load ~/Desktop/COP1_docking/drug_06_docked.sdf, DOC_pivalate
load ~/Desktop/COP1_docking/drug_10_docked.sdf, ergocalciferol
```

Basic styling:

```pymol
hide everything
show cartoon, COP1
color gray80, COP1
show sticks, sulfapyridine DOC_acetate DOC_pivalate ergocalciferol
color cyan, sulfapyridine
color yellow, DOC_acetate
color magenta, DOC_pivalate
color orange, ergocalciferol
zoom
```

A pocket/interface region was selected around the docked ligands:

```pymol
select pocket, byres (COP1 within 5 of (sulfapyridine or DOC_acetate or DOC_pivalate or ergocalciferol))
show sticks, pocket
color gray60, pocket
show surface, pocket
set transparency, 0.55, pocket
zoom pocket, 8
```

The PyMOL session was saved as:

```text
~/Desktop/COP1_docking/COP1_pilot_analysis.pse
```

---

## 8. Receptor preparation attempts

AMBER tools were checked on DCC:

```bash
module avail 2>&1 | grep -Ei 'openbabel|pdb2pqr|reduce|amber'
module load AMBER/24-MPI_GPU

which reduce
which pdb4amber
which tleap
```

Available tools included:

```text
/opt/apps/rhel9/amber24-mpi-cuda/bin/reduce
/opt/apps/rhel9/amber24-mpi-cuda/bin/pdb4amber
/opt/apps/rhel9/amber24-mpi-cuda/bin/tleap
```

An AMBER-prepped receptor was generated:

```bash
pdb4amber \
  -i ../../cop1_receptor.pdb \
  -o ../../cop1_receptor_prepped.pdb \
  --reduce \
  --dry
```

However, GNINA 1.0 could not parse the prepped PDB:

```text
Parse error ... ATOM syntax incorrect
```

A direct `reduce` hydrogen-added receptor was also tested:

```bash
reduce -BUILD ../../cop1_receptor.pdb > ../../cop1_receptor_reduceH.pdb
```

This also caused old GNINA/Open Babel PDB parsing issues due to strict PDB column formatting. Several reformatting attempts were tested, but the practical conclusion was:

> Use the original `cop1_receptor.pdb` for GNINA because it is GNINA-readable and successfully completed the pilot docking workflow.

This should be documented as a troubleshooting note, not as a failure of the docking screen itself.

---

## 9. Full FDA library split

The full FDA-like library file was:

```text
e-Drug3D_2162.sdf
```

It was split into individual SDF files:

```bash
mkdir -p fda_split

awk '{
  file=sprintf("fda_split/drug_%04d.sdf", n+1)
  print >> file
}
(/^\$\$\$\$$/) {
  close(file)
  n++
}' e-Drug3D_2162.sdf
```

This produced:

```text
2162 individual ligand files
```

Example:

```text
fda_split/drug_0001.sdf
fda_split/drug_0002.sdf
...
fda_split/drug_2162.sdf
```

---

## 10. Full FDA GNINA screen

A full Slurm array script was created:

```bash
#!/bin/bash
#SBATCH --job-name=gnina_fda2162
#SBATCH --partition=gpu-common
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=01:00:00
#SBATCH --array=1-2162%8
#SBATCH --output=fda2162_logs/gnina_%A_%a.out

module load GNINA/1.0

i=$(printf "%04d" "$SLURM_ARRAY_TASK_ID")

gnina \
  -r ../../cop1_receptor.pdb \
  -l "fda_split/drug_${i}.sdf" \
  --center_x 9.785 \
  --center_y 11.044 \
  --center_z 7.827 \
  --size_x 35 \
  --size_y 35 \
  --size_z 35 \
  --cnn_scoring=none \
  --exhaustiveness 4 \
  --num_modes 3 \
  -o "fda2162_docked/drug_${i}_docked.sdf"
```

Output directories:

```bash
mkdir -p fda2162_logs fda2162_docked
```

Submitted job:

```bash
sbatch gnina_fda2162_array.sh
```

Job ID:

```text
48768278
```

Progress check:

```bash
squeue -j 48768278
```

Count finished docking files:

```bash
cd /hpc/group/woodlab/Michael/colabfold_test/cop1_etv4/docking/fda_screen
ls fda2162_docked/*.sdf 2>/dev/null | wc -l
```

Important note: run the count command from the docking folder, not from `~`.

---

## 11. Full-screen monitoring and partial leaderboard

Early completed jobs were checked with:

```bash
sacct -j 48768278 --format=JobID,State,ExitCode,Elapsed | head -60
```

Early completed jobs showed:

```text
COMPLETED 0:0
```

which indicates successful completion.

A partial leaderboard was generated from completed logs:

```bash
for f in fda2162_logs/gnina_48768278_*.out; do
  [ -e "$f" ] || continue
  task=$(basename "$f" .out | awk -F_ '{print $3}')
  i=$(printf "%04d" "$task")
  lig="fda_split/drug_${i}.sdf"
  name=$(awk '/> <name>/{getline; print; exit}' "$lig")
  score=$(awk '$1==1 && $2 ~ /^-/ {print $2; exit}' "$f")
  if [ -n "$score" ]; then
    printf "%s\t%s\t%s\t%s\n" "$i" "$name" "$score" "fda2162_docked/drug_${i}_docked.sdf"
  fi
done | sort -k3,3n | head -25
```

Some early scores were extremely negative, especially for large molecules such as:

- deslanoside
- digoxin
- acetyldigitoxin
- dihydroergotamine

These scores should be interpreted cautiously because Vina/GNINA scoring can favor large, bulky, contact-rich molecules.

---

## 12. Filtering outside-box warnings

Outside-box warnings were checked with:

```bash
grep -Hil 'ligand outside box' fda2162_logs/*.out 2>/dev/null | wc -l
```

Some early warning-containing ligands included:

- `drug_0070` / ergotamine
- `drug_0002` / triptorelin

A cleaner leaderboard excluding outside-box warning logs was generated:

```bash
for f in fda2162_logs/gnina_48768278_*.out; do
  [ -e "$f" ] || continue
  grep -qi 'ligand outside box' "$f" && continue

  task=$(basename "$f" .out | awk -F_ '{print $3}')
  i=$(printf "%04d" "$task")
  lig="fda_split/drug_${i}.sdf"
  name=$(awk '/> <name>/{getline; print; exit}' "$lig")
  score=$(awk '$1==1 && $2 ~ /^-/ {print $2; exit}' "$f")

  if [ -n "$score" ]; then
    printf "%s\t%s\t%s\t%s\n" "$i" "$name" "$score" "fda2162_docked/drug_${i}_docked.sdf"
  fi
done | sort -k3,3n | head -25
```

Early clean leaderboard hits included:

| Drug number | Compound | Score |
| --- | --- | --- |
| `0150` | Deslanoside | `-38.28` |
| `0152` | Digoxin | `-37.19` |
| `0158` | Acetyldigitoxin | `-35.43` |
| `0036` | Dihydroergotamine | `-33.58` |
| `0173` | Riboflavin | `-30.03` |

Interpretation note:

> The most negative docking score is not automatically the best biological candidate. Hits should be filtered by warnings, molecule size, pose quality, visual inspection, and biological relevance.

---

## 13. COP1–ETV4 poster figure generation

A figure similar to a published AlphaFold complex panel was desired:

- overview complex on the left;
- zoomed COP1–ETV4 interface on the right;
- colored chains;
- highlighted predicted interface residues.

PyMOL aliases were made:

```pymol
create cop1_chain, fold_cop1_etv4_multimer_model_0_A_A
create etv4_chain, fold_cop1_etv4_multimer_model_0_B_B
```

Interface selections were made:

```pymol
select cop1_interface, byres (cop1_chain within 4 of etv4_chain)
select etv4_interface, byres (etv4_chain within 4 of cop1_chain)
```

Cleaner zoomed interface version:

```pymol
hide everything
show cartoon, cop1_chain or etv4_chain
color slate, cop1_chain
color wheat, etv4_chain

select cop1_interface, byres (cop1_chain within 4 of etv4_chain)
select etv4_interface, byres (etv4_chain within 4 of cop1_chain)

show sticks, cop1_interface or etv4_interface
color marine, cop1_interface
color orange, etv4_interface

hide labels
bg_color white
set cartoon_transparency, 0.25
set stick_radius, 0.18
set sphere_scale, 0.25
set ray_trace_mode, 1
set antialias, 2
set depth_cue, 0
set ray_opaque_background, 0

zoom cop1_interface or etv4_interface, 6
```

Images were exported:

```pymol
png ~/Desktop/COP1_ETV4_overview.png, width=2400, height=1800, dpi=300, ray=1
png ~/Desktop/COP1_ETV4_interface_zoom.png, width=2400, height=1800, dpi=300, ray=1
```

Note: these PyMOL exports contained a visible PyMOL evaluation watermark. For final poster use, rerender using licensed/open-source PyMOL if possible.



## 14. Summary

AlphaFold/ColabFold-predicted COP1–ETV4 structures were visualized in PyMOL to identify the predicted interaction region. GNINA 1.0 was used on Duke DCC to perform structure-based docking of FDA-like compounds against COP1. A 10-compound pilot screen was first used to validate the docking box, Slurm array workflow, and output parsing. The full 2,162-compound SDF library was then split into individual ligand files and docked using a GPU Slurm array. Docking outputs were monitored for completion, parsed for top-scoring poses, and filtered for `ligand outside box` warnings. PyMOL was used to visualize pilot hits and generate COP1–ETV4 interface figures for poster presentation.
