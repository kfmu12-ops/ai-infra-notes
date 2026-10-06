# ai-infra-notes

Real-world benchmarks, bug fixes, and usage notes from running local AI tooling (ComfyUI, Ollama, local LLM agents) on AMD ROCm hardware. Each note is written up with specific measured numbers, not just anecdotes.

실제 AMD ROCm 환경에서 ComfyUI/Ollama/로컬 LLM 에이전트를 운용하며 얻은 실측 데이터·버그 수정·사용 노트 모음입니다. 각 글은 영어 원문 뒤에 한국어 번역을 함께 싣습니다.

## Notes

1. [ComfyUI + ROCm (Windows) fp8 crash fix via memmap](01_comfyui_rocm_windows_fp8_memmap_fix.md) — fixing `THPStorage_assertNotNull` when loading 35GB+ fp8 checkpoints on a 32GB-RAM machine
2. [Ollama's undocumented Vulkan backend on AMD](02_ollama_hidden_vulkan_backend_amd.md) — fixed RDNA4 hard-freezes and was ~2x faster than ROCm, measured
3. [`OLLAMA_GPU_OVERHEAD` doesn't actually reserve VRAM](03_ollama_gpu_overhead_not_respected.md) — confirmation + a working alternative (browser GPU-acceleration policies)
4. [20 failure patterns running a local LLM as a coding agent](04_local_llm_coding_agent_pitfalls.md) — a checklist from months of delegating real tasks to gpt-oss:20b
5. [A silent `<script>`-killing bug in Python triple-quoted HTML templates](05_python_triple_quote_js_escaping_silent_breakage.md) — breaks the frontend while every API test still passes
