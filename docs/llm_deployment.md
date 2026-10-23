# Language models on the lab hardware

Checked against the Ollama library on 2026-10-23. Model lineups change every
few months, so re-check before reinstalling.

The design rule in the README still applies: **facts come from the rule
engine, never from a language model.** A model's job here is to phrase
hints, explain output the read-only console (`rack/session.py`) already
fetched, and decide which allowed `show` command to run. It never states a
subnet, an IOS command's syntax, or a device's state from memory.

## Hardware

| Machine | GPU | Usable for inference? |
|---|---|---|
| Master PC: 12th-gen i3/i5, 16 GB RAM | RTX 3050 (8 GB, or 6 GB on some cards) | Yes, fully on GPU |
| 8 rack PCs | GeForce GT 710 (Kepler, 1–2 GB) | **No.** Ollama needs CUDA compute capability 5.0 or newer. The GT 710 is below that, so these PCs run on CPU only. |

## Recommended models

| Role | Machine | Model | Size | Why |
|---|---|---|---|---|
| Master agent | RTX 3050 8 GB | `qwen3.5:9b` (Q4_K_M) | ~6.6 GB | Strongest model that stays fully on an 8 GB card. Supports tools and thinking. |
| Master agent | RTX 3050 6 GB | `qwen3.5:4b` | ~3.4 GB | `9b` will spill to CPU on a 6 GB card. |
| Master (alternative) | RTX 3050 8 GB | `gemma4:12b` | ~7.7 GB | Too tight: almost no room left for context. Use only for one-off batch jobs. |
| Rack agent (default) | CPU | `qwen3.5:2b` | ~2.7 GB | Fast enough on CPU for short tutoring replies. Supports tools. |
| Rack agent (weak/old CPU, ≤8 GB RAM) | CPU | `qwen3.5:0.8b` | ~1.2 GB | Use as a router/classifier only. Hand anything substantive to the master. |
| Rack agent (≥16 GB RAM, recent CPU) | CPU | `qwen3.5:4b` | ~3.4 GB | Better tool calling, about half the speed of `2b`. |

Using one family (Qwen3.5) on both tiers means one prompt format and one
tool-call format to test.

## Topology

```
 rack PC 1..8  (CPU, qwen3.5:2b)            master PC (RTX 3050, qwen3.5:9b)
 ┌──────────────────────────────┐           ┌──────────────────────────────┐
 │ ccnabot: drills, rule engine │  LAN/HTTP │ Ollama :11434                │
 │ read-only console (show only)│ ────────► │  escalated questions          │
 │ small model: hints, routing  │           │  transcript summaries for     │
 └──────────────────────────────┘           │  the instructor report        │
                                            └──────────────────────────────┘
```

* The rack agent answers locally first, so a rack still works if the master
  or the network is down.
* It escalates to the master when the question needs reasoning across several
  devices' `show` output, or when a transcript needs summarising.
* With eight racks sharing one 3050, cap concurrency on the master:
  `OLLAMA_NUM_PARALLEL=4` and `num_ctx` 8192. Every parallel slot holds its
  own KV cache in VRAM.

## Settings

Master (`/etc/systemd/system/ollama.service.d/override.conf`):

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_NUM_PARALLEL=4"
Environment="OLLAMA_KEEP_ALIVE=-1"
```

Only let the lab subnet reach port 11434. Ollama has no authentication.

Rack PCs: set `"think": false` in requests (thinking is too slow on CPU),
and keep `num_ctx` at 4096 or less.

Check that nothing has spilled off the GPU: `ollama ps` should show
`100% GPU` on the master. If you see a CPU/GPU split, lower `num_ctx` or
`OLLAMA_NUM_PARALLEL`.

## Before choosing

Run `lscpu` and `free -g` on one rack PC. If it has an AVX2-capable CPU and
8 GB or more of RAM, use the default. If not, drop to `qwen3.5:0.8b`, or
skip local models and send every request to the master.
