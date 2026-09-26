# llamacpp-fastmtp

CI-built `llama-server` binaries with the
[HauhauCS FastMTP](https://huggingface.co/HauhauCS/Qwen3.8-27B-Uncensored-HauhauCS-Aggressive-MTP-GGUF)
patch applied — for serving Qwen3.8-27B (uncensored) on Kaggle's free T4 GPUs with
MTP speculative decoding via the FastMTP sidecar draft model.

## Why

Upstream llama.cpp supports the model's *embedded* NextN head
(`--spec-type draft-mtp`), but the HauhauCS FastMTP sidecar draft (+35% decode
over embedded MTP) needs a small patch that isn't merged upstream. Kaggle
sessions can't compile CUDA efficiently (4 vCPU, ~25 min), so this repo builds
once on GitHub Actions and serves a tarball.

## Build

Triggered on push to `main` or manual dispatch. The workflow:

1. checks out `llama.cpp` at the commit pinned by the FastMTP provenance file
   (`4df29be4f4c3673f428170fda944a5b19f743bb8`)
2. downloads the patch from Hugging Face and verifies its sha256 against the
   pinned value before applying
3. builds `llama-server` with CUDA 12.8 for sm_75 (Turing T4) on ubuntu:22.04
   (glibc match with Kaggle images)
4. bundles `bin/llama-server` + CUDA runtime libs (`lib/`) into
   `llama-fastmtp-sm75.tar.gz` and attaches it to a GitHub Release

## Use

```bash
curl -L -o llama.tar.gz https://github.com/nkhanhtrn/llamacpp-fastmtp/releases/latest/download/llama-fastmtp-sm75.tar.gz
tar xzf llama.tar.gz        # -> bin/llama-server, lib/*.so*
LD_LIBRARY_PATH=lib ./bin/llama-server -m model.gguf ...
```

Model files come from
`HauhauCS/Qwen3.8-27B-Uncensored-HauhauCS-Aggressive-MTP-GGUF` on HF:

- target quant, e.g. `Qwen3.8-27B-Uncensored-HauhauCS-Aggressive-Q4_K_P.gguf` (17.9 GB)
- `mmproj-Qwen3.8-27B-Uncensored-HauhauCS-Aggressive-BF16.gguf` (vision, 0.9 GB)
- `Qwen3.8-27B-Uncensored-HauhauCS-Aggressive-FastMTP-32K.gguf` (draft, 0.9 GB)

FastMTP serving flags: `--spec-draft-model <draft> --spec-draft-ngl all
--spec-type draft-mtp --spec-draft-n-max 3 --spec-draft-p-min 0`.

## Integrity

- patch sha256 `981285400b59dc45cf99936b6ff66d4b3aa0f1b532f85fa51418cb407e51d615`
  (from the release's signed `FastMTP-PROVENANCE.json`)
- base commit pinned by the same provenance file
- asset sha256 printed in the CI log; verify against it if paranoid
