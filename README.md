# τ₀-VLA: a Hierarchical Robot Foundation Model with World-Model-Guided Test-Time Computation

<div id="top" align="center">

![τ₀-VLA overview](assets/overview.png)

<a href="https://tau0-vla.github.io/"><img src="https://img.shields.io/badge/Project_Website-tau0_VLA-blue" height="25" alt="Project Website"></a> &nbsp; <a href="https://tau0-vla.github.io/tau0-vla.pdf"><img src="https://img.shields.io/badge/Paper-tau0_VLA-red" height="25" alt="Paper"></a> &nbsp; <a href="https://huggingface.co/sii-research/tau-0-vla"><img src="https://img.shields.io/badge/Weight-Hugging_Face-orange" height="25" alt="Model Weights"></a>

</div>

This repo is the official implementation of **τ₀-VLA: a Hierarchical Robot
Foundation Model with World-Model-Guided Test-Time Computation**.

## News

- **[2026.09.20]** Community-contributed **LIBERO post-training and simulation evaluation**,
  thanks to [jian-77](https://github.com/jian-77) and liuyi. See the
  [LIBERO guide](configs/libero/README.md) for checkpoint availability, setup,
  and separately labeled author-reported and validation results.

- **[2026.07.27]** 🚀 We release the **τ₀-VLA** model
  [Paper](https://tau0-vla.github.io/tau0-vla.pdf),
  [Project Website](https://tau0-vla.github.io/), and
  [Hugging Face](https://huggingface.co/sii-research/tau-0-vla).

## Overview

τ₀-VLA is a hierarchical robot foundation model for long-horizon
manipulation. A memory-augmented high-level policy generates the next subtask
and uses world-model-guided test-time computation to search over alternatives
when additional reasoning is needed. A generalist low-level policy then
executes the selected subtask across robot embodiments.

The low-level policy combines a Qwen3.5 vision-language backbone with a
Mixture-of-Transformers action expert trained through conditional flow
matching. It uses a unified 40-dimensional state/action space and was trained
on 40,115 hours of heterogeneous real-world robot data with multimodal
co-training.

![Hierarchical τ₀-VLA pipeline](assets/method.png)

## Installation

The reference environment uses Python 3.11, CUDA 12.8, and PyTorch 2.7.1.

```bash
git clone git@github.com:sii-research/tau-0-vla.git
cd tau-0-vla
bash scripts/setup.sh
```

## Example data and post-training

[`example_data/`](example_data/README.md) contains a small AgiBot World subset
in LeRobot v3.0 format. The matching post-training recipe is under
[`configs/example_agibot_world_gong/`](configs/example_agibot_world_gong/README.md).

```bash
bash scripts/train.sh configs/example_agibot_world_gong/train.yaml \
    --model_name_or_path /path/to/tau-0-vla-checkpoint
```

For another dataset or robot, start from
[`configs/_template/`](configs/_template/README.md) and
[`src/tau0_vla/adapters/_template/`](src/tau0_vla/adapters/_template/README.md).
The repository also includes a complete LIBERO simulation recipe under
[`configs/libero/`](configs/libero/README.md).

## Serving and evaluation

Public hardware serving supports joint-control checkpoints only. The LIBERO
integration below uses a separate simulator-only EEF server and does not
change the hardware-serving contract.

Serve a post-trained joint-control checkpoint:

```bash
python -m deploy.server --model outputs/<run_name>
```

Run open-loop evaluation:

```bash
python deploy/openloop.py --ckpt outputs/<run_name> --no-plot
```

### LIBERO simulation evaluation

The community adaptation fine-tunes the released base model on LIBERO. The
**authors report** the following success rates from 50 rollouts per task:

| Spatial | Goal | Object | Long (`libero_10`) | Average |
| ---: | ---: | ---: | ---: | ---: |
| 97.40 | 98.20 | 98.80 | 95.00 | 97.35 |

Historical rollout logs were not available for independent verification.
**Independent coarse validation:** 193/200 successes (96.5%), five initial
states per task, with no evaluation exceptions. See the
[validation report](configs/libero/validation/2026-09-20.md) for per-suite
results and limitations; this does not establish an exact reproduction of 97.35%.

See the [LIBERO guide](configs/libero/README.md) for the complete checkpoint
contract, separate model/simulator environments, and commands for all four
suites. A publicly downloadable LIBERO fine-tuned checkpoint has **not yet
been verified**; the public base model alone is not the evaluated checkpoint.

See [`deploy/`](deploy/README.md) for the payload and action-order contracts.

## Repository layout

```text
src/tau0_vla/
├── adapters/    embodiment-specific data layouts and deployment I/O
├── data/        LeRobot loading, prompting, masking, and normalization
├── models/      Qwen3.5 backbone and flow-matching action expert
├── trainer/     post-training entry point
├── vlm/         multimodal collation and tokenization
└── utils/       logging and run specifications
configs/         reusable template and the AgiBot World example
deploy/          policy server and open-loop evaluation
example_data/    bundled AgiBot World subset
scripts/         setup, training, and normalization utilities
```

Additional documentation:

- [Dataset format](src/tau0_vla/data/DATASET_FORMAT.md)
- [Data pipeline](src/tau0_vla/data/README.md)
- [Robot adapters](src/tau0_vla/adapters/README.md)

## License

Code and model weights are released under the [Apache License 2.0](LICENSE).
The bundled example data follows the license described in
[`example_data/README.md`](example_data/README.md).
