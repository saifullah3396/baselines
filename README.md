This is the repository that provide tools to download data, reproduce the baseline results and evaluation.



## What can you achieve with this guide
Based on this repository, you may be able to:

1. download data for benchmark in a unified format.
2. process it and prepare for running baselines.
3. run all the baselines.
4. evaluate already trained baseline models. 

## Install benchmark-related repositories
Start the container:
```bash
sudo userdocker run nvcr.io/nvidia/pytorch:20.12-py3
```

Clone the repo with:
```bash
git clone git@github.com:due-benchmark/baselines.git
```

Install the requirements:
```bash
pip install -e .
```

# 1. Download datasets and the base model
The datasets are re-hosted on the https://duebenchmark.com/data and can be downloaded from there. 
Moreover, since the baselines are finetuned based on the T5 model, you need to download the original model. 
Again it is re-hosted at https://duebenchmark.com/data. 
Please place it into the `due_benchmark_data` directory after downloading.

# 2. Process datasets into memmaps (binarization)
In order to process datasets into memmaps, set the directory `downloaded_data_path` to downloaded data,
 set `memmap_directory` to a new directory that will store binarized datas, and use the following script:
```bash
./create_memmaps.sh
```
# TODO: dopisać resztę
# 3. Run baseline trainings
Single training can be started with the following command, assuming `out_dir` is set as an output for the trained model's checkpoints and generated outputs.
Additionally, set `datas` to any of the previously generated datasets (e.g., to `DeepForm`). 
```bash
python benchmarker/cli/l5/train.py \
    --model_name_or_path ${downloaded_data_path}/t5-base \
    --relative_bias_args="[{\"type\":\"1d\"}]" \
    --dropout_rate 0.15 \
    --model_type=t5 \
    --output_dir ${out_dir} \
    --data_dir ${memmap_directory}/${datas}_memmap/train \
    --val_data_dir ${memmap_directory}/${datas}_memmap/dev \
    --test_data_dir ${memmap_directory}/${datas}_memmap/test \
    --gpus 1 \
    --max_epochs 30 \
    --train_batch_size 1 \
    --eval_batch_size 2 \
    --overwrite_output_dir \
    --accumulate_grad_batches 64 \
    --max_source_length 1024 \
    --max_target_length 256 \
    --eval_max_gen_length 16 \
    --learning_rate 2e-4 \
    --lr_scheduler constant \
    --warmup_steps 100 \
    --trim_batches \ 
    --do_train \
    --do_predict \ 
    --additional_data_fields doc_id label_name \
    --early_stopping_patience 20 \
    --segment_levels tokens pages \
    --optimizer adamw \
    --weight_decay 1e-5 \
    --adam_epsilon 1e-8 \
    --num_workers 4 \
    --val_check_interval 1
```
The models presented in the paper differs only in two places. The first is the choice of `--relative_bias_args`.
T5 uses	`[{'type': '1d'}]` whereas both `+2D` and `+DALL-E` use 	`[{'type': '1d'}, {'type': 'horizontal'}, {'type': 'vertical'}]`

Moreover `+DALL-E` had `--context_embeddings` set to `[{'dimension': 1024, 'use_position_bias': False, 'embedding_type': 'discrete_vae', 'pretrained_path': '', 'image_width': 256, 'image_height': 256}]`

# 4. Evaluate
## 4.1 Convert output to the submission file
In order to compare two files (generated by the model with the provided library and the gold-truth answers), one has to convert the generated output into a format that can be directly compared with `documents.jsonl`.
Please use:
```bash
python to_submission_file.py ${downloaded_data_path} ${out_dir}
```

## 4.2 Evaluate reproduced models
Finally outputs can be evaluated using the provided evaluator. 
First, get back into main directory, where this README.md is placed and install it by ```cd due_evaluator-master && pip install -r requirement```
And run:
```bash
python due_evaluator --out-files baselines/test_generations.jsonl --reference ${downloaded_data_path}/DeepForm
```

## 4.3 Evaluate baseline outputs
We provide an examples of outputs generated by our baseline (DeepForm). They should be processed with:
```bash
python benchmarker-code/to_submission_file.py ${downloaded_data_path}/model_outputs_example ${downloaded_data_path}
python due_evaluator --out-files ./benchmarker/cli/l5/baselines/test_generations.txt.jsonl --reference ${downloaded_data_path}/DeepForm/test/document.jsonl
```

The expected output should be:
```bash
       Label       F1  Precision   Recall
  advertiser 0.512909   0.513793 0.512027
contract_num 0.778761   0.780142 0.777385
 flight_from 0.794376   0.795775 0.792982
   flight_to 0.804921   0.806338 0.803509
gross_amount 0.355476   0.356115 0.354839
         ALL 0.649771   0.650917 0.648630
```
