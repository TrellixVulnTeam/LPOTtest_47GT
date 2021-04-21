Step-by-Step
============

This document is used to list steps of reproducing PyTorch BERT tuning zoo result.

> **Note**
>
> Dynamic Quantization is the recommended method for huggingface models. 

# Prerequisite

### 1. Installation

#### Python First

Recommend python 3.6 or higher version.

#### Install BERT model

```bash
cd examples/pytorch/huggingface_models
python setup.py install
```

> **Note**
>
> Please don't install public transformers package.

#### Install dependency

```shell
cd examples/test-classification
pip install -r requirements.txt
```

#### Install PyTorch
```
pip install torch -f https://download.pytorch.org/whl/torch_stable.html
```

### 2. Prepare pretrained model

Before use Intel® Low Precision Optimization Tool, you should fine tune the model to get pretrained model, You should also install the additional packages required by the examples:

#### BERT

* For BERT base and glue tasks(task name can be one of CoLA, SST-2, MRPC, STS-B, QQP, MNLI, QNLI, RTE, WNLI...)

```shell
export TASK_NAME=MRPC

python run_glue.py \
  --model_name_or_path bert-base-cased \
  --task_name $TASK_NAME \
  --do_train \
  --do_eval \
  --max_seq_length 128 \
  --per_device_train_batch_size 32 \
  --learning_rate 2e-5 \
  --num_train_epochs 3 \
  --output_dir /tmp/$TASK_NAME/
```

where task name can be one of CoLA, SST-2, MRPC, STS-B, QQP, MNLI, QNLI, RTE, WNLI.

The dev set results will be present within the text file 'eval_results.txt' in the specified output_dir. In case of MNLI, since there are two separate dev sets, matched and mismatched, there will be a separate output folder called '/tmp/MNLI-MM/' in addition to '/tmp/MNLI/'.

please refer to [BERT base scripts and instructions](examples/text-classification/README.md#PyTorch version).

* After fine tuning, you can get a checkpoint dir which include pretrained model, tokenizer and training arguments. This checkpoint dir will be used by lpot tuning as below.

# Run

### BERT glue task

```bash
export TASK_NAME=MRPC

python run_glue_tune.py \
    --model_name_or_path /path/to/checkpoint/dir \
    --task_name $TASK_NAME \
    --do_eval \
    --max_seq_length 128 \
    --per_gpu_eval_batch_size 8 \
    --no_cuda \
    --tune \
    --output_dir /path/to/checkpoint/dir
```

where task name can be one of CoLA, SST-2, MRPC, STS-B, QQP, MNLI, QNLI, RTE, WNLI.
Where output_dir is path of checkpoint which be created by fine tuning.

### seq2seq task

```bash
export TASK_NAME=translation_en_to_ro

python run_seq2seq_tune.py \
    --model_name_or_path /path/to/checkpoint/dir \
    --task $TASK_NAME \
    --data_dir /path/to/data/dir \
    --output_dir /path/to/checkpoint/dir \
    --overwrite_output_dir \
    --predict_with_generate \
    --tune \
    --tuned_checkpoint /path/to/checkpoint/dir
```

Where task name can be one of summarization_{summarization dataset name},translation_{language}\_to\_{language}.

Where summarization dataset can be one of xsum,billsum etc.

Where output_dir is path of checkpoint which be created by fine tuning.


Examples of enabling Intel® Low Precision Optimization Tool
============================================================

This is a tutorial of how to enable BERT model with Intel® Low Precision Optimization Tool.

# User Code Analysis

Intel® Low Precision Optimization Tool supports two usages:

1. User specifies fp32 'model', calibration dataset 'q_dataloader', evaluation dataset "eval_dataloader" and metrics in tuning.metrics field of model-specific yaml config file.
2. User specifies fp32 'model', calibration dataset 'q_dataloader' and a custom "eval_func" which encapsulates the evaluation dataset and metrics by itself.

As BERT's matricses are 'f1', 'acc_and_f1', mcc', 'spearmanr', 'acc', so customer should provide evaluation function 'eval_func', it's suitable for the second use case.

### Write Yaml config file

In examples directory, there is conf.yaml. We could remove most of items and only keep mandotory item for tuning.

```yaml
model:
  name: bert
  framework: pytorch

device: cpu

quantization:
  approach: post_training_dynamic_quant

tuning:
  accuracy_criterion:
    relative: 0.01
  exit_policy:
    timeout: 0
    max_trials: 300
  random_seed: 9527
```

Here we set accuracy target as tolerating 0.01 relative accuracy loss of baseline. The default tuning strategy is basic strategy. The timeout 0 means early stop as well as a tuning config meet accuracy target.

> **Note** : lpot does NOT support "mse" tuning strategy for pytorch framework

### Code Prepare

We just need update run_squad_tune.py and run_glue_tune.py like below

```python
if training_args.tune:
    def eval_func_for_lpot(model_tuned):
        trainer = Trainer(
            model=model_tuned,
            args=training_args,
            train_dataset=train_dataset,
            eval_dataset=eval_dataset,
            compute_metrics=compute_metrics,
            tokenizer=tokenizer,
            data_collator=data_collator,
        )
        result = trainer.evaluate(eval_dataset=eval_dataset)
        bert_task_acc_keys = ['eval_f1', 'eval_accuracy', 'mcc', 'spearmanr', 'acc']
        for key in bert_task_acc_keys:
            if key in result.keys():
                logger.info("Finally Eval {}:{}".format(key, result[key]))
                acc = result[key]
                break
        return acc
    from lpot.experimental import Quantization, common
    quantizer = Quantization("./conf.yaml")
    calibration_dataset = quantizer.dataset('bert', dataset=eval_dataset,
                                         task="classifier", model_type=config.model_type)
    quantizer.model = common.Model(model)
    quantizer.calib_dataloader = common.DataLoader(
        calibration_dataset, batch_size=training_args.per_device_eval_batch_size)
    quantizer.eval_func = eval_func_for_lpot
    q_model = quantizer()
    q_model.save(training_args.tuned_checkpoint)
    exit(0)
```

For seq2seq task,We need update run_seq2seq_tune.py like below

```python
if training_args.tune:
  def eval_func_for_lpot(model):
      trainer.model = model
      results = trainer.evaluate(
          eval_dataset=eval_dataset,metric_key_prefix="val", max_length=data_args.val_max_target_length, num_beams=data_args.eval_beams
      )
      assert data_args.task.startswith("summarization") or data_args.task.startswith("translation") , "data_args.task should startswith summarization or translation"
      if data_args.task.startswith("summarization"):
          task_metrics_keys = ['val_gen_len','val_loss','val_rouge1','val_rouge2','val_rougeL','val_rougeLsum','val_samples_per_second']
          rouge = 0
          for key in task_metrics_keys:
              if key in results.keys():
                  logger.info("Finally Eval {}:{}".format(key, results[key]))
                  if 'rouge' in key:
                      rouge += results[key]
          return rouge/4
      if data_args.task.startswith("translation"):
          task_metrics_keys = ['val_bleu','val_samples_per_second']
          for key in task_metrics_keys:
              if key in results.keys():
                  logger.info("Finally Eval {}:{}".format(key, results[key]))
                  if 'bleu' in key:
                      bleu = results[key]
          return bleu
  from lpot import Quantization, common
  quantizer = Quantization("./conf.yaml")
  quantizer.model = common.Model(model)
  quantizer.calib_dataloader = common.DataLoader(
                                          eval_dataset, 
                                          batch_size=training_args.eval_batch_size,
                                          collate_fn=Seq2SeqDataCollator_lpot(tokenizer, data_args, training_args.tpu_num_cores)
                                          )
  quantizer.eval_func = eval_func_for_lpot
  q_model = quantizer()
  q_model.save(training_args.tuned_checkpoint)
  exit(0)
```
# Original BERT README

Please refer to [BERT README](BERT_README.md).
