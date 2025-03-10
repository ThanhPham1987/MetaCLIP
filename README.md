# Demystifying CLIP Data

This repository contains the code for paper [Demystifying CLIP Data](https://arxiv.org/abs/2309.16671) that formalizes CLIP's secret curation as an algorithm. The main contributions are:
  - Curating data from scratch without filtering via prior models (e.g., different from existing open source [efforts](https://arxiv.org/abs/2111.02114) that uses OpenAI's CLIP to have a **student** data or [adjust training setup](https://github.com/mlfoundations/open_clip) to improve trained CLIP performance);
  - Making training data more transparent, we released **training data distribution** over [metadata](metadata.json); 
  - Running algorithm in a data pipeline to scale **data pool** to the whole CommonCrawl (CC) w/ 300B+ image-text pairs: this ended w/ a distribution similar to OpenAI's WIT400M for data quality, which is **way more** important than training quantity (different from existing [open source efforts](https://arxiv.org/abs/2210.08402) or [ALGIN](https://arxiv.org/abs/2102.05918) that scales on quantity);
  - [OpenAI CLIP training setup](run_configs_400m.py) for controlled experiments.

As a summary:
  - We belive pretraining data should go for **maximally preserving signals and mitigates noises** for foundation data, instead of hard removal noises case-by-case w/ blackbox filters, that leads to unknown distribution;
  - The proposed algorithm is simpler and scalable to curate the whole Internet; 
  - We believe open-sourcing is not just about the trained model but also **pre-training data distribution** for model users.


```bibtex 
@inproceedings{xu2023metaclip,
   title={Demystifying CLIP Data},
   author={Hu Xu, Saining Xie, Xiaoqing Ellen Tan, Po-Yao Huang, Russell Howes, Vasu, Sharma, Shang-Wen Li, Gargi Ghosh, Luke Zettlemoyer and Christoph Feichtenhofer},
   journal={arXiv preprint arXiv:2309.16671},
   year={2023}
}
```

## Updates
* 09/28/2023: initial release.


## Quick Links

  - [Getting Started](#getting-started)
  - [Metadata](#metadata)
  - [Pre-trained Models](#pre-trained-models)
  - [Curation](#curation)
  - [Training](#training)
  - [Bugs or Questions?](#bugs-or-questions)
  - [Citation](#citation)
  - [Reference](#reference)


## Getting Started

This code is developed with minimal changes on [OpenCLIP](https://github.com/mlfoundations/open_clip), the following should give you the right pkgs to install requirements for OpenCLIP and `submitit=1.2.1` used by this repo:

```bash
conda create -n python=3.10 pytorch torchvision pytorch-cuda=11.7 tqdm ftfy braceexpand regex pandas submitit=1.2.1 \
    -c pytorch-nightly \
    -c nvidia \
    -c conda-forge \
    -c anaconda
```

## Metadata

CLIP uses 500,000 queries as [metadata](metadata.json) to align the training data to distribution over quality writing of Wikipedia/WordNet terms. This metadata also allows us to release training data distribution of a released model as **data card**.

### Pre-trained Models

We hack OpenCLIP to allow training on original CLIP model setup (w/ [ViT-B-16-quickgelu](src/open_clip/model_configs/ViT-B-16-quickgelu.json), [ViT-L-14-quickgelu](src/open_clip/model_configs/ViT-L-14-quickgelu.json) and [ViT-H-14-quickgelu](src/open_clip/model_configs/ViT-H-14-quickgelu.json)). Most OpenCLIP models use `nn.GELU` not `quickgelu` used by OpenAI CLIP. We hope this help research w/ controlled experiments in the "CLIP era of ImageNet".

```python 
import torch
from PIL import Image
import open_clip

model, _, preprocess = open_clip.create_model_and_transforms('ViT-B-32-quickgelu', pretrained='metaclip/b32_400m.pt')

image = preprocess(Image.open("CLIP.png")).unsqueeze(0)
text = open_clip.tokenize(["a diagram", "a dog", "a cat"])

with torch.no_grad():
    image_features = model.encode_image(image)
    text_features = model.encode_text(text)
    image_features /= image_features.norm(dim=-1, keepdim=True)
    text_features /= text_features.norm(dim=-1, keepdim=True)

    text_probs = (100.0 * image_features @ text_features.T).softmax(dim=-1)

print("Label probs:", text_probs)
```

|              Model              | Data Card | IN ZS Acc. |
|:--------------------------------|:---------:|:--------------:|
| [MetaCLIP B32 400M](https://dl.fbaipublicfiles.com/MMPT/metaclip/b32_400m.pt) | [data card](https://dl.fbaipublicfiles.com/MMPT/metaclip/datacard_400m.json) | 65.5 |
| [MetaCLIP B16 400M](https://dl.fbaipublicfiles.com/MMPT/metaclip/b16_400m.pt) | [data card](https://dl.fbaipublicfiles.com/MMPT/metaclip/datacard_400m.json) | 70.8 |
| [MetaCLIP L14 400M](https://dl.fbaipublicfiles.com/MMPT/metaclip/l14_400m.pt) | [data card](https://dl.fbaipublicfiles.com/MMPT/metaclip/datacard_400m.json) | 76.2 |
| [MetaCLIP B32 FullCC2.5B](https://dl.fbaipublicfiles.com/MMPT/metaclip/b32_fullcc2.5b.pt) | [data card](https://dl.fbaipublicfiles.com/MMPT/metaclip/datacard_fullcc2.5b.json) | 67.6 |
| [MetaCLIP B16 FullCC2.5B](https://dl.fbaipublicfiles.com/MMPT/metaclip/b16_fullcc2.5b.pt) | [data card](https://dl.fbaipublicfiles.com/MMPT/metaclip/datacard_fullcc2.5b.json) | 72.1 |
| [MetaCLIP L14 FullCC2.5B](https://dl.fbaipublicfiles.com/MMPT/metaclip/l14_fullcc2.5b.pt) | [data card](https://dl.fbaipublicfiles.com/MMPT/metaclip/datacard_fullcc2.5b.json) | 79.2 |
| [MetaCLIP H14 FullCC2.5B](https://dl.fbaipublicfiles.com/MMPT/metaclip/h14_fullcc2.5b.pt) | [data card](https://dl.fbaipublicfiles.com/MMPT/metaclip/datacard_fullcc2.5b.json) | 80.5 |
| MetaCLIP G14 FullCC2.5B | [data card](https://dl.fbaipublicfiles.com/MMPT/metaclip/datacard_fullcc2.5b.json) | ongoing |

## How to Curate ?

### I already have a (head distributed) dataset:
CLIP curation can still help as online balancing (Table 6 in the paper). We wrap CLIP curation in two key functions: [substring matching](metaclip/substr_matching.py) (recommened run offline) and [balancing](metaclip/balancing.py) (either offline or online, please check `metaclip.balancing:main`).

```python 
import json
import numpy as np
from metaclip.substr_matching import substr_matching
from metaclip.balancing import balance_sampling

with open("metadata.json") as f:
  metadata = json.load(f)
# entry counts for our 1.6B(pool) -> 400M(curated); please check balance_sampling:main and substr match and count on your own data.
with open("metaclip/entry_counts_400m.json") as f:
  entry_count_json = json.load(f)
entry_count = np.array([entry_count_json[entry] for entry in metadata], dtype=np.uint64)  # uint64 to be safe for scaling.

t = 20000
entry_count[entry_count < t] = t
entry_prob = t / entry_count

for text in ["jacksons chameleon", "battery plate"]:
  matched_entry_ids = substr_matching(text, metadata)
  if balance_sampling(matched_entry_ids, entry_prob):
    print(f"'{text}' curated")
```

### I want curate data from scratch:
We release a skeleton code for [sub-string matching](metaclip/cc_matching.py) from CommonCrawl WAT or WARC and [balancing](metaclip/balancing.py). Check [here](metaclip/README.md) for details.

## Training

```python 
python submitit_openclip.py b32_400m
```
Please config the corresponding `training_data` in `run_configs_400m.py`.


## Bugs or questions?

If you have any questions related to the code or the paper, feel free to email Hu Xu (`huxu@meta.com`).


## Citation

Please cite our paper if MetaCLIP helps in your work:

```bibtex 
@inproceedings{xu2023metaclip,
   title={Demystifying CLIP Data},
   author={Hu Xu, Saining Xie, Xiaoqing Ellen Tan, Po-Yao Huang, Russell Howes, Vasu, Sharma, Shang-Wen Li, Gargi Ghosh, Luke Zettlemoyer and Christoph Feichtenhofer},
   journal={arXiv preprint arXiv:2309.16671},
   year={2023}
}
```

## Reference

The codebase is developed from [OpenCLIP](https://github.com/mlfoundations/open_clip) w/ OpenAI CLIP training setup.

## TODO
- cross-json URL dedup in skeleton code;
- numpy implementation for matching and balancing;
- support online downloading;
- support OpenAI CLIP API;
- Huggingface integration;
- (welcome your use cases or suggestions to update this codebase regularly)


## License

The majority of MetaCLIP is licensed under CC-BY-NC, however portions of the project are available under separate license terms: open_clip is licensed under the https://github.com/mlfoundations/open_clip license.