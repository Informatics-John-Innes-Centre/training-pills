# Training Image Models with Nextflow and DVC

## Counting and classification models on the GPU nodes, with every run on record

**Time:** about 45–60 minutes (plus queue time for the GPU test)
**Level:** Intermediate — assumes you have done the [Slurm](slurm-basics.md), [git](git-basics.md), [conda](conda-basics.md) and [Singularity](singularity-basics.md) pills

By the end of this tutorial, you will be able to:

- Explain what an "experiment" is, and why tracking them matters for a paper
- Explain the few DVC ideas you need: parameters, stages, experiments and the cache
- Set up a project from the template, build the training container and download pretrained models
- Lay out your images for a **counting** model (YOLO) or a **classification** model (timm)
- Compare several models or settings in one go, each trained as its own GPU job
- Read the results table, pick the best experiment and keep it
- Use a trained model to count objects or classify new images

The main idea to take away:

> **Nextflow runs the experiments, DVC remembers them. Each run is saved as an experiment: the exact code, data, pretrained weights and settings that went in, and the metrics and model that came out. You can always say how a published result was made, and make it again.**

---

## 1. Why track experiments?

Training a model is rarely one run. You try a bigger model, a larger image size, more epochs, a different confidence threshold… and soon you have:

```text
runs/
├── train/
├── train2/
├── train_yolo11m_final/
└── train_yolo11m_final_REALLY_final/
```

Six months later, nobody can say which code, which version of the images and which settings produced the model in the paper.

This template treats every run as an **experiment**, and records all of it together:

```text
code (git commit)  +  data version  +  pretrained weights  +  settings  →  metrics + trained model
```

The record is kept by [DVC](https://dvc.org) (Data Version Control), which works alongside git: git tracks the small text files, DVC tracks the large files (images, models) and keeps the experiment history.

## 2. What the pipeline does

Two kinds of task are supported:

| Task | Question it answers | Model | Main score |
| --- | --- | --- | --- |
| `count` | "How many wheat heads / fruits / insects are in this image?" | YOLO (Ultralytics), detects each object then counts them | `count_mae`: average miscount per image |
| `classify` | "Which disease / growth stage / variety is this?" | Any of hundreds of pretrained networks from timm (ConvNeXt, EfficientNet, ViT, ResNet…) | `f1_macro`: balanced accuracy over all classes |

**Which task fits your question?**

- **One answer per image → `classify`.** The whole photo gets one label: a phenotype class or a score. Examples: leaf disease (healthy / yellow rust / septoria), a disease score from 0 to 5, growth stage, variety, stressed versus control. You need one folder of example images per class.
- **Many things to find in an image → `count`.** Every object gets a box, and the boxes are counted. Examples: wheat heads in a plot, fruits on a plant, flowers, pods, spikelets, insects on a sticky trap, seedlings in a tray. You need images with a box drawn around every object (Section 5).
- **How many of each kind → `count` with several classes.** Give each box a class, e.g. `ripe` / `unripe` fruit, heads by growth stage, or insects by species. You get the total **and** a count per class for every image, each with its own scores.

**Not covered (yet):**

- **Several labels on one image.** Classification picks exactly one class per photo, so a leaf with both rust *and* mildew can't be labelled as both.
- **Areas and percentages.** For example, "% of the leaf covered by lesions" needs segmentation (outlining pixels rather than drawing boxes), which is a different kind of model.

If your project needs one of these, talk to Informatics before you start annotating: the right annotation depends on it.

You list what you want to compare, and the pipeline does the rest:

```text
params.yaml + sweep.yaml       what to try, e.g. 10 YOLO sizes
   └─ PLAN     (seconds)       checks settings, data and pretrained weights before anything is queued
       └─ TRAIN  (GPU jobs)    one Slurm GPU job per experiment, all in parallel, in a container
           └─ RECORD           each finished run becomes a named DVC experiment
               └─ SUMMARY      a leaderboard: best experiment first
```

Template repository: `https://git.nbi.ac.uk/workflows/ai-classification-dvc-tracked`

> **Pretrained models:** the pipeline never trains from scratch. It starts from a model already trained on millions of general images and **fine-tunes** it on yours. This needs far fewer images: hundreds rather than millions.

## 3. Just enough DVC

You only need four ideas:

- **`params.yaml`**: the settings of one experiment (task, data folder, model, epochs, image size…). Every line has a comment explaining it.
- **`dvc.yaml`**: the recipe, i.e. which command to run, what it depends on (code, data, weights, settings) and what it produces (model, metrics, plots). You don't need to edit it.
- **Experiment**: one run of the recipe with particular settings, saved under a name such as `yolo11m`. It's stored in git (as a hidden branch), so it costs almost nothing.
- **Cache**: where DVC keeps the large files (trained models, versioned data). Git only stores a short fingerprint of them.

And four commands:

```bash
dvc exp show              # table of all experiments: settings + metrics
dvc exp apply yolo11m     # bring one experiment's settings, metrics and model into your folder
dvc add <folder>          # version a folder of large files (data, weights)
dvc plots diff a b --open # compare two experiments' charts
```

> **A DVC remote is optional.** Just as git can push to a remote (GitLab), DVC can push the large files to shared storage (`dvc push` / `dvc pull`) so others can download your models and data. Here we use DVC only for **version control**: everything stays in your project's DVC cache (`.dvc/cache`), which is enough to record, compare and reproduce every experiment. A remote can be added later without changing anything else.

> **Nextflow in one line:** it turns a list of experiments into Slurm jobs, runs them in a fixed container and remembers which finished, so a failed run picks up where it stopped (`-resume`). The [Parabricks](parabricks-nextflow.md) pill covers it in more depth.

## 4. One-time setup

> **Internet access:** `software23` is the only node with internet access, and it **can't submit Slurm jobs**. Do everything that downloads (conda environment, container, pretrained models) on `software23`, and submit runs from a login node.

**Git identity.** Saving experiments makes git commits, so git needs to know who you are (once per account):

```bash
git config --global user.name "Your Name"
git config --global user.email "you@nbi.ac.uk"
```

**The conda environment.** It holds Nextflow and DVC; the training software lives in the container. If you haven't used conda on the HPC, do the [conda](conda-basics.md) pill first for the `~/.condarc` channel setup. On `software23`:

```bash
ssh software23
cd /jic/scratch/groups/<your-group>/<you>
git clone https://git.nbi.ac.uk/workflows/ai-classification-dvc-tracked.git my-project
cd my-project
conda env create -f environment.yml          # creates the "vision-dvc" environment
conda activate vision-dvc
```

> **GitLab asks for a password?** Over HTTPS it usually wants a **personal access token**, not your normal password: in GitLab, avatar → *Preferences* → *Access tokens*, with the `read_repository` and `write_repository` scopes. Paste the token when asked for the password. `git config --global credential.helper 'cache --timeout=28800'` remembers it for a working day.

> **Already have a Nextflow environment** (e.g. `nf-env` from the Parabricks pill)? Add DVC to it instead: `conda install -n nf-env -c conda-forge dvc pyyaml`, then put `export NF_CONDA_ENV=nf-env` in your `~/.bashrc` so the launcher uses it.

**Make it your own project.** Your experiments will be pushed to git, so point the clone at a repository of your own (create an empty one on git.nbi.ac.uk first):

```bash
git remote set-url origin https://git.nbi.ac.uk/<your-group>/my-project.git
git push -u origin main
```

**The training container.** Still on `software23`:

```bash
./containers/build.sh
```

This builds `containers/vision-train.sif` (PyTorch, Ultralytics YOLO and timm). It takes 10–20 minutes and needs about 20 GB free in `/tmp`. It builds in `/tmp` because Singularity's `--fakeroot` builds can't read `/jic/scratch`: building in the project folder fails with *"Failed to restore current working directory: Permission denied"*.

> **Build it on `software23`, but don't run it there.** On `software23`, Singularity can't see your project folders (*"Not mounting current directory: user bind control is disabled"*). That's fine: training runs the container on the GPU nodes, and nothing else needs it on `software23`.

> **Share the image.** It's several GB. Build it once per group, keep it somewhere shared, and point each project to it with `--train_container /path/to/vision-train.sif`.

**Pretrained models.** Download the models you want to compare. Compute nodes are offline, so this has to happen here. It's plain Python, so the conda environment is all it needs:

```bash
./scripts/fetch_weights.sh yolo11n yolov8n                    # the two small ones used by the test
./scripts/fetch_weights.sh yolo11m yolov8m                    # counting
./scripts/fetch_weights.sh convnext_tiny.fb_in22k_ft_in1k     # classification
./scripts/fetch_weights.sh --for params.yaml sweep.yaml       # every model your setup uses
```

`--for` reads the model names from your `params.yaml` and `sweep.yaml` (or an example folder), so you never miss one. Models already downloaded are skipped.

YOLO names run from `n` (nano, fastest) through `s`, `m`, `l` to `x` (largest, most accurate, slowest). For classification, any name from https://huggingface.co/timm works.

Then version them, so each experiment records exactly which weights it started from:

```bash
dvc add models/pretrained
git add models/pretrained.dvc models/.gitignore
git commit -m "Add pretrained weights"
exit
```

## 5. Your images

**For counting**, use the YOLO layout: one text file per image, one line per object, each line a class number and a box (centre x, centre y, width, height, all as fractions of the image size):

```text
my_dataset/
├── images/
│   ├── train/   IMG_001.jpg ...
│   ├── val/     IMG_201.jpg ...
│   └── test/    IMG_251.jpg ...     (optional, but recommended)
└── labels/
    ├── train/   IMG_001.txt ...     0 0.412 0.533 0.051 0.078
    ├── val/                          0 0.700 0.120 0.048 0.081
    └── test/                         ...
```

Annotation tools such as CVAT, Label Studio and Roboflow can all export in this "YOLO" format. Then set `count.classes` in `params.yaml` to the names of your classes, in the order of their class numbers: `[wheat_head]` for one class, or `[ripe, unripe]` when class `0` is ripe and `1` is unripe.

**For classification**, use one folder per class. The pipeline splits them into training, validation and test images itself (70/15/15, the same way every time):

```text
my_photos/
├── healthy/      *.jpg
├── yellow_rust/  *.jpg
└── septoria/     *.jpg
```

If you've already split them yourself, use `train/<class>/`, `val/<class>/` and `test/<class>/` instead.

> **Group related photos.** If your scores live in a spreadsheet rather than in folder names, or several photos show the same plant, write a small script that builds the class folders and keeps each plant (or line) in a single split. `examples/pea_aphanomyces/prepare.py` does exactly this and is a good starting point.

> **Train, validation and test:** the model learns from **train**, the best epoch is chosen on **val**, and the final scores come from **test**, images the model never saw while training. Keep near-identical images (e.g. the same plot photographed twice) in the same split, or the scores will look better than they really are.

Your data can stay where it is: set `data.path` in `params.yaml` to the folder, e.g. `/jic/scratch/groups/<your-group>/images/wheat_2026`. DVC records a fingerprint of it with every experiment, so you can tell when a model was trained on a different version.

## 6. Test before you scale up

This trains two tiny YOLO models for 3 epochs on a small bundled dataset of synthetic images (a few minutes plus queue time).

For this first test, it's easiest to **watch it live** from an `interactive` session. Running the launcher with `bash` instead of `sbatch` keeps Nextflow on your screen, while training still goes to the GPU nodes as separate jobs:

```bash
interactive
cd /jic/scratch/groups/<your-group>/<you>/my-project
bash submit.sh -profile jic,test
```

```text
[2b/a3cfb2] Submitted process > PLAN
[c4/7a3c43] Submitted process > TRAIN (smoketest-yolov8n)
[6d/ee5a30] Submitted process > TRAIN (smoketest-yolo11n)
...
```

The session has to stay open until the run finishes. For real sweeps, which take hours, submit from a **login node** (not `software23`) instead:

```bash
cd /jic/scratch/groups/<your-group>/<you>/my-project
sbatch submit.sh -profile jic,test
```

and follow it with:

```bash
squeue -u $USER                 # the head job + the GPU jobs it submitted
tail -f logs/vision-train-<jobid>.out   # the progress
```

When it finishes, the log ends with a leaderboard, and:

```bash
dvc exp show
```

lists two experiments, `smoketest-yolo11n` and `smoketest-yolov8n`. Their scores are poor (3 epochs on fake data); the test only checks that everything is connected. Remove them afterwards:

```bash
dvc exp remove smoketest-yolo11n smoketest-yolov8n
```

## 7. A real run

**1. Settings: `params.yaml`.** Set at least `task`, `data.path`, and for counting `count.classes`. The defaults for the rest are sensible starting points.

**2. What to compare: `sweep.yaml`.** Each combination becomes one experiment, trained as its own GPU job:

```yaml
train.model: [yolov8m, yolo11m, yolo11l]    # 3 models
train.imgsz: [640, 1024]                    # x 2 image sizes = 6 experiments
```

Any setting in `params.yaml` can go here, written as a dotted path (`train.epochs`, `count.conf`, …). For a single run of `params.yaml`, set the file to `{}`.

**3. Launch** (from a login node):

```bash
sbatch submit.sh
```

`submit.sh` runs Nextflow as a small, long-running **head job** on `jic-long`. It activates the conda environment, works offline and always adds `-resume`. The head job submits one GPU job per experiment to `jic-gpu`.

Ready-made setups for real JIC datasets are in `examples/`. Each `params.yaml` explains itself in its first lines:

| Example | Task | What it shows |
|---|---|---|
| `wheat_heads` | count | Wheat heads (GWHD 2021): YOLOv8 vs YOLO11, every size |
| `flea_beetle` | count, 3 classes | Count per class (L1 / L2 / L3) on large TIFFs with outlined objects |
| `wheat_disease` | classify | Wheat disease photos, one folder per class |
| `pea_aphanomyces` | classify | Disease index 0-4 from a score spreadsheet; `prepare.py` builds the classes and splits **by line** so test lines are unseen |

To run one, first fetch its models on `software23` (and, for `pea_aphanomyces`, build its dataset with the command at the top of its `params.yaml`):

```bash
./scripts/fetch_weights.sh --for examples/wheat_heads
dvc add models/pretrained && git add models/pretrained.dvc models/.gitignore && git commit -m "Weights for wheat_heads"
```

then launch it from a login node. For example, the YOLOv8-versus-YOLO11 comparison:

```bash
sbatch submit.sh --params_file examples/wheat_heads/params.yaml --sweep examples/wheat_heads/sweep.yaml
```

Useful options:

| Option | Use it when |
| --- | --- |
| `--exp_prefix wheat2026` | You want experiment names like `wheat2026-yolo11m` |
| `--slurm_gpu_type ''` | Jobs wait a long time for an A100: accept any GPU |
| `--gpu_time 48h` | Large models or many epochs need longer than 24 h |
| `--max_parallel 4` | You want fewer GPU jobs running at the same time (default 8) |
| `--push true` | Push each new experiment to your git remote straight away |

## 8. Reading the results

```bash
dvc exp show
```

The leaderboard for the latest sweep, best first, is in `nf-results/leaderboard.md`. Here it is for the `flea_beetle` example (3 classes, L1 to L3; 212 training and 27 test photos), with 4 YOLO11 sizes at 2 image sizes:

```text
| rank | experiment        | count_mae | count_rel_mae | count_bias | count_r2 | mAP50 | train_minutes |
|------|-------------------|-----------|---------------|------------|----------|-------|---------------|
| 1    | yolo11m-imgsz1536 | 1.778     | 3.76          | 1.778      | 0.994    | 0.988 | 15.0          |
| 2    | yolo11s-imgsz1024 | 2.889     | 6.11          | 2.889      | 0.9869   | 0.976 | 9.1           |
| 3    | yolo11s-imgsz1536 | 3.259     | 6.89          | 3.259      | 0.9875   | 0.983 | 11.0          |
| ...  |                   |           |               |            |          |       |               |
| 8    | yolo11x-imgsz1536 | 5.519     | 11.67         | 5.519      | 0.9673   | 0.944 | 27.4          |
```

> **Reading it like a scientist.** The best model miscounts by 1.8 objects per photo (3.8%), on photos holding 5 to 226 objects. Three things stand out:
>
> - **`count_bias` equals `count_mae` in every row.** That only happens when the model *over*-counts on every photo and never under-counts: a systematic offset, not random error. Raising the confidence threshold removes such borderline extra detections. The pipeline now does this for you (`count_conf`, below).
> - **Bigger isn't better.** The `l` and `x` models do worse than `s` and `m`: with 212 training photos, large models overfit. They also take longer.
> - **Small gaps aren't proof.** With 27 test photos, first and second place differ by about one object per photo. Treat close rankings as ties.

**Counting scores:**

- `count_mae`: the average number of objects miscounted per image (lower is better). This is the main score.
- `count_rel_mae`: the same, as a % of the true count, so you can compare datasets.
- `count_bias`: negative means the model **under-counts** on average, positive means it over-counts. Under-counting usually means objects are touching or overlapping.
- `count_r2`: how well predicted counts follow the true counts (1 = perfectly).
- `mAP50`, `precision`, `recall`: how good the boxes themselves are.
- `per_class.<name>.count_mae` (and `count_bias`, `mAP50`…): the same scores for each class, when you have more than one. A model can count the total well while confusing ripe and unripe; these columns show it.
- `count_conf`: the confidence threshold used for counting. A box counts only if the model is at least this sure. Several thresholds are tried (`count.conf_scan` in `params.yaml`), and the one that counts best on the **validation** photos is used. The scores above are then measured on the **test** photos, which played no part in the choice, so they stay honest. `predict.py` uses the chosen threshold automatically.

**Classification scores:**

- `f1_macro`: the main score. It weighs every class equally, so a rare class the model gets wrong still pulls it down.
- `accuracy`: the fraction of images classified correctly. It can look high when one class dominates.
- `recall.<class>`: the fraction of each class found. Look here to see which class the model struggles with.

**Plots:** training curves, predicted-versus-true counts and confusion matrices, side by side for any experiments:

```bash
dvc plots diff yolo11l yolov8m --open
```

The [DVC extension for VS Code](https://marketplace.visualstudio.com/items?itemName=Iterative.dvc) shows the same table and plots interactively.

> **Big differences only:** with a small test set, two experiments a few percent apart may not really differ. Prefer the simpler, faster model unless the better one is clearly better.

## 9. Keep the best and share it

```bash
git status                       # start from a clean folder (commit or `git stash` first)
dvc exp apply yolo11m-imgsz1536  # its settings, metrics and model, into your folder
git add -A
git commit -m "Adopt yolo11m-imgsz1536 (count_mae 1.78)"
git push                         # the record: settings, code, metrics, fingerprints
```

Adopt an experiment soon after its sweep: `dvc exp apply` also restores the code in `src/` as it was when the experiment ran, so applying an old one after the code has been updated would roll the code back. To *use* a model later, export it instead (Section 10).

Your project's `main` branch now holds that experiment: its exact settings, code and metrics, and fingerprints of its data, weights and model. The trained model itself is in the project's DVC cache on the cluster, so anyone working in the project folder can use it. A fresh clone elsewhere gets the record but not the large files, until a DVC remote is set up (see the note in Section 3).

## 10. Use the best model for your analysis

**1. Copy the model out of DVC** (with the conda environment active):

```bash
./scripts/export_model.sh yolo11m-imgsz1536
```

This creates `models/trained/yolo11m-imgsz1536/` holding the weights, the experiment's test scores and its settings. Your folder and code stay as they are. If a rerun sweep left several experiments with the same name, the newest is used and the older ones are listed, with the full name to pass if you want one of those.

> **Why not `dvc exp apply`?** `apply` brings back *everything* the experiment was made with, including the code in `src/` as it was then. That's what you want for reproducing it, not for analysis with today's tools.

**2. Run it on your images as a GPU job** (from a login node):

```bash
sbatch scripts/predict.sh models/trained/yolo11m-imgsz1536 /path/to/field/images analysis/field_2026.csv --save-images
```

The image folder may contain subfolders (one per date, plot or flight). Each CSV row keeps the image's path, so it can be joined to your plot metadata in R, Python or Excel:

```text
image,count,L1,L3,L2
2026-06-01/plot_A/IMG_0012.tif,48,48,0,0
2026-06-01/plot_B/IMG_0013.tif,31,0,0,31
```

For a counting model you get the total and one column per class; for a classification model, the predicted class, its `confidence` and a probability for every class. `--save-images` also writes copies of the images with the boxes drawn, in `analysis/field_2026_images/`.

**3. Keep the `.json` next to the CSV.** `analysis/field_2026.json` records what produced the numbers: the experiment and its commit, the model's settings (including the counting threshold), its test scores and a fingerprint of the weights. Cite the experiment name and commit in your methods.

> **Before trusting thousands of numbers:**
> - **Look at the boxes** (`--save-images`) on a few dozen images first.
> - **Stay in your training domain.** A model trained on one camera, growth stage or site can do worse on another. Label a small sample of the new images and compare.
> - **Flag doubtful classifications** with a low `confidence` (e.g. below 0.6) for a manual look.

## 11. When something fails

**Most mistakes are caught in the first seconds**, before any GPU job is queued, with a message saying what to fix:

```text
ERROR: sweep.yaml: 'train.modle' does not exist in params.yaml. Keys are dotted paths, e.g. train.model or count.conf
ERROR: Pretrained weights for 'yolo11x' not found in .../models/pretrained/yolo11x.
  Compute nodes have no internet, so fetch them once on software23 (the node with internet):
    ./scripts/fetch_weights.sh yolo11x
```

If a training job fails, Nextflow prints its work directory. Look there:

```bash
cat work/d4/376aa8*/.command.log     # the full training output
```

Fix the cause and run `sbatch submit.sh` again. Thanks to `-resume`, finished training jobs are reused, not repeated.

**Common causes:**

- **`CUDA out of memory`:** lower `train.batch` or `train.imgsz`.
- **`src/ or dvc.yaml changed since experiment…`:** you edited the code while a run was training. The experiment can't be recorded against code that didn't produce it. `git stash` your edits and resubmit: training is reused.
- **`sbatch: command not found`:** you're on `software23`. Submit from a login node.
- **`Authentication failed` on `git pull` or `git push`:** use a personal access token as the password (Section 4).
- **`URL using bad/illegal format`:** the git remote mixes two styles. HTTPS uses a slash, `https://git.nbi.ac.uk/<group>/<repo>.git`; SSH uses a colon, `git@git.nbi.ac.uk:<group>/<repo>.git`. Fix it with `git remote set-url origin <correct URL>`.
- **One model failed but the others finished:** a failed experiment is retried once and then skipped, so the rest of the sweep still completes. Its log is in the work directory listed in the head job's output.

> **Don't delete `work/` while you are still running sweeps.** That's where `-resume` finds finished training. Once your experiments are recorded (they live in git and the DVC cache, not in `work/`), delete it to free space.

## 12. A few habits worth building

- **Always run the test profile first** on a new setup.
- **Change one thing at a time** in a sweep, so you know what made the difference.
- **Name your sweeps** with `--exp_prefix`, e.g. the dataset or the date.
- **Commit before you sweep**, so every experiment hangs off a clean, named commit.
- **Version your data** if it changes: `dvc add` the image folder, so experiments on old and new annotations can't be confused.
- **Keep a test set aside** and never tune settings on it.
- **Report the experiment name and git commit** in your methods, so the result can be traced.

## 13. Your image-model survival kit

| Command | What it does |
| --- | --- |
| `ssh software23` | The only node with internet: environment, container, pretrained models |
| `./containers/build.sh` | Build the training container (on `software23`) |
| `./scripts/fetch_weights.sh yolo11m` | Download pretrained weights (on `software23`) |
| `dvc add models/pretrained` | Version the pretrained weights |
| `bash submit.sh -profile jic,test` | Five-minute check of the whole setup, watched live (inside `interactive`) |
| `sbatch submit.sh` | Run every experiment in `sweep.yaml` |
| `tail -f logs/vision-train-<jobid>.out` | Follow progress |
| `dvc exp show` | Compare all experiments |
| `cat nf-results/leaderboard.md` | Best experiment of the last sweep |
| `dvc plots diff <exp1> <exp2> --open` | Compare training curves and results |
| `dvc exp apply <name>` | Bring an experiment into your folder |
| `dvc exp remove <name>` | Delete an experiment |
| `./scripts/export_model.sh <name>` | Copy an experiment's model out, ready for analysis |
| `sbatch scripts/predict.sh <model> <images> <out.csv>` | Count or classify a folder of images on a GPU |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Set your git name and email, then clone the template on `software23` and create the conda environment
- [ ] Build the container with `./containers/build.sh`
- [ ] Fetch `yolo11n` and `yolov8n`, and version them with `dvc add models/pretrained`
- [ ] In an `interactive` session, run `bash submit.sh -profile jic,test` and watch the jobs go through
- [ ] Run `dvc exp show` and find the two `smoketest` experiments and their `count_mae`
- [ ] Run `dvc exp apply smoketest-yolo11n` and look at `results/metrics.json` and `results/plots/counts.csv`
- [ ] Export `smoketest-yolo11n` with `./scripts/export_model.sh`, run `scripts/predict.sh` on `tests/data/count_tiny/images/test` with `--save-images`, and look at the boxes and the `.json`
- [ ] Restore your folder with `git checkout -- .` and remove the smoke-test experiments
- [ ] Lay out a small set of your own images, set `data.path`, and sweep two models
