---
name: lightning
description: PyTorch Lightning module and Trainer best practices. Consult before writing or reviewing Lightning training code, datamodules, callbacks, or multi-device configs.
---

# PyTorch Lightning

Scope: LightningModule structure, letting the Trainer handle devices, callbacks, Trainer flags, reproducibility, manual optimization. Targets Lightning 2.x. Lightning manages device placement and the step loop — don't fight it.

## Do / Don't

### Structure

All training logic lives in `LightningModule` hooks: `training_step` (return loss), `validation_step` (+ `self.log("val_loss", x)`), `configure_optimizers` (return optimizer + optional LR schedulers). Data goes in `train_dataloader`/`val_dataloader` or a `LightningDataModule`.

```python
class LitNet(LightningModule):
    def training_step(self, batch, batch_idx):
        x, y = batch
        loss = self.criterion(self(x), y)
        self.log("train_loss", loss)
        return loss

    def configure_optimizers(self):
        return torch.optim.Adam(self.parameters(), lr=1e-3)
```

### No manual .cuda()/.to(device)

Don't call `.cuda()` or `.to(device)` inside the module, datamodule, or step methods — it breaks DDP, precision, and accelerator handling. Return CPU tensors from dataloaders; Lightning moves them.

```python
# Do — declarative device config
Trainer(accelerator="auto", devices="auto")
Trainer(accelerator="gpu", devices=4, strategy="ddp")
```

### Callbacks over hardcoded logic

Use callbacks for cross-cutting behavior instead of baking it into the module: `ModelCheckpoint`, `EarlyStopping`, `ModelSummary`, `LearningRateMonitor`.

```python
trainer = Trainer(callbacks=[
    ModelCheckpoint(monitor="val_loss", mode="min", save_top_k=1),
    EarlyStopping(monitor="val_loss", patience=3),
])
```

### Mixed precision

Use `Trainer(precision=...)` — Lightning wires `torch.autocast` and the `GradScaler` for you. `"16-mixed"` (fp16) needs a scaler; `"bf16-mixed"` does not (Ampere+). Don't add your own `torch.autocast`/`GradScaler` in step methods under automatic optimization.

```python
trainer = Trainer(accelerator="gpu", devices=4, precision="16-mixed")
```

Under manual optimization (`self.automatic_optimization = False`), Lightning no longer manages AMP — run `torch.autocast` and `torch.amp.GradScaler` yourself, exactly as in the `pytorch` skill.

### Flags that fix real bugs

`gradient_clip_val` + `gradient_clip_algorithm="norm"|"value"`, `max_epochs`, `deterministic=True`, `fast_dev_run=True` to smoke-test a loop, `logger=...` / `enable_checkpointing=False`.

### Reproducibility

`seed_everything(42, workers=True)` (seeds numpy/torch/random and derives unique per-worker seeds) + `Trainer(deterministic=True)`.

### Manual optimization

Research loops: set `self.automatic_optimization = False`, then `opt = self.optimizers()` (or `use_pl_optimizer=True`), `opt.zero_grad()`, `self.manual_backward(loss)`, `opt.step()`.

### predict_step ownership

In `predict_step` (prediction mode), you own `model.eval()` and the `no_grad`/`inference_mode` context — Lightning does not add them there (it does manage grad/eval in the val/test loops).

## Pitfalls: symptom -> cause -> fix

- Doubled optimizer steps or broken gradient clipping -> calling `optimizer.step()`/`zero_grad()` yourself while `automatic_optimization=True` -> let Lightning orchestrate, or switch to manual optimization.
- Error across GPUs / wrong device behavior -> `.cuda()`/`.to(device)` inside step methods or datamodule -> remove; let Trainer place devices.
- `--gpus` unknown argument -> referencing the removed legacy CLI flag -> use `--devices`/`accelerator`.
- Slow validation with per-batch `.item()` -> manual sync — use `self.log` (it aggregates and syncs in DDP correctly).

## Sources

- https://lightning.ai/docs/pytorch/stable/common/lightning_module.html
- https://lightning.ai/docs/pytorch/stable/common/trainer.html
