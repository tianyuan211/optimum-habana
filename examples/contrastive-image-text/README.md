<!---
Copyright 2022 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# VisionTextDualEncoder and CLIP-like model training examples

This folder contains two examples:

1. The first one showcases how to train a CLIP-like vision-text dual encoder model using pre-trained vision and text encoders. The model is inspired by [CLIP](https://openai.com/blog/clip/), introduced by Alec Radford et al. The idea is to train a vision encoder and a text encoder jointly to project the representation of images and their captions into the same embedding space, such that the caption embeddings are located near the embeddings of the images they describe.
2. The second one showcases how to train a [BridgeTower](https://arxiv.org/abs/2206.08657) model. This model contains bridges between the text and vision encoders that are linked to a cross-modal encoder. This enables effective bottom-up cross-modal alignment between visual and textual representations at different semantic levels in the cross-modal encoder.

Such models can be used for natural language image search and potentially zero-shot image classification.

## Requirements

First, you should install the requirements:
```bash
pip install -r requirements.txt
```

## Download COCO dataset (2017)
This example uses COCO dataset (2017) through a custom dataset script, which requires users to manually download the
COCO dataset before training.

```bash
mkdir data
cd data
wget http://images.cocodataset.org/zips/train2017.zip
wget http://images.cocodataset.org/zips/val2017.zip
wget http://images.cocodataset.org/zips/test2017.zip
wget http://images.cocodataset.org/annotations/annotations_trainval2017.zip
wget http://images.cocodataset.org/annotations/image_info_test2017.zip
cd ..
```

Having downloaded COCO dataset manually you should be able to load with the `ydshieh/coco_dataset_script` dataset loading script:

```python
import os
import datasets

COCO_DIR = os.path.join(os.getcwd(), "data")
ds = datasets.load_dataset("ydshieh/coco_dataset_script", "2017", data_dir=COCO_DIR)
```

## CLIP-like models

Here is how to run the `run_clip.py` script for training CLIP-like models.


### Create a model from a vision encoder model and a text encoder model
Next, we create a [VisionTextDualEncoderModel](https://huggingface.co/docs/transformers/model_doc/vision-text-dual-encoder#visiontextdualencoder).
The `VisionTextDualEncoderModel` class lets you load any vision and text encoder model to create a dual encoder.
Here is an example of how to load the model using pre-trained vision and text models.

```python
from transformers import (
    VisionTextDualEncoderModel,
    VisionTextDualEncoderProcessor,
    AutoTokenizer,
    AutoImageProcessor
)

model = VisionTextDualEncoderModel.from_vision_text_pretrained(
    "openai/clip-vit-base-patch32", "roberta-base"
)

tokenizer = AutoTokenizer.from_pretrained("roberta-base")
image_processor = AutoImageProcessor.from_pretrained("openai/clip-vit-base-patch32")
processor = VisionTextDualEncoderProcessor(image_processor, tokenizer)

# save the model and processor
model.save_pretrained("clip-roberta")
processor.save_pretrained("clip-roberta")
```

This loads both the text and vision encoders using pre-trained weights, the projection layers are randomly
initialized except for CLIP's vision model. If you use CLIP to initialize the vision model then the vision projection weights are also
loaded using the pre-trained weights.

### Single-HPU training

Finally, we can run the example script to train the model.
Run the following command for single-device training:

```bash
python run_clip.py \
    --output_dir ./clip-roberta-finetuned \
    --model_name_or_path ./clip-roberta \
    --data_dir $PWD/data \
    --dataset_name ydshieh/coco_dataset_script \
    --dataset_config_name=2017 \
    --image_column image_path \
    --caption_column caption \
    --remove_unused_columns=False \
    --do_train  --do_eval \
    --per_device_train_batch_size="512" \
    --per_device_eval_batch_size="64" \
    --learning_rate="5e-5" --warmup_steps="0" --weight_decay 0.1 \
    --overwrite_output_dir \
    --save_strategy epoch \
    --use_habana \
    --gaudi_config_name Habana/clip \
    --throughput_warmup_steps 3 \
    --dataloader_num_workers 16 \
    --sdp_on_bf16 \
    --bf16 \
    --trust_remote_code \
    --torch_compile_backend=hpu_backend \
    --torch_compile
```


### Multi-HPU training

Run the following command for distributed training:

```bash
PT_ENABLE_INT64_SUPPORT=1 \
python3 ../gaudi_spawn.py --world_size 8 --use_mpi run_clip.py \
    --output_dir=/tmp/clip_roberta \
    --model_name_or_path=./clip-roberta \
    --data_dir $PWD/data \
    --dataset_name ydshieh/coco_dataset_script \
    --dataset_config_name 2017 \
    --image_column image_path \
    --caption_column caption \
    --remove_unused_columns=False \
    --do_train --do_eval \
    --mediapipe_dataloader \
    --per_device_train_batch_size="64" \
    --per_device_eval_batch_size="64" \
    --learning_rate="5e-5" --warmup_steps="0" --weight_decay 0.1 \
    --overwrite_output_dir \
    --use_habana \
    --use_lazy_mode=False \
    --gaudi_config_name="Habana/clip" \
    --throughput_warmup_steps=3 \
    --save_strategy="no" \
    --dataloader_num_workers=2 \
    --use_hpu_graphs \
    --max_steps=100 \
    --torch_compile_backend=hpu_backend \
    --torch_compile \
    --logging_nan_inf_filter \
    --trust_remote_code
```

> [!NOTE]
> Please note `--mediapipe_dataloader` is not supported on Gaudi1.


### DeepSpeed

You can check the [DeepSpeed](https://github.com/huggingface/optimum-habana/tree/main/examples#deepspeed) section in Optimum Habana examples for how to run DeepSpeed.
You can also look at the [documentation](https://huggingface.co/docs/optimum/habana/usage_guides/deepspeed) for more information about how to use DeepSpeed in Optimum Habana.


## BridgeTower

For training BridgeTower, you need to run the `run_bridgetower.py` script.
For instance, to reproduce the results presented in [this blog post](https://huggingface.co/blog/bridgetower), you should run:

```bash
PT_HPU_LAZY_MODE=1 python ../gaudi_spawn.py --use_mpi --world_size 8 run_bridgetower.py \
  --output_dir /tmp/bridgetower-test \
  --model_name_or_path BridgeTower/bridgetower-large-itm-mlm-itc \
  --dataset_name jmhessel/newyorker_caption_contest --dataset_config_name matching \
  --dataset_revision 3c6c4f6c0ff7e902833d3afa5f8f3875c2b036e6 \
  --image_column image --caption_column image_description \
  --remove_unused_columns=False \
  --do_train --do_eval --do_predict \
  --per_device_train_batch_size="40" --per_device_eval_batch_size="16" \
  --num_train_epochs 5 \
  --learning_rate="1e-5" \
  --overwrite_output_dir \
  --save_strategy no \
  --use_habana --use_lazy_mode --use_hpu_graphs_for_inference --gaudi_config_name Habana/clip \
  --throughput_warmup_steps 3 \
  --logging_steps 10 \
  --dataloader_num_workers 1 \
  --mediapipe_dataloader \
  --trust_remote_code \
  --sdp_on_bf16
```

> [!NOTE]
> Please note `--mediapipe_dataloader` is not supported on Gaudi1.


## Inference

To run only inference, you can start from the commands above and you just have to remove the training-only arguments such as `--do_train`, `--per_device_train_batch_size`, `--num_train_epochs`, etc...

For instance, you can run inference with CLIP on COCO on 1 Gaudi card with the following command:
```bash
PT_HPU_LAZY_MODE=1 python run_clip.py \
    --output_dir ./clip-roberta-finetuned \
    --model_name_or_path ./clip-roberta \
    --data_dir $PWD/data \
    --dataset_name ydshieh/coco_dataset_script \
    --dataset_config_name=2017 \
    --image_column image_path \
    --caption_column caption \
    --remove_unused_columns=False \
    --do_eval \
    --per_device_eval_batch_size="64" \
    --overwrite_output_dir \
    --use_habana \
    --use_lazy_mode \
    --use_hpu_graphs_for_inference \
    --gaudi_config_name Habana/clip \
    --bf16 \
    --sdp_on_bf16 \
    --mediapipe_dataloader \
    --trust_remote_code
```

> [!NOTE]
> Please note `--mediapipe_dataloader` is not supported on Gaudi1.
