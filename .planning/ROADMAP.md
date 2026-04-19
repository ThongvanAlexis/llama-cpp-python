# Roadmap: Windows CUDA Wheels CI for llama-cpp-python (fork)

## Milestones

- ✅ **v1.0 Windows CUDA Wheels CI** — Phases 1–4 (shipped 2026-04-19) — see [`milestones/v1.0-ROADMAP.md`](milestones/v1.0-ROADMAP.md) and [`MILESTONES.md`](MILESTONES.md)
- 📋 **v2 (candidates)** — matrix expansion, automation, provenance, gh-pages PEP 503 index — not yet scheduled; see [`PROJECT.md` v2 Candidates section](PROJECT.md)

## Phases

<details>
<summary>✅ v1.0 Windows CUDA Wheels CI (Phases 1–4) — SHIPPED 2026-04-19</summary>

- [x] **Phase 1: Scaffold & Toolchain Pinning** (3/3 plans) — completed 2026-04-16. Windows-only workflow file with MSVC auto-select from CUDA compat matrix and CUDA toolkit unified behind mamba. 16 requirements (WF-01..05, TC-01..10, DOC-04).
- [x] **Phase 2: Build & Cache** (2/2 plans) — completed 2026-04-17. scikit-build-core produces correctly-tagged `cp311-cp311-win_amd64.whl` under 400 MB with `+cu126.ll<sha12>` version embed; sccache + mamba split caches save-on-failure. 13 requirements (BLD-01..13).
- [x] **Phase 3: Smoke Test (Publish Gate)** (1/1 plan) — completed 2026-04-18. Fresh-runner DLL-load import gate (simplified from inference, user-approved), nvcuda.dll stub via pefile, cuobjdump arch verification, `GGML_NATIVE=OFF` portability. 8 requirements (ST-01..08; ST-01 + ST-06 superseded by DLL-load gate).
- [x] **Phase 4: Publish & Consumer UX** (2/2 plans) — completed 2026-04-19. Publish job on `ubuntu-latest` with `needs: [build, smoke-test]` gate, `softprops/action-gh-release@v2` byte-identical upload, forensics body with 12-char llama.cpp SHA, fork-focused README. 5 active requirements (PUB-01, PUB-02, DOC-01, DOC-02, DOC-05); 9 deferred to v2 (PUB-03..10, DOC-03).

Full details: [`milestones/v1.0-ROADMAP.md`](milestones/v1.0-ROADMAP.md).

</details>

### 📋 v2 (Planned — not yet scheduled)

No phases scheduled. Candidate scope lives in [`PROJECT.md` v2 Candidates section](PROJECT.md) (matrix expansion MX-*, automation AUT-*, provenance QA-*, gh-pages PUB-03..10).

Start with `/gsd:new-milestone` when ready.

## Progress

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Scaffold & Toolchain Pinning | v1.0 | 3/3 | Complete | 2026-04-16 |
| 2. Build & Cache | v1.0 | 2/2 | Complete | 2026-04-17 |
| 3. Smoke Test (Publish Gate) | v1.0 | 1/1 | Complete | 2026-04-18 |
| 4. Publish & Consumer UX | v1.0 | 2/2 | Complete | 2026-04-19 |

---
*Roadmap created: 2026-04-15 — v1.0 shipped 2026-04-19.*
