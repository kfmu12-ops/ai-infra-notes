# ai-infra-notes

Real-world benchmarks, bug fixes, and usage notes from running local AI tooling (ComfyUI, Ollama, local LLM agents) on AMD ROCm hardware. Each note is written up with specific measured numbers, not just anecdotes.

실제 AMD ROCm 환경에서 ComfyUI/Ollama/로컬 LLM 에이전트를 운용하며 얻은 실측 데이터·버그 수정·사용 노트 모음입니다. 각 글은 영어 원문 뒤에 한국어 번역을 함께 싣습니다.

## Notes

1. [ComfyUI + ROCm (Windows) fp8 crash fix via memmap](01_comfyui_rocm_windows_fp8_memmap_fix.md) — fixing `THPStorage_assertNotNull` when loading 35GB+ fp8 checkpoints on a 32GB-RAM machine
2. [Ollama's undocumented Vulkan backend on AMD](02_ollama_hidden_vulkan_backend_amd.md) — fixed RDNA4 hard-freezes and was ~2x faster than ROCm, measured
3. [`OLLAMA_GPU_OVERHEAD` doesn't actually reserve VRAM](03_ollama_gpu_overhead_not_respected.md) — confirmation + a working alternative (browser GPU-acceleration policies)
4. [20 failure patterns running a local LLM as a coding agent](04_local_llm_coding_agent_pitfalls.md) — a checklist from months of delegating real tasks to gpt-oss:20b
5. [A silent `<script>`-killing bug in Python triple-quoted HTML templates](05_python_triple_quote_js_escaping_silent_breakage.md) — breaks the frontend while every API test still passes
6. [OpenUtau "noise" was the monitor's audio jack, not software](06_openutau_noise_was_hardware_not_software.md) — plus practical Korean vocal-synthesis tips (phonemizer, liaison rules, gender curve)
7. ["Caging" a coding agent: blacklist → whitelist shell sandboxing](07_sandboxing_a_coding_agent_blacklist_to_whitelist.md) — why keyword-blocking a shell hook doesn't hold against command substitution
8. [A RAG bug where reused `chunk_id`s silently corrupt vectors](08_rag_chunk_id_reuse_corrupts_vectors_silently.md) — and why content-hash keys fix it structurally
9. [A brand-new AMD RDNA4 card with no manual fan control in the driver](09_amd_rdna4_no_manual_fan_control.md) — Overdrive masking, a stale EFI boot entry, and a dead end
10. [Android `Dialog` + `WRAP_CONTENT` leaves dead whitespace below content](10_android_dialog_wrap_content_timing_bug.md) — a measure-timing bug, not a layout bug
11. [Fixing ACE-Step VRAM OOM on repaint — 4 wrong turns before the real fix](11_acestep_vram_oom_four_step_fix.md) — the fix was one env var
12. ["Hybrid models can't use prompt caching" was wrong](12_ollama_prompt_cache_hybrid_model_self_correction.md) — a two-session debugging arc on Ollama/llama.cpp prompt cache behavior
