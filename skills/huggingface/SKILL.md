---
name: huggingface
description: HuggingFace transformers, datasets, accelerate, and PEFT best practices, including version-specific breaks. Consult before training, fine-tuning, loading, or running inference with transformers models.
---

# HuggingFace

Scope: `transformers` Trainer, tokenization, `datasets` streaming, `accelerate` big-model loading, PEFT/LoRA. Targets Transformers v5.x. Version-specific: v4 -> v5 renamed the Trainer `tokenizer=` kwarg to `processing_class=`. Pin versions in any reproducible workflow.

## Do / Don't

### Trainer with processing_class

In Transformers v5, pass the tokenizer as `processing_class=`. `tokenizer=` is gone.

```python
trainer = Trainer(
    model,
    args,
    train_dataset=ds["train"],
    eval_dataset=ds["test"],
    processing_class=tokenizer,
    data_collator=DataCollatorForSeq2Seq(tokenizer, model=model),
)
```

### TrainingArguments gotchas

`remove_unused_columns` defaults to `True` and silently drops custom features you pass to forward. Use `gradient_accumulation_steps`, `fp16`/`bf16` (bf16 preferred on Ampere+), `optim="adamw_torch"`.

```python
args = TrainingArguments(
    output_dir="./out",
    per_device_train_batch_size=8,
    gradient_accumulation_steps=4,
    bf16=True,
    remove_unused_columns=False,
)
```

### Tokenization

Set explicit padding/truncation: `padding="max_length", truncation=True, max_length=model_max_len` (or `padding="longest"`). Use `truncation="only_first"/"only_second"/"longest_first"` to truncate one side. For decoder-only models set `tokenizer.pad_token = tokenizer.eos_token` and `padding_side="left"` for generation. Rely on collators to mask label padding with `-100`.

### Streaming huge datasets

Don't `load_dataset` a multi-GB corpus without `streaming=True` — it downloads everything. Streaming yields an `IterableDataset` that pulls only what you consume.

```python
ds = load_dataset("HuggingFaceFW/fineweb", split="train", streaming=True)
for ex in itertools.islice(ds, 1000):
    process(ex)
```

### Accelerate device placement

Let Accelerate place devices: `accelerator.prepare(model, optimizer, dataloader)`, use `accelerator.device`. Never `model.to("cuda")` inside Accelerate/Trainer code.

```python
accelerator = Accelerator()
model, opt, dl = accelerator.prepare(model, opt, dl)
```

### Big-model loading

Use `device_map="auto"`, `low_cpu_mem_usage`, `no_split_module_classes=[...]` so chunked modules aren't split. For 4-bit LoRA base:

```python
bnb = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=False,
)
model = AutoModel.from_pretrained(path, device_map="auto", quantization_config=bnb)
```

Load checkpoints with `dtype="auto"` — default loads fp32 and doubles memory.

### PEFT / LoRA

`LoraConfig(r=16, lora_alpha=16, lora_dropout=0.1, bias="none", target_modules=[...], modules_to_save=[...])`. Verify with `model.print_trainable_parameters()` (sanity: ~0.7% trainable). Merge with `merge_and_unload()` only when inference latency matters and the base is float — merging into int8/4bit bases is unsupported. Keep base + adapter separately if you need to unload adapters later.

### Version pinning

Transformers ships breaking changes across point releases (e.g., v4 -> v5 tokenizer -> processing_class). Pin `transformers==x.y.z` plus a matching tokenizer, and re-audit code on any upgrade.

## Pitfalls: symptom -> cause -> fix

- `TypeError: Trainer.__init__() got an unexpected keyword argument 'tokenizer'` -> v5 rename -> use `processing_class=`.
- Failing forward / missing features -> `remove_unused_columns=True` dropped your columns -> set it to False.
- OOM / disk full on a big corpus -> non-streaming load -> add `streaming=True`.
- Merged checkpoint can't be reloaded as adapters -> merged with a quantized base -> don't merge; keep base + adapter.

## Sources

- https://huggingface.co/docs/transformers/main/en/main_classes/trainer
- https://huggingface.co/docs/transformers/main/en/training
- https://huggingface.co/docs/transformers/main/en/pad_truncation
- https://huggingface.co/docs/datasets/stream
- https://huggingface.co/docs/accelerate/usage_guides/big_modeling
- https://huggingface.co/docs/peft/main/en/developer_guides/lora
