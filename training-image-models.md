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

**Make it your own project.** Your experiments are pushed to git, so each project needs a repository of its own. Create an empty one on git.nbi.ac.uk, then keep the template as a second remote, called `template`, so you can pick up its improvements later:

```bash
git remote rename origin template
git remote add origin https://git.nbi.ac.uk/<your-group>/my-project.git
git push -u origin main
```

Later, `git pull template main` brings in template updates (never while a sweep is running).

**The training container.** All training runs in one Singularity image with PyTorch, Ultralytics YOLO and timm. **A shared copy is installed** at `/jic/common/workflows/ai/containers/vision-train.sif`, like the Parabricks pipeline's images, and the scripts use it automatically, so normally there's nothing to do.

Only if you need your own image (or the shared one is missing), build it on `software23`:

```bash
./containers/build.sh                  # builds containers/vision-train.sif, which then takes precedence
./containers/install_shared.sh         # optional: make it the shared copy for everyone
```

Building takes 10–20 minutes and needs about 20 GB free in `/tmp`. It builds in `/tmp` because Singularity's `--fakeroot` builds can't read `/jic/scratch`: building in the project folder fails with *"Failed to restore current working directory: Permission denied"*.

> **Build it on `software23`, but don't run it there.** On `software23`, Singularity can't see your project folders (*"Not mounting current directory: user bind control is disabled"*). That's fine: training runs the container on the GPU nodes, and nothing else needs it on `software23`.

**Pretrained models.** Download the models you want to compare. Compute nodes are offline, so this has to happen here. It's plain Python, so the conda environment is all it needs:

```bash
./scripts/fetch_weights.sh --for tests/params.count.yaml tests/sweep.count.yaml   # the two small models the test uses
./scripts/fetch_weights.sh --for params.yaml sweep.yaml                           # every model your own setup uses
./scripts/fetch_weights.sh yolo11m convnext_tiny.fb_in22k_ft_in1k                 # or name them
```

`--for` reads the model names from a `params.yaml` and `sweep.yaml`, so you never miss one. Models already downloaded are skipped.

YOLO names run from `n` (nano, fastest) through `s`, `m`, `l` to `x` (largest, most accurate, slowest). For classification, any name from https://huggingface.co/timm works.

Then version them, so each experiment records exactly which weights it started from:

```bash
dvc add models/pretrained
git add models/pretrained.dvc models/.gitignore
git commit -m "Add pretrained weights"
exit
```

## 5. Your images

**Lay them out for your task.**

*For counting*, use the YOLO layout: one text file per image, one line per object, each line a class number and a box (centre x, centre y, width, height, all as fractions of the image size):

```text
<your-images>/
├── images/
│   ├── train/   IMG_001.jpg ...
│   ├── val/     IMG_201.jpg ...
│   └── test/    IMG_251.jpg ...     (optional, but recommended)
└── labels/
    ├── train/   IMG_001.txt ...     0 0.412 0.533 0.051 0.078
    ├── val/                          1 0.700 0.120 0.048 0.081
    └── test/                         ...
```

Annotation tools such as CVAT, Label Studio and Roboflow export this "YOLO" format. Set `count.classes` in `params.yaml` to one name per class number, in order: `[head]` for one class, or `[ripe, unripe]` when `0` is ripe and `1` is unripe.

*For classification*, use one folder per class. The pipeline splits them into training, validation and test images itself (70/15/15, the same way every time):

```text
<your-images>/
├── healthy/      *.jpg
├── yellow_rust/  *.jpg
└── septoria/     *.jpg
```

If you've already split them yourself, use `train/<class>/`, `val/<class>/` and `test/<class>/` instead.

**Connect them to the project**, in one of two ways:

- **Point to them.** Set `data.path` in `params.yaml` to the folder, wherever it is on the cluster. Nothing is copied. Every experiment records a fingerprint of the folder, so you can tell if it changed between experiments, but an older version can't be brought back if you edit the images.
- **Version them in the project.** Each version is kept and can be restored, at the cost of the disk space of a copy:

  ```bash
  mkdir -p data && cp -r /path/to/your/images data/my_images
  dvc add data/my_images
  git add data/my_images.dvc data/.gitignore && git commit -m "Images, version 1"
  ```

  and set `data.path: data/my_images`. After adding or relabelling images, `dvc add data/my_images` and commit again: that's a new version, and every experiment records which one it used.

**Check them** (seconds, any node, conda environment active):

```bash
./scripts/check_data.sh
```

```text
Data OK [count] /path/to/your/images: images train=<n>, val=<n>, test=<n>; objects <class 1>=<n>, <class 2>=<n>
```

It reports the images per split or class, and for counting the objects per class. It stops with a clear message on a wrong layout, a label using a class number that `count.classes` doesn't list, and similar mistakes, all before anything is queued.

> **Train, validation and test:** the model learns from **train**, the best epoch (and, for counting, the confidence threshold) is chosen on **val**, and the final scores come from **test**, images the model never saw while training.

> **Keep related photos in one split.** If several photos show the same plant (or line, plot…), a photo-by-photo split puts near-identical twins in both training and test, and the scores look better than they really are. When the group is part of the file name, `data.group_pattern` keeps each group together: `'^(plant\d+)_'` for `plant12_a.jpg`, `plant12_b.jpg`. For counting, or with your own train/val/test folders, put each group in a single split yourself.

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

**Starting points.** `examples/count/` and `examples/classify/` each hold a `params.yaml` and a `sweep.yaml` with sensible settings and a model comparison. Copy the pair for your task to the project root and edit it:

```bash
cp examples/classify/params.yaml examples/classify/sweep.yaml .
./scripts/check_data.sh
```

Useful options:

| Option | Use it when |
| --- | --- |
| `--exp_prefix trial2026` | You want experiment names like `trial2026-yolo11m` |
| `--slurm_gpu_type ''` | Jobs wait a long time for an A100: accept any GPU |
| `--gpu_time 48h` | Large models or many epochs need longer than 24 h |
| `--max_parallel 4` | You want fewer GPU jobs running at the same time (default 8) |
| `--push true` | Push each new experiment to your git remote straight away |

## 8. Reading the results

```bash
dvc exp show
```

The leaderboard for the latest sweep, best first, is in `nf-results/leaderboard.md`: one row per experiment, with its main scores and training time.

> **Reading it like a scientist.** Look beyond the top row:
>
> - **Is the error systematic?** If `count_bias` is about as large as `count_mae`, the model errs in the same direction on almost every image: it consistently over- (or under-) counts. That's an offset, not noise, and the confidence threshold (`count_conf`, below) is the usual fix.
> - **Is bigger better?** Often not with a few hundred training images: large models can overfit and score worse than small ones, while costing more GPU time.
> - **Are the gaps real?** With a few dozen test images, experiments a few percent apart are effectively tied. Prefer the simpler, faster model unless the better one is clearly better.
> - **Which classes struggle?** The per-class scores show where errors come from: one rare class, or two similar classes confused with each other.

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
- `ordinal_mae` and `within_one`: for **ordered classes** such as disease scores or growth stages. Set `classify.ordinal: true` and name the class folders with their number (`score_0`, `score_1`, …). `ordinal_mae` is the average number of steps a prediction is off, and `within_one` the fraction exact or one step off. Mixing up neighbouring scores is common, since people scoring by eye don't always agree either, so these often describe an ordinal model more fairly than `f1_macro`.

**Plots:** training curves, predicted-versus-true counts and confusion matrices, side by side for any experiments:

```bash
dvc plots diff <experiment 1> <experiment 2> --open
```

The [DVC extension for VS Code](https://marketplace.visualstudio.com/items?itemName=Iterative.dvc) shows the same table and plots interactively.

## 9. Keep the best and share it

```bash
git status                       # start from a clean folder (commit or `git stash` first)
dvc exp apply <experiment>       # its settings, metrics and model, into your folder
git add -A
git commit -m "Adopt <experiment>"
git push                         # the record: settings, code, metrics, fingerprints
```

Adopt an experiment soon after its sweep: `dvc exp apply` also restores the code in `src/` as it was when the experiment ran, so applying an old one after the code has been updated would roll the code back. To *use* a model later, export it instead (Section 10).

Your project's `main` branch now holds that experiment: its exact settings, code and metrics, and fingerprints of its data, weights and model. The trained model itself is in the project's DVC cache on the cluster, so anyone working in the project folder can use it. A fresh clone elsewhere gets the record but not the large files, until a DVC remote is set up (see the note in Section 3).

## 10. Use the best model for your analysis

**1. Copy the model out of DVC** (with the conda environment active):

```bash
./scripts/export_model.sh <experiment>
```

This creates `models/trained/<experiment>/` holding the weights, the experiment's test scores and its settings. Your folder and code stay as they are. If a rerun sweep left several experiments with the same name, the newest is used and the older ones are listed, with the full name to pass if you want one of those.

> **Why not `dvc exp apply`?** `apply` brings back *everything* the experiment was made with, including the code in `src/` as it was then. That's what you want for reproducing it, not for analysis with today's tools.

**2. Run it on your images as a GPU job** (from a login node):

```bash
sbatch scripts/predict.sh models/trained/<experiment> /path/to/new/images analysis/<name>.csv --save-images
```

The image folder may contain subfolders (one per date, plot or flight). Each CSV row keeps the image's path, so it can be joined to your plot metadata in R, Python or Excel:

```text
image,count,<class 1>,<class 2>,...
<date>/<plot>/<image>,<total>,<n class 1>,<n class 2>,...
```

For a counting model you get the total and one column per class; for a classification model, the predicted class, its `confidence` and a probability for every class. `--save-images` also writes copies of the images with the boxes drawn, in `analysis/<name>_images/`.

**3. Keep the `.json` next to the CSV.** `analysis/<name>.json` records what produced the numbers: the experiment and its commit, the model's settings (including the counting threshold), its test scores and a fingerprint of the weights. Cite the experiment name and commit in your methods.

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
| `./containers/build.sh` | Build your own training container, if the shared one doesn't suit (on `software23`) |
| `./scripts/fetch_weights.sh --for params.yaml sweep.yaml` | Download the pretrained weights your setup uses (on `software23`) |
| `dvc add models/pretrained` | Version the pretrained weights |
| `./scripts/check_data.sh` | Check your settings and images before training |
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
| `git pull template main` | Bring in template updates (not during a sweep) |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Set your git name and email, clone the template on `software23`, make it your own project (`template` + `origin` remotes) and create the conda environment
- [ ] Check the shared container exists: `ls /jic/common/workflows/ai/containers/`
- [ ] Fetch the test's models with `--for tests/params.count.yaml tests/sweep.count.yaml`, and version them with `dvc add models/pretrained`
- [ ] In an `interactive` session, run `bash submit.sh -profile jic,test` and watch the jobs go through
- [ ] Run `dvc exp show` and find the two `smoketest` experiments and their `count_mae`
- [ ] Run `dvc exp apply smoketest-yolo11n` and look at `results/metrics.json` and `results/plots/counts.csv`
- [ ] Export `smoketest-yolo11n` with `./scripts/export_model.sh`, run `scripts/predict.sh` on `tests/data/count_tiny/images/test` with `--save-images`, and look at the boxes and the `.json`
- [ ] Restore your folder with `git checkout -- .` and remove the smoke-test experiments
- [ ] Lay out a small set of your own images, copy the `examples/` starting point for your task, set `data.path`, and run `./scripts/check_data.sh` until it reports OK
- [ ] Fetch the models with `--for`, sweep two of them, and read the leaderboard with the questions from Section 8
