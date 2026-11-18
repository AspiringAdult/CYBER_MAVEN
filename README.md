# CYBER_MAVEN
Continual Deep Learning-Based Intrusion Detection for Emerging Cyber Attacks Using Buffer Replay
# CYBER_MAVEN — ContinualIDS

<p align="center">
  <strong>Continual Deep Learning-Based Intrusion Detection for Emerging Cyber Attacks Using Buffer Replay</strong>
  <br>
  A research prototype for class-incremental intrusion detection using experience replay.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.12-blue?logo=python" alt="Python 3.12">
  <img src="https://img.shields.io/badge/PyTorch-2.4.1-orange?logo=pytorch" alt="PyTorch">
  <img src="https://img.shields.io/badge/Framework-Mammoth-informational" alt="Mammoth">
  <img src="https://img.shields.io/badge/Status-Research%20Prototype-yellow" alt="Research prototype">
  <img src="https://img.shields.io/badge/Platform-Linux-lightgrey?logo=linux" alt="Linux">
</p>

---

## 1. About the project

**CYBER_MAVEN** contains the ContinualIDS research project: an experimental deep-learning intrusion detection system designed to study how a model can learn newly emerging cyberattack categories without completely forgetting previously learned categories.

Traditional machine-learning intrusion detection systems are often trained on a fixed collection of attack classes. In a changing network environment, new attack types may appear after deployment. Retraining a model from scratch whenever a new category appears can be expensive, while simply fine-tuning the existing model can cause **catastrophic forgetting**.

ContinualIDS investigates this problem through **class-incremental learning** and **experience replay**.

The project uses the CSE-CIC-IDS2018 network-flow dataset, a PyTorch multilayer perceptron (MLP), and the AImageLab Mammoth continual-learning framework.

### Main research question

> Can a fixed-capacity replay memory help an intrusion detection model learn newly introduced attack categories while retaining useful knowledge about previously learned categories?

The project compares different continual-learning strategies under a shared model architecture and a staged class-incremental learning schedule.

## 2. How the system works

At a high level, the pipeline is:

```text
CSE-CIC-IDS2018 raw CSV files
            |
            v
   Chunked data ingestion
            |
            v
 Cleaning and feature extraction
            |
            v
   78 numeric flow features
            |
            v
   Feature normalization
            |
            v
  Sequential task construction
            |
            v
     PyTorch MLP model
            |
            v
  Continual learning algorithm
            |
            v
      Replay memory
            |
            v
 Evaluation and comparison
```

The raw dataset is processed in chunks rather than loaded entirely into RAM. The processed feature partitions are stored as binary files and accessed through memory mapping.

The model learns the classes in stages. Replay-based methods retain selected examples from earlier stages and use them during later training.

## 3. Dataset and class-incremental schedule

### Dataset

- **Benchmark:** CSE-CIC-IDS2018.
- **Input:** Network-flow records.
- **Features:** 78 numeric flow features.
- **Raw data:** 10 CSV files, approximately 16.2 million rows in the documented ingestion pipeline.
- **Processed data:** Binary feature partitions under `data/processed/`.

The repository does not include the full raw or processed dataset because these files are large. You must obtain the dataset separately and prepare it according to the instructions in [`data/README.md`](data/README.md).

Do not assume that cloning this repository downloads the dataset.

### Active class mapping

| Class ID | Class name |
|---:|---|
| 0 | BENIGN |
| 1 | DoS |
| 2 | BruteForce |
| 3 | WebAttack |
| 4 | Botnet |

The current experiment uses five canonical classes. Other labels in the original dataset are not part of this active five-class experiment.

### Learning stages

The task schedule is **[3, 1, 1]**:

| Stage | Newly introduced classes | Classes available after the stage |
|---|---|---|
| Stage 1 | BENIGN, DoS, BruteForce | 0, 1, 2 |
| Stage 2 | WebAttack | 0, 1, 2, 3 |
| Stage 3 | Botnet | 0, 1, 2, 3, 4 |

This schedule simulates a system that initially learns established traffic categories and subsequently encounters additional attack categories.

## 4. Model architecture

The project uses a fully connected PyTorch MLP:

```text
Input layer:       78 features
       |
       v
Hidden layer 1:    128 neurons
       |
      ReLU
       |
       v
Hidden layer 2:    64 neurons
       |
      ReLU
       |
       v
Output layer:      5 classes
```

The documented architecture has **18,757 trainable parameters**.

### Fixed output head

The model maintains a fixed five-class output head throughout the learning process.

During an earlier stage, classes that have not yet been introduced are excluded from the active classification space. Their output logits are masked, and the implementation enforces zero gradients for unseen output classes.

This is intended to prevent the model from receiving future-class training information before those classes are introduced.

## 5. Continual-learning methods

The repository integrates four methods into the Mammoth framework.

| Method | Model identifier | Purpose |
|---|---|---|
| Class-Balanced Experience Replay | `ids_er` | Retains examples using dynamic per-class memory quotas |
| Dark Experience Replay | `ids_derpp` | Uses replay examples and stored logits for distillation |
| Global Reservoir Experience Replay | `ids_global_er` | Provides an unconstrained reservoir-sampling comparison |
| Sequential SGD | `ids_sgd` | Fine-tuning baseline without replay memory |

### 5.1 Class-Balanced Experience Replay — `ids_er`

This is the project's principal balanced-replay approach.

The replay memory has a documented capacity of 1,500 examples. Its per-class quota changes as new classes are introduced:

| Learning stage | Active classes | Target quota per class |
|---|---:|---:|
| Stage 1 | 3 | 500 |
| Stage 2 | 4 | 375 |
| Stage 3 | 5 | 300 |

The intention is to reduce the risk that high-frequency benign traffic dominates the replay memory while rare attack categories receive too few stored examples.

### 5.2 Dark Experience Replay — `ids_derpp`

DER++ extends ordinary replay by retaining additional information about the model's previous outputs.

The buffer stores input features, labels, logits and task-level class metadata. The training objective combines:

- Cross-entropy loss on the current stream batch.
- Replay cross-entropy loss.
- Logit-distillation loss using stored logits.

This allows the model to learn from stored examples while also being regularized toward earlier output behaviour.

### 5.3 Global Reservoir Experience Replay — `ids_global_er`

This method uses standard reservoir sampling without enforcing equal per-class quotas.

It acts as a comparison method for investigating how unconstrained memory admission behaves under an imbalanced intrusion-detection stream.

### 5.4 Sequential SGD — `ids_sgd`

This baseline trains on the current task data without replaying earlier examples. It provides a reference for studying catastrophic forgetting.

**Important:** The existence of these implementations does not by itself establish which method performs best. That requires controlled, reproducible experiments and analysis of the resulting metrics.

## 6. Repository structure

```text
CYBER_MAVEN/
|
|-- src/
|   |-- config.py
|   |-- data_ingestion.py
|   |-- preprocessing.py
|   |-- model.py
|   |-- evaluation.py
|   |-- continual.py
|
|-- integration/
|   |-- mammoth_bridge/
|   |   |-- backbones/
|   |   |-- datasets/
|   |   |-- buffers/
|   |   |-- models/
|   |   |-- metrics/
|   |   |-- run_mammoth.py
|   |
|   |-- demo/
|   |   |-- diagnostics.py
|   |   |-- buffer_demo.py
|   |   |-- runner.py
|   |   |-- terminal_ui.py
|   |
|   |-- docs/
|   |-- tests/
|
|-- tests/
|-- data/
|   |-- README.md
|   |-- make_dummy_data.py
|   |-- raw/                 # Obtained separately; not included
|   |-- processed/           # Generated locally; not included
|
|-- outputs/
|   |-- demo_runs/           # Generated locally; not included
|
|-- run_demo.sh
|-- README.md
```

### Important directories

- `src/` — Original research pipeline, data processing, model and evaluation components.
- `integration/mammoth_bridge/` — Adapter layer connecting the project to Mammoth.
- `integration/demo/` — Terminal interface, diagnostics, buffer demonstration and run orchestration.
- `integration/tests/` — Tests for the Mammoth integration and its contracts.
- `tests/` — Tests for the original project components.
- `data/` — Dataset instructions and locally prepared data.
- `outputs/` — Locally generated experiment and demonstration artifacts.

## 7. Requirements

The documented development environment uses:

| Component | Documented environment |
|---|---|
| Operating system | Linux / Ubuntu-based distribution |
| Python | 3.12 |
| PyTorch | 2.4.1 |
| CUDA-enabled PyTorch build | `2.4.1+cu121` |
| GPU | NVIDIA GPU recommended |
| Continual-learning framework | AImageLab Mammoth |

A CUDA-capable NVIDIA GPU is recommended for faster training. The demonstration runner is documented to support CPU fallback when CUDA is unavailable.

Your actual CUDA compatibility depends on your NVIDIA driver, installed PyTorch build and hardware. Do not install a CUDA build blindly without checking compatibility.

## 8. Getting the project

Open a terminal.

### Step 1 — Install Git

On Ubuntu or Linux Mint:

```bash
sudo apt update
sudo apt install git -y
git --version
```

### Step 2 — Clone the repository

```bash
cd ~/Desktop
git clone https://github.com/AspiringAdult/CYBER_MAVEN.git
cd CYBER_MAVEN
```

If the repository is private, sign in to GitHub when prompted or configure an appropriate authentication method.

### Step 3 — Inspect the project

```bash
pwd
ls
```

You should see the project files, including `src/`, `integration/`, `tests/` and `run_demo.sh`.

## 9. Environment setup

Use Python 3.12 for the documented environment.

Check your Python version:

```bash
python3 --version
```

Create and activate a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Upgrade pip:

```bash
python -m pip install --upgrade pip
```

Install the project's Python dependencies using the dependency file supplied in the repository, if present:

```bash
pip install -r requirements.txt
```

If the repository does not contain `requirements.txt`, do not guess package versions. Check the project's dependency documentation or ask the maintainer for the tested dependency list.

### Mammoth framework

The integration uses the AImageLab Mammoth continual-learning framework. The Mammoth code and its dependencies must be available to the Python environment used to run the project.

Follow the project's documented Mammoth setup instructions and verify that the local Mammoth installation is compatible with the integration. Do not assume that installing this repository's Python dependencies alone installs Mammoth.

## 10. Dataset setup

The full CSE-CIC-IDS2018 dataset is not distributed inside this Git repository.

1. Obtain the CSE-CIC-IDS2018 dataset from its official source.
2. Read [`data/README.md`](data/README.md) before placing files.
3. Put the raw CSV files in the expected `data/raw/` directory.
4. Run the project's documented ingestion and preprocessing procedure.
5. Confirm that the required binary partitions are present under `data/processed/`.
6. Run the preflight diagnostics before starting a training demonstration.

Expected directories:

```text
data/
|-- raw/
|   |-- [CSE-CIC-IDS2018 CSV files]
|
|-- processed/
    |-- [processed binary partitions]
```

The raw CSV files and processed binary partitions are intentionally excluded from Git because of their size.

## 11. Running the demonstration

The main entry point is the root-level shell script:

```bash
./run_demo.sh
```

If Linux reports a permission error, make the script executable:

```bash
chmod +x run_demo.sh
./run_demo.sh
```

### Useful commands

Run the system diagnostics:

```bash
./run_demo.sh --preflight
```

Run the default quick demonstration using `ids_er`:

```bash
./run_demo.sh --quick
```

Run DER++:

```bash
./run_demo.sh --model ids_derpp --quick
```

Run Global Reservoir ER:

```bash
./run_demo.sh --model ids_global_er --quick
```

Run the sequential SGD baseline:

```bash
./run_demo.sh --model ids_sgd --quick
```

Display help:

```bash
./run_demo.sh --help
```

The interactive terminal interface provides options for running the guided demonstration, selecting a model, demonstrating replay-buffer behaviour, checking system and dataset diagnostics, viewing the architecture and inspecting the latest demonstration results.

### What does `--quick` mean?

The quick demonstration uses Mammoth's debug mode and a small number of training batches. It is designed to demonstrate that the pipeline, staged learning loop and replay implementation execute.

**A quick/debug run is not a full-scale experiment. Its metrics must not be presented as final research results or as evidence of generalization.**

## 12. Running tests

From the repository root, activate the project's virtual environment and run:

```bash
python -m pytest
```

To run a particular test directory:

```bash
python -m pytest tests/
python -m pytest integration/tests/
```

A passing test suite verifies the tested software contracts. It does not replace full-scale model training, statistical analysis or reproducible experimental evaluation.

## 13. Outputs and results

Demonstration artifacts are stored under:

```text
outputs/demo_runs/
```

These may include logs, JSON summaries and evaluation outputs from individual demonstration runs.

The output directory is generated locally. It should not be treated as a source-code directory, and generated results should not automatically be committed to Git.

When interpreting results, examine more than overall accuracy. Relevant measures include:

- Per-class precision.
- Per-class recall.
- Per-class F1-score.
- Macro F1-score.
- Confusion matrix.
- Class-incremental accuracy.
- Retention and forgetting across stages.

In intrusion detection, high overall accuracy can hide poor detection of rare attack categories. Per-class metrics are therefore important.

## 14. Troubleshooting

### `./run_demo.sh: Permission denied`

```bash
chmod +x run_demo.sh
./run_demo.sh
```

### CUDA is unavailable

Check whether PyTorch can detect CUDA:

```bash
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"
```

Check the NVIDIA driver:

```bash
nvidia-smi
```

If CUDA is unavailable, the documented runner can fall back to CPU. To force CPU execution:

```bash
export CUDA_VISIBLE_DEVICES=""
./run_demo.sh --quick
```

### Terminal colours or escape characters look incorrect

Try the plain-text terminal mode:

```bash
TERM=dumb ./run_demo.sh
```

### Dataset files are missing

Check:

```bash
ls -lh data/raw/
ls -lh data/processed/
```

If the expected files are absent, follow `data/README.md`. Cloning the Git repository does not download the dataset.

### Python package or Mammoth import errors

First confirm that the intended virtual environment is active:

```bash
which python
python --version
```

Then verify the project's dependency and Mammoth setup instructions. Avoid installing random package versions to fix an import error without checking compatibility.

## 15. Research limitations and responsible interpretation

This repository is a research prototype. Its current staged class-incremental setup is a controlled experimental representation of emerging attack categories; it should not be described as a complete real-time production IDS.

Keep the following distinctions clear:

1. A successful demo confirms that the demonstration pipeline executes; it does not establish production readiness.
2. Debug-mode metrics are not final experimental results.
3. Replay-buffer behaviour demonstrations illustrate memory-admission policies; they are not substitutes for full model evaluation.
4. Claims about retention, catastrophic forgetting or superiority of a method must be supported by comparable experiments.
5. A fair method comparison requires consistent data splits, training exposure, random seeds, evaluation procedures and compute budgets.

The longer-term research work can investigate full-scale experiments, replay-memory trade-offs, class imbalance, retention, and adaptation to emerging attack categories.

## 16. Further documentation

Additional project documentation is maintained under `integration/docs/`, including:

- Project architecture overview.
- Friday demonstration runbook.
- Experience Replay implementation notes.
- DER++ implementation and test reports.
- Global Reservoir ER implementation and test reports.
- Experiment-readiness and protocol documentation.

These documents provide implementation-level detail beyond this introductory README.

## 17. Acknowledgements and references

- **Dataset:** CSE-CIC-IDS2018, developed through the Canadian Institute for Cybersecurity and the Communications Security Establishment.
- **Continual-learning framework:** [AImageLab Mammoth](https://github.com/aimagelab/mammoth).
- **Deep-learning framework:** [PyTorch](https://pytorch.org/).

Please consult the original dataset and framework publications and documentation when using this project for academic work.

## 18. License

A license has not been specified in this README. Check the repository's `LICENSE` file before redistributing or reusing the code. If no license file exists, do not assume that the repository is open-source licensed.

---

<p align="center">
  <strong>CYBER_MAVEN — ContinualIDS Research Project</strong>
  <br>
  Learning emerging attacks while studying the challenge of forgetting what came before.
</p>
