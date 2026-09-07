---
name: pytorch
description: PyTorch idioms and correctness pitfalls beyond beginner level. Consult before writing or reviewing PyTorch training, inference, or data-loading code.
---

# PyTorch

Scope: training-loop correctness, device handling, mixed precision, DataLoader tuning, reproducibility. Targets PyTorch 2.x. Does not cover torchvision, HuggingFace, Ray, or Lightning (separate skills).

## Do / Don't

### eval() and no_grad()

`model.eval()` only flips dropout/batchnorm. You still need `torch.no_grad()` to drop the autograd graph, or `torch.inference_mode()` (faster) for pure inference. Never wrap a training step in `no_grad()`.

```python
model.eval()
with torch.inference_mode():
    out = model(x)
```

### Device-agnostic code

Never hardcode `.cuda()`. Compute a device token once and use `.to(device)`.

```python
# Don't
if torch.cuda.is_available():
    model = model.cuda()

# Do
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = model.to(device)
```

### Mixed precision

Pair `torch.autocast` with a `GradScaler`. Order matters: `unscale_` before clipping grads, then `step`, then `update`. Save/load the scaler state or a resumed run is stuck on a stale scale.

```python
scaler = torch.amp.GradScaler("cuda")
for x, y in loader:
    with torch.autocast(device_type="cuda", dtype=torch.float16):
        loss = loss_fn(model(x), y)
    scaler.scale(loss).backward()
    scaler.unscale_(optimizer)          # before clip
    torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
    scaler.step(optimizer)
    scaler.update()
```

### DataLoader flags

One main-process shuffle, `num_workers>0`, `pin_memory=True` for CUDA, `persistent_workers=True` to avoid re-spawn cost each epoch, `prefetch_factor>1`. Don't call `.pin_memory().to(device, non_blocking=True)` on every small tensor — pin via the DataLoader.

```python
loader = DataLoader(
    ds,
    batch_size=64,
    shuffle=True,
    num_workers=8,
    persistent_workers=True,
    pin_memory=True,
)
```

For CUDA, `pin_memory=True` plus `tensor.to(device, non_blocking=True)` enables async host-to-device copies that overlap with compute — see the [pin_memory tutorial](https://docs.pytorch.org/tutorials/intermediate/pinmem_nonblock.html).

### Sync stalls

`.item()` and `.cpu()` force a CPU<->GPU sync that stalls the pipeline. Accumulate and log in chunks, not per iteration.

```python
# Don't
for i, batch in enumerate(loader):
    loss = train_step(batch)
    log(loss.item())          # sync every iteration

# Do — accumulate the loss tensor, sync once per chunk
total = torch.zeros((), device=device)
for i, batch in enumerate(loader):
    total = total + train_step(batch)
    if i % 100 == 0:
        log({"loss": (total / (i + 1)).item()})   # one sync per chunk
```

### zero_grad

Prefer `optimizer.zero_grad(set_to_none=True)` (default in recent versions). Note `.grad` becomes `None`, so guard manual `param.grad` reads.

```python
optimizer.zero_grad(set_to_none=True)
```

### In-place ops on the autograd graph

Don't mutate a tensor in place when it still needs gradients for backward (breaks the graph). Prefer out-of-place ops, e.g. `x = x + 1` over `x += 1` on leaf tensors.

### Reproducibility

Seeding torch, numpy, and random is not enough. Also set `torch.backends.cudnn.deterministic = True` and `torch.use_deterministic_algorithms(True)` (this throws on ops with no deterministic impl). Determinism costs speed and full bit-exactness across GPUs/releases isn't guaranteed.

```python
import torch, numpy as np, random
random.seed(42); np.random.seed(42); torch.manual_seed(42)
torch.backends.cudnn.deterministic = True
torch.use_deterministic_algorithms(True)
```

### torch.compile

Consider `torch.compile(model, ...)` for speed; works in train and eval; best with stable input shapes.

## Pitfalls: symptom -> cause -> fix

- `RuntimeError: a leaf Variable that requires grad is being used in an in-place operation` -> in-place op on a leaf/needed tensor -> switch to out-of-place.
- Loss is NaN from step 1 -> Amplifier/grad scale overflow or bad LR under autocast -> check scaler.step return / scale factor, lower LR, use bf16 on Ampere+.
- Training much slower than expected -> per-iteration `.item()` syncs, or tiny batches pinning individually -> batch syncs, let DataLoader pin.
- Non-deterministic across identical runs -> cuDNN benchmark / nondeterministic reductions -> set the deterministic flags above.

## Sources

- https://pytorch.org/docs/stable/generated/torch.no_grad.html
- https://pytorch.org/tutorials/recipes/recipes/amp_recipe.html
- https://pytorch.org/tutorials/intermediate/pinmem_nonblock.html
- https://pytorch.org/docs/stable/data.html
- https://pytorch.org/docs/stable/notes/randomness.html
- https://pytorch.org/docs/stable/generated/torch.optim.Optimizer.zero_grad.html
