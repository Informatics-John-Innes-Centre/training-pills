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
| `count` | "How many wheat heads / insects / spikelets are in this image?" | YOLO (Ultralytics), detects each object then counts them | `count_mae`: average miscount per image |
| `classify` | "Which disease / growth stage / variety is this?" | Any of hundreds of pretrained networks from timm (ConvNeXt, EfficientNet, ViT, ResNet…) | `f1_macro`: balanced accuracy over all classes |

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
dvc push / dvc pull       # copy large files to / from shared storage
```

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

> **Share the image.** It's several GB. Build it once per group, keep it somewhere shared, and point each project to it with `--train_container /path/to/vision-train.sif`.

**Pretrained models.** Download the models you want to compare. Compute nodes are offline, so this has to happen here:

```bash
./scripts/fetch_weights.sh yolo11n yolov8n                    # the two small ones used by the test
./scripts/fetch_weights.sh yolo11m yolov8m                    # counting
./scripts/fetch_weights.sh convnext_tiny.fb_in22k_ft_in1k     # classification
```

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

Annotation tools such as CVAT, Label Studio and Roboflow can all export in this "YOLO" format. Then set `count.classes` in `params.yaml` to the names of your classes, e.g. `[wheat_head]`.

**For classification**, use one folder per class. The pipeline splits them into training, validation and test images itself (70/15/15, the same way every time):

```text
my_photos/
├── healthy/      *.jpg
├── yellow_rust/  *.jpg
└── septoria/     *.jpg
```

If you've already split them yourself, use `train/<class>/`, `val/<class>/` and `test/<class>/` instead.

> **Train, validation and test:** the model learns from **train**, the best epoch is chosen on **val**, and the final scores come from **test**, images the model never saw while training. Keep near-identical images (e.g. the same plot photographed twice) in the same split, or the scores will look better than they really are.

Your data can stay where it is: set `data.path` in `params.yaml` to the folder, e.g. `/jic/scratch/groups/<your-group>/images/wheat_2026`. DVC records a fingerprint of it with every experiment, so you can tell when a model was trained on a different version.

## 6. Test before you scale up

From a **login node** (not `software23`), in your project folder:

```bash
cd /jic/scratch/groups/<your-group>/<you>/my-project
sbatch submit.sh -profile jic,test
```

This trains two tiny YOLO models for 3 epochs on a small bundled dataset of synthetic images (a few minutes plus queue time). Follow it with:

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

Ready-made setups for two wheat datasets are in `examples/`, e.g. the YOLOv8-versus-YOLO11 comparison:

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

```text
 Experiment        count_mae  count_rel_mae  count_bias  mAP50   train.model  train.imgsz
 main              -          -              -           -       yolo11m      1024
 ├── yolo11l       2.1        4.3            -0.4        0.93    yolo11l      1024
 ├── yolo11m       2.4        4.9            -0.9        0.92    yolo11m      1024
 └── yolov8m       2.9        5.9            -1.6        0.91    yolov8m      1024
```

*(illustrative numbers)*

The leaderboard for the latest sweep, best first, is in `nf-results/leaderboard.md`.

**Counting scores:**

- `count_mae`: the average number of objects miscounted per image (lower is better). This is the main score.
- `count_rel_mae`: the same, as a % of the true count, so you can compare datasets.
- `count_bias`: negative means the model **under-counts** on average, positive means it over-counts. Under-counting usually means objects are touching or overlapping.
- `count_r2`: how well predicted counts follow the true counts (1 = perfectly).
- `mAP50`, `precision`, `recall`: how good the boxes themselves are.

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
dvc exp apply yolo11l            # its settings, metrics and model, into your folder
git add -A
git commit -m "Adopt yolo11l (count_mae 2.1)"
git push
```

Your project's `main` branch now holds that experiment. Anyone who clones it gets the exact settings, code and metrics, and with `dvc pull` the trained model too, once a DVC remote (shared storage for the large files) has been set up: ask Informatics.

## 10. Use a trained model

After `dvc exp apply`, the model is in `results/model/`. Run it on new images:

```bash
srun --partition=jic-gpu --gres=gpu:1 --mem=16G --pty \
    singularity exec --nv containers/vision-train.sif \
    python src/predict.py --images /path/to/new/images --out counts.csv --save-images
```

This writes one row per image: the count (and, with `--save-images`, copies of the images with the boxes drawn), or the predicted class with its probabilities. Use `--conf` to try a different confidence threshold without retraining.

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
| `sbatch submit.sh -profile jic,test` | Five-minute check of the whole setup (from a login node) |
| `sbatch submit.sh` | Run every experiment in `sweep.yaml` |
| `tail -f logs/vision-train-<jobid>.out` | Follow progress |
| `dvc exp show` | Compare all experiments |
| `cat nf-results/leaderboard.md` | Best experiment of the last sweep |
| `dvc plots diff <exp1> <exp2> --open` | Compare training curves and results |
| `dvc exp apply <name>` | Bring an experiment into your folder |
| `dvc exp remove <name>` | Delete an experiment |
| `python src/predict.py --images <dir>` | Use the trained model (inside the container) |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Set your git name and email, then clone the template on `software23` and create the conda environment
- [ ] Build the container with `./containers/build.sh`
- [ ] Fetch `yolo11n` and `yolov8n`, and version them with `dvc add models/pretrained`
- [ ] From a login node, run `sbatch submit.sh -profile jic,test` and follow it with `tail -f`
- [ ] Run `dvc exp show` and find the two `smoketest` experiments and their `count_mae`
- [ ] Run `dvc exp apply smoketest-yolo11n` and look at `results/metrics.json` and `results/plots/counts.csv`
- [ ] Run `predict.py` on `tests/data/count_tiny/images/test` with `--save-images` and look at the boxes
- [ ] Restore your folder with `git checkout -- .` and remove the smoke-test experiments
- [ ] Lay out a small set of your own images, set `data.path`, and sweep two models
