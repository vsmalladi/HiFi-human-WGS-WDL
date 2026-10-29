# GPU support

Workflows can optionally run small variant calling on GPU-enabled nodes. Two inputs control this:

| Type | Name | Description | Notes |
| ---- | ---- | ----------- | ----- |
| Boolean | use_gpu | Use GPU for DeepVariant's `call_variants` step. | default = `false` |
| Boolean | use_parabricks_deepvariant | Use NVIDIA Parabricks DeepVariant instead of standard DeepVariant. Only takes effect when `use_gpu` is also `true`. | default = `false` |
| String | gpuType | Type of GPU/Accelerator to use. | Required when `use_gpu` is `true` on a cloud backend; must match backend. |
| String | gpuMachineType | GCP only. Fixed machine type for the GPU task, e.g. `g2-standard-8`. | Required for `nvidia-l4`, `nvidia-tesla-a100`, `nvidia-a100-80gb` and `nvidia-h100-80gb`; leave unset for `nvidia-tesla-v100`. Overrides the task's CPU and memory. |

## Where GPU is used

- **[DeepVariant](deepvariant.md)** — when `use_gpu=true` (and Parabricks is not selected), the `call_variants` step runs on 1 GPU/8 CPU/32GiB instead of 64 CPU/256GiB. `make_examples` and `postprocess_variants` always run on CPU.
- **[Parabricks DeepVariant](parabricks.md)** — when `use_gpu=true` and `use_parabricks_deepvariant=true`, small variant calling is replaced entirely by NVIDIA Parabricks' `pbrun deepvariant`, which runs on 4 GPUs/48 CPU/192GiB and requires GPUs with ≥16GB memory (e.g. V100 or A100). If `use_parabricks_deepvariant=true` but `use_gpu=false`, the workflow falls back to standard CPU DeepVariant.

## GPU Types

| Backend | GPU Type | Notes |
| ------- | -------- | ----- |
| GCP | `nvidia-l4`, `nvidia-tesla-a100`, `nvidia-a100-80gb`, `nvidia-h100-80gb`, `nvidia-tesla-v100` | See [GCP GPU recommendations](#gcp-gpu-recommendations). [GPU availability varies by zone.](https://cloud.google.com/compute/docs/gpus/gpu-regions-zones) `nvidia-tesla-t4` and `nvidia-tesla-p4` reach [end of support on August 1, 2027](https://docs.cloud.google.com/compute/docs/gpus) and are not recommended. `nvidia-tesla-v100` is available only in a limited set of zones. |
| AWS-HealthOmics | `nvidia-tesla-t4`, `nvidia-tesla-t4-a10g`, `nvidia-tesla-a10g`, `nvidia-t4-a10g-l4`, `nvidia-l4-a10g`, `nvidia-l4`, `nvidia-l40s` | Maps to HealthOmics instance families: `nvidia-tesla-t4`→G4, `nvidia-tesla-t4-a10g`→G4/G5, `nvidia-tesla-a10g`→G5, `nvidia-t4-a10g-l4`→G4/G5/G6, `nvidia-l4-a10g`→G5/G6, `nvidia-l4`→G6, `nvidia-l40s`→G6e. **Recommended: `nvidia-t4-a10g-l4`** — it spans the most instance families (G4/G5/G6) and is what we've tested on us-east-1. |
| Azure | | GPUs are not available on Azure. |
| HPC | | Depends on HPC and miniwdl/Cromwell/sprocket configuration. Reach out to [support@pacb.com](mailto:support@pacb.com?subject=WDL%20Workflows%20-%20GPU%20Support) |

## GCP GPU recommendations

`nvidia-l4` on a G2 machine is recommended for both DeepVariant and Parabricks: the G2 shapes match the resources each step asks for, and L4 has 24 GB per GPU (Parabricks needs ≥16 GB).

| Step | `gpuType` | `gpuMachineType` | Notes |
| ---- | --------- | ---------------- | ----- |
| DeepVariant (1 GPU/8 CPU/32 GiB) | `nvidia-l4` (recommended) | `g2-standard-8` | Matches the step's resources exactly. |
| DeepVariant | `nvidia-tesla-v100` | _unset_ | N1 custom machine; legacy option. |
| DeepVariant | `nvidia-tesla-a100` | `a2-highgpu-1g` | Larger than needed. |
| Parabricks (4 GPU/48 CPU/192 GiB) | `nvidia-l4` (recommended) | `g2-standard-48` | 4x L4; matches the step's resources exactly. |
| Parabricks | `nvidia-tesla-a100` | `a2-highgpu-4g` | 4x A100 40 GB. |
| Parabricks | `nvidia-a100-80gb` | `a2-ultragpu-4g` | 4x A100 80 GB. |
| DeepVariant | `nvidia-h100-80gb` | `a3-highgpu-1g` | 1x H100; costly, use only if A100/L4 capacity is unavailable. |
| Parabricks | `nvidia-h100-80gb` | `a3-highgpu-4g` | 4x H100 on 104 vCPU/936 GB; costly. |

Newer GPUs (H200, B200 and GB200) are not recommended for these steps; they have not been tested with the DeepVariant and Parabricks containers.

Machine shapes were checked against the [GCP GPU machine types](https://docs.cloud.google.com/compute/docs/gpus) page: `g2-standard-8` (8 vCPU/32 GB/1 L4), `g2-standard-48` (48 vCPU/192 GB/4 L4), `a2-highgpu-1g` (12 vCPU/85 GB/1 A100 40 GB), `a2-highgpu-4g` (48 vCPU/340 GB/4 A100 40 GB), `a2-ultragpu-4g` (48 vCPU/680 GB/4 A100 80 GB), `a3-highgpu-1g` (26 vCPU/234 GB/1 H100) and `a3-highgpu-4g` (104 vCPU/936 GB/4 H100).
