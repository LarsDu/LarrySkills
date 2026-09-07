---
name: ray
description: Ray core, Ray Data, and Ray Tune best practices. Consult before scaling Python compute with Ray, building distributed data pipelines, or running hyperparameter tuning.
---

# Ray

Scope: tasks vs actors, `ray.get`/`ray.wait` patterns, ObjectRef passing, Ray Data transforms, Ray Tune's modern `Tuner` API, resource allocation. Targets Ray 2.x.

## Do / Don't

### Batch ray.get

Scheduling tasks and fetching results should be decoupled. A `ray.get` per iteration serializes execution and kills parallelism.

```python
# Don't
results = [ray.get(task.remote(i)) for i in range(100)]

# Do — schedule all, fetch once
refs = [task.remote(i) for i in range(100)]
results = ray.get(refs)

# Incremental results: use ray.wait for first-available
done, pending = ray.wait(refs, num_returns=10)
```

### Tasks vs actors

Prefer tasks for stateless compute. Use actors (`@ray.remote class`, `c.method.remote()`) only when state must persist across calls: caches, preloaded state (a loaded model), counters, services.

### Minimize round trips

Pass ObjectRefs between tasks instead of `ray.get`-fetching, converting, and re-putting. `ray.get` only where the driver needs concrete values — unnecessary `ray.get` harms performance.

### Ray Data

Ingest with `ray.data.read_*`, transform with `.map_batches(...)` (vectorized, GPU-friendly) rather than python `.map`. Materialize for training via `iter_torch_batches()` / `streaming_split(n)`, not by pulling everything into memory with `to_pandas()` / `take_all()`.

```python
ds = ray.data.read_parquet("s3://bucket/part/")
ds = ds.map_batches(preprocess_batch, batch_size=256, num_gpus=1)
for batch in ds.iter_torch_batches(batch_size=64):
    train_step(batch)
```

### Ray Tune: modern API

Use `tune.Tuner`, not the old `tune.run`.

```python
tuner = tune.Tuner(
    trainable,
    param_space={
        "lr": tune.loguniform(1e-4, 1e-1),
        "bs": tune.grid_search([32, 64, 128]),
    },
    tune_config=tune.TuneConfig(
        num_samples=10,
        metric="loss",
        mode="min",
        # search_alg=..., scheduler=...
    ),
    run_config=tune.RunConfig(stop={"training_iteration": 20}),
)
results = tuner.fit()
```

Search-space API: `grid_search`, `loguniform`, `qloguniform`, `randint`, `qrandint`. Control budget via `num_samples` or `time_budget_s` (`num_samples=-1` for pure time budget).

### Resource allocation

Allocate precisely: `@ray.remote(num_gpus=1)` for GPU actors, `num_cpus=0` for a pure-GPU actor. Over-requesting (e.g. `num_cpus=100`) leaves tasks pending forever.

## Pitfalls: symptom -> cause -> fix

- Everything runs serially, no speedup -> `ray.get` inside the scheduling loop -> batch with `refs` then one `ray.get(refs)`.
- Tasks stuck in PENDING indefinitely -> over-requested resources that will never free -> ensure `num_cpus`/`num_gpus` match actual usage.
- `RuntimeError: Ray has not been started` from inside a task -> calling `ray.init()` in a remote func/actor -> init once in the driver only.
- OOM during data ingestion -> `to_pandas()` / `take_all()` pulling whole dataset into memory -> use `iter_torch_batches()` / `map_batches`.

## Sources

- https://docs.ray.io/en/latest/ray-core/patterns/ray-get-loop.html
- https://docs.ray.io/en/latest/ray-core/patterns/unnecessary-ray-get.html
- https://docs.ray.io/en/latest/ray-core/actors.html
- https://docs.ray.io/en/latest/data/loading-data.html
- https://docs.ray.io/en/latest/data/api/dataset.html
- https://docs.ray.io/en/latest/tune/key-concepts.html
