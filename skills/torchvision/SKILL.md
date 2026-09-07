---
name: torchvision
description: torchvision transforms and model loading best practices. Consult before building image-classification, object-detection, or any vision data pipeline, and when loading pretrained vision models.
---

# torchvision

Scope: the v2 transforms API, pretrained weights enums, `datasets.ImageFolder`. Targets torchvision 0.28.x. The legacy `torchvision.transforms` API is deprecated; do not write new code against it.

## Do / Don't

### Use the v2 transforms API

`torchvision.transforms.v2` handles images, boxes, masks, and keypoints in one pass and is scriptable. Canonical pipeline:

```python
from torchvision import transforms as v2

transforms = v2.Compose([
    v2.ToImage(),                             # PIL / numpy / tensor to Image
    v2.ToDtype(torch.float32, scale=True),    # rescale to [0, 1]
    v2.Resize((224, 224), antialias=True),    # or v2.RandomResizedCrop
    v2.Normalize(mean=[0.485, 0.456, 0.406],
                 std=[0.229, 0.224, 0.225]),
])
```

Don't mix legacy ops with v2 ops in one pipeline — the tensor formats (HWC vs CHW, uint8 vs float) conflict and cause subtle bugs. Pick one API; v2.

### Pretrained weights via enums

`pretrained=True` is deprecated and slated for removal. Use the enum API, and fetch the model's own preprocessing via `Weights.DEFAULT.transforms()` instead of hand-rolling resize/interpolation.

```python
# Do
from torchvision.models import ResNet50_Weights, resnet50
weights = ResNet50_Weights.DEFAULT
model = resnet50(weights=weights)          # or weights="DEFAULT"
preprocess = weights.transforms()          # exact preprocessing this model expects
```

### Normalization ordering

`v2.Normalize` assumes a [0, 1] float tensor — scale first with `ToDtype(scale=True)` before normalizing. Don't feed uint8 or numpy arrays straight into Normalize.

### antialias

Pass `antialias=True` on `v2.Resize` / `v2.RandomResizedCrop` when downsampling; the default changed and quality/accuracy drop without it.

### ImageFolder

`datasets.ImageFolder` returns `(PIL.Image, class_index)`, with class indices sorted alphabetically by folder name; it's lazy (reads on `__getitem__`), so use DataLoader `num_workers` for parallel I/O. Pass it a `transform=` callable (wrap the v2 Compose).

### Augmentations in eval

Don't apply Random* augmentations at eval; eval is typically resize + normalize only.

## Pitfalls: symptom -> cause -> fix

- Output images look wrong/dark -> feeding uint8 into v2.Normalize or skipping the scale step -> add `ToDtype(scale=True)` before Normalize.
- `AttributeError: ... no attribute 'ToImage'` or mixed formats -> accidentally using legacy transforms in a v2 pipeline -> standardize on v2.Compose throughout.
- Validation accuracy suddenly lower than expected -> downsampling without antialias -> add `antialias=True`.

## Sources

- https://pytorch.org/vision/stable/models.html
- https://pytorch.org/vision/stable/transforms.html
- https://pytorch.org/vision/stable/generated/torchvision.transforms.v2.Compose.html
- https://pytorch.org/vision/stable/generated/torchvision.datasets.ImageFolder.html
