---
id: RFC-0001
title: "Mac-native isolated runtime for agentic workloads"
status: Draft
author: Donald Gifford
created: 2026-05-20
---

<!-- markdownlint-disable-file MD025 MD041 -->

# RFC 0001: Mac-native isolated runtime for agentic workloads

<!--toc:start-->

- [RFC 0001: Mac-native isolated runtime for agentic workloads](#rfc-0001-mac-native-isolated-runtime-for-agentic-workloads)
  - [Summary](#summary)
  - [Problem Statement](#problem-statement)
  - [Proposed Solution](#proposed-solution)
  - [Design](#design)
    - [Workload definition](#workload-definition)
    - [Engine choice](#engine-choice)
    - [Implementation language](#implementation-language)
    - [Per-agent VM lifecycle](#per-agent-vm-lifecycle)
    - [Inspection surface](#inspection-surface)
    - [EKS deployment](#eks-deployment)
    - [Multi-arch image pipeline](#multi-arch-image-pipeline)
  - [Alternatives Considered](#alternatives-considered)
  - [Implementation Phases](#implementation-phases)
    - [Phase 1: MVP](#phase-1-mvp)
    - [Phase 2: Multi-agent and inspection](#phase-2-multi-agent-and-inspection)
    - [Phase 3: Agent profiles](#phase-3-agent-profiles)
    - [Phase 4: EKS deployment](#phase-4-eks-deployment)
    - [Phase 5: MCP endpoint](#phase-5-mcp-endpoint)
    - [Phase 6 (optional, ecosystem): Docker Engine API compatibility](#phase-6-optional-ecosystem-docker-engine-api-compatibility)
  - [Risks and Mitigations](#risks-and-mitigations)
  - [Success Criteria](#success-criteria)
  - [References](#references)
  <!--toc:end-->

**Status:** Draft **Author:** Donald Gifford **Date:** 2026-05-20

## Summary

A Mac-first developer tool, written in Go and built on libkrun, that runs AI
coding agents (Claude Code, Codex, Gemini CLI, Aider, etc.) inside lightweight
microVMs. The workload unit is an OCI image, which means the same agent
definition that runs locally on a developer's Mac or Linux machine also runs in
production on EKS with the gVisor runtime class. The product is independent of
the naos VMM project and ships on existing engines.

## Problem Statement

AI coding agents now routinely write files, run shell commands, install
dependencies, and make network calls on a developer's machine with the same
privileges as the developer themselves. The risk vector this opens is real and
concrete:

- In February 2026, the Cline VS Code extension (5M+ users) was compromised
  through a prompt-injection chain that exfiltrated npm release tokens and
  published an unauthorised package.
- Running Claude Code, Codex, Cursor's agent mode, or similar in "yolo" mode
  gives those tools full access to the host filesystem, SSH keys, AWS
  credentials, browser cookies (via filesystem access), and the network.
- The standard defence — review every action — destroys the productivity that
  made the agent worth using in the first place.

Existing approaches each have a meaningful flaw on Mac specifically:

| Option                                                 | Problem                                                                                                                                                                                                        |
| ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cloud sandboxes (E2B, Daytona, Modal, Codex cloud)     | Network round-trip kills interactive ergonomics; pricing scales with use; some workloads must stay local for IP or compliance reasons.                                                                         |
| Mac OS-level (Safehouse, sandvault, agent-sandbox.nix) | Built on `sandbox-exec`, which Apple has deprecated; not VM-grade isolation; will eventually break.                                                                                                            |
| Docker Sandboxes                                       | Commercial; requires Docker Desktop / Docker engine; ships microVM isolation but is a generic-container product, not agent-shaped.                                                                             |
| OrbStack + Incus, sandbox-claude setups                | Two-stage on Mac (host Linux VM + containers inside); high friction for fast iteration.                                                                                                                        |
| CodeRunner (InstaVM)                                   | Closest existing solution. Built on Apple's container runtime. MCP-tooling-shaped rather than workflow-shaped; weaker inspection surface; positioned as a backend for IDEs rather than a developer-facing CLI. |
| Apple Containerization framework + custom UX           | Mac-only, ties to Apple's roadmap (pre-1.0, Docker socket explicitly punted, plugin architecture undocumented), requires either subprocess overhead or a Swift daemon component.                               |

The gap is a Mac-first, VM-isolated, agent-workflow-shaped tool with first-class
network egress controls and a clean inspection surface, that also has a credible
production deployment story so the same workload definition follows the
developer from laptop to cluster.

## Proposed Solution

A Go CLI, distributed as a signed Mac binary (with Linux binary as a bonus),
that:

1. **Defines a workload as an OCI image + policy spec.** The OCI image is the
   agent's runtime environment (e.g. a Python 3.12 + Node 20 + git image). The
   policy spec describes the workspace mount, network egress allowlist,
   credentials forwarding, and resource limits.
2. **Runs the workload as a lightweight microVM via libkrun.** libkrun spins up
   a per-workload VM using Hypervisor.framework on Mac and KVM on Linux.
   Sub-second boot, hardware-enforced isolation, no Docker dependency, no daemon
   required for Phase 1.
3. **Treats the agent as a first-class concept.** Built-in profiles for Claude
   Code, OpenAI Codex, Gemini CLI, Aider, OpenHands, and similar.
   `agent run -- claude` launches Claude Code inside a sandbox that already
   knows how to forward the right credentials and mount the project workspace.
4. **Ships a parallel deployment target for EKS.** The same OCI image and policy
   spec translate to a Kubernetes Job manifest with `runtimeClassName: gvisor`.
   Developers iterate locally; long-horizon or background agents run in the
   cluster against the same image.
5. **Provides a clean inspection surface.** File diff, network log, and command
   log per agent session, viewable before any change is allowed to leave the
   sandbox.

## Design

### Workload definition

A single declarative format describes what runs and how it's allowed to behave:

```toml
# example.toml
[workload]
image = "ghcr.io/<org>/agent-runtime:python-3.12"
arch = ["arm64", "amd64"]

[workspace]
mount = "."
mode = "rw"

[network]
egress_allow = [
  "registry.npmjs.org",
  "pypi.org",
  "github.com",
  "api.anthropic.com",
]

[credentials]
ssh_agent = true
git_config = true
env_passthrough = ["ANTHROPIC_API_KEY"]

[resources]
vcpu = 2
memory_gb = 4

[agent]
profile = "claude-code"
```

### Engine choice

libkrun is the local engine across both Mac and Linux dev machines. EKS with the
gVisor runtime class is the cloud target. Both consume the same OCI image and
produce equivalent runtime behaviour for the supported workload class
(code-execution agents).

| Backend                       | Host                   | Used for                                         | Isolation model                          |
| ----------------------------- | ---------------------- | ------------------------------------------------ | ---------------------------------------- |
| libkrun (HVF backend)         | macOS, Apple Silicon   | Local Mac dev                                    | VM-per-workload via Hypervisor.framework |
| libkrun (KVM backend)         | Linux x86_64 / aarch64 | Local Linux dev                                  | VM-per-workload via KVM                  |
| EKS with gVisor runtime class | Linux nodes (any arch) | Cloud deployment, long-horizon background agents | Userspace-kernel intercept               |

The product links libkrun via CGo. The Go runtime owns the host-side process;
libkrun handles VM creation, vCPU execution, and virtio device emulation. No
separate daemon process is required for Phase 1 — each `agent run` invocation is
a single Go process that lives as long as the workload. A daemon mode for
persistent pools and shared state is deferred to a later phase.

### Implementation language

Go. Rationale:

- The container and Kubernetes ecosystem is Go-native: `client-go`, `cobra`,
  `oras-go`, `containers/image`, `go-containerregistry`, BuildKit, containerd's
  API definitions. Aligns directly with the EKS export side and image-handling
  needs.
- libkrun is a C library; CGo is a well-understood integration path.
- Single-binary distribution is simple; cross-compilation is straightforward;
  the toolchain is one users already have for adjacent tools.
- naos lives in Rust and remains independent; keeping the product in Go
  reinforces the separation between projects.

### Per-agent VM lifecycle

A `agent run -- <command>` invocation:

1. Resolves the agent profile to a workload definition.
2. Pulls (or reuses cached) OCI layers via `oras-go` or `containers/image`.
3. Materialises the rootfs into an ext4 image or directory.
4. Calls into libkrun to create a microVM: kernel (libkrunfw or user-supplied),
   rootfs, vCPU count, memory, vsock for control plane.
5. Mounts the workspace into the VM via virtio-fs.
6. Establishes the network egress filter (libkrun's networking + an in-product
   policy enforcement layer).
7. Forwards SSH-agent socket into the VM via vsock (keys remain on host).
8. Runs the agent process to completion or until the user terminates the
   session.
9. Captures the session's file diff, network log, and command log.
10. Tears down the VM. The session log is retained for inspection until
    `agent accept` or `agent reject`.

Cold start target: ≤2s end-to-end from `agent run` to the agent's first prompt.
Warm start with cached image: ≤500ms.

### Inspection surface

Before the user accepts changes from an agent session, the product surfaces
three artifacts:

- **File diff** — every file the agent created, modified, or deleted, against
  the workspace's git HEAD.
- **Network log** — every connection attempt, allowed or denied, with hostname,
  port, bytes transferred.
- **Command log** — every shell command the agent invoked, in order, with exit
  status.

Acceptance is explicit: `agent accept <session>` applies the diff;
`agent reject <session>` discards it. Rejected sessions leave no trace on the
host.

### EKS deployment

The CLI generates a Kubernetes manifest from the same workload definition:

```bash
agent export --target eks workload.toml > job.yaml
kubectl apply -f job.yaml
```

The generated Job specifies `runtimeClassName: gvisor`, a NetworkPolicy derived
from the egress allowlist, secret mounts derived from the credentials section,
and image references that resolve to the same multi-arch image used locally.
IRSA / pod identity replaces SSH-agent forwarding for cloud credentials.

### Multi-arch image pipeline

Mac is arm64; default EKS nodes are amd64 unless on Graviton. The product
assumes multi-arch images are non-negotiable. Built-in agent profiles ship as
multi-arch from day one via `buildx`. A `agent build` command wraps `buildx` for
users producing custom workload images.

## Alternatives Considered

**Apple Containerization framework as engine.** Rejected. Mac-only (would need a
second engine for Linux dev), pre-1.0 with frequent breaking changes, plugin
architecture undocumented, Docker socket explicitly punted by Apple
(apple/container#66). Would force the product to either subprocess the
`container` CLI (subprocess overhead, output-parsing fragility) or write a Swift
daemon (language split, FFI work). libkrun avoids all of this with a single C
library that works on both host OSes.

**smolvm or CodeRunner as engine.** Rejected. Either makes this product a UX
skin on a third-party project. Faster to MVP, but every UX decision becomes
"what does the underlying tool allow?" The product needs to own its abstractions
even if it doesn't own the VMM.

**naos-macos / naos-linux as engine.** Rejected for this product's timescale.
naos is a first-principles VMM learning project on a multi-year arc. Even when
complete, it would be one engine among several; using it here would couple the
product's ship date to naos finishing. The two projects share a developer but
should not share a timeline.

**Contributing to CodeRunner instead of building a new product.** Considered.
Faster to deliver value; trades off product ownership entirely. Worth keeping as
a fallback if user research suggests the differentiation isn't strong enough to
warrant a standalone product, but the workflow-shaped UX and inspection surface
are large enough surface areas to justify standalone.

**sandbox-exec-based isolation only.** Rejected. Apple has deprecated
`sandbox-exec`. Even before deprecation, process-level isolation is weaker than
VM-per-workload. The product would not survive the first credible escape
disclosure.

**Cloud-only.** Rejected. The product's reason for existing is local-first
ergonomics. Cloud-only is well-served by E2B, Daytona, Modal, Codex. The market
gap is local.

**Building a custom VMM for this product.** Rejected. 6-12 months of focused
work before anything useful ships. Wrong pacing. libkrun took years of community
effort; reimplementing it without a learning-project framing is bad use of time.
naos can fill this role eventually, but not on the product's timeline.

**Rust instead of Go.** Considered. Rust would give a tighter binary, slightly
nicer FFI to libkrun, and reuse of the user's naos toolchain. Rejected because
the container/k8s/OCI ecosystem the product depends on most heavily is
overwhelmingly Go: `client-go`, `oras-go`, `containers/image`,
`go-containerregistry`, BuildKit, containerd's API definitions. The dependency
surface fits Go better than Rust. Keeping the product in a different language
than naos also reinforces project separation.

## Implementation Phases

### Phase 1: MVP

- Go binary linking libkrun via CGo.
- `agent run -- <command>` runs a fixed default image in a libkrun microVM on
  Mac.
- Workspace mount via virtio-fs.
- Default egress allowlist (configurable via TOML).
- Session log captures stdout/stderr.
- Code-signed for Hypervisor.framework entitlement; distributed via brew
  formula.

Success: a single-developer demo where Claude Code completes a non-trivial task
inside the sandbox without host access, on a Mac.

### Phase 2: Multi-agent and inspection

- Parallel agents (each in its own libkrun VM).
- File diff, network log, command log surfaces.
- `agent accept` / `agent reject` workflow.
- Snapshot/restore where libkrun's backend supports it.
- Linux dev support (same Go binary, KVM backend).

### Phase 3: Agent profiles

- First-class profiles for Claude Code, Codex, Gemini CLI, Aider, OpenHands.
- Per-profile defaults: image, command, MCP wiring, credential forwarding.
- `agent profiles list` / `agent profiles inspect`.

### Phase 4: EKS deployment

- `agent export --target eks` produces a Job manifest with gVisor runtime class.
- NetworkPolicy generation from egress allowlist.
- IRSA documentation and example.
- Multi-arch build pipeline reference.

### Phase 5: MCP endpoint

- Local MCP server endpoint so external agents (IDEs, orchestrators) can
  delegate execution to the sandbox.
- Equivalent to CodeRunner's MCP integration but positioned as one of several
  integration surfaces, not the primary one.

### Phase 6 (optional, ecosystem): Docker Engine API compatibility

- A separate Docker Engine API socket layered on top of the libkrun engine.
- Unlocks `docker compose`, `kind`, `act`, dev containers, testcontainers
  against the product's runtime.
- Positioned as an ecosystem contribution, not a critical path for the product.

## Risks and Mitigations

| Risk                                                                               | Impact | Likelihood | Mitigation                                                                                                                                                                    |
| ---------------------------------------------------------------------------------- | ------ | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| libkrun project pace or direction changes                                          | Medium | Low        | libkrun is owned by the containers org / RedHat with multi-year track record; if needed, fork and vendor. Engine boundary is small.                                           |
| CGo overhead or build complexity hurts iteration speed                             | Medium | Medium     | Keep the CGo surface narrow; provide a Go-native interface layer above. Static linking on Linux; codesign + `.dylib` bundling on Mac.                                         |
| Hypervisor.framework entitlement / distribution friction                           | Medium | Medium     | Already a known pattern (UTM, OrbStack, smolvm, krunkit handle this); document the signing/entitlement story; ship a brew formula.                                            |
| gVisor syscall compatibility breaks specific agent workloads                       | Medium | Medium     | CI canary that runs the agent profile suite under gVisor; document known-incompatible patterns; users can target Firecracker or runc on their own clusters as escape hatches. |
| Multi-arch image complexity slows iteration                                        | Medium | High       | Adopt `buildx` in the build pipeline from day one; ship multi-arch profile images via GHCR; document `agent build` for user images.                                           |
| Network and credentials abstractions don't translate cleanly between local and EKS | High   | Medium     | Design the policy spec early with both targets in mind; the translation layer is a bounded design problem, not an open-ended one.                                             |
| Competing products (CodeRunner, Docker Sandboxes, smolvm) move faster              | Medium | High       | Differentiate explicitly on workflow UX (`agent run`, profiles), inspection surface (file diff, network log, command log), and dev/prod parity (EKS export).                  |
| Apple changes Hypervisor.framework or entitlement requirements                     | High   | Low        | libkrun's HVF backend is a thin wrapper; community would react quickly. Engine boundary lets us swap backends if needed.                                                      |
| User research shows CodeRunner is "good enough"                                    | High   | Unknown    | Run user interviews in Phase 1; pivot to upstream contribution if the differentiation hypothesis is wrong.                                                                    |

## Success Criteria

The product is successful if, twelve months after Phase 1 ship:

1. **Cold start performance** — `agent run -- claude` reaches the agent's first
   prompt in ≤2 seconds on a baseline M-series Mac.
2. **Parallel capacity** — at least 8 concurrent agent VMs run reliably on a
   36GB M-series Mac.
3. **Workload parity** — at least one published agent profile runs unmodified on
   Mac (libkrun + HVF), Linux (libkrun + KVM), and EKS (gVisor) with identical
   observable behaviour.
4. **Adoption signal** — at least one well-known agent project (Claude Code,
   Aider, OpenHands, similar) recommends the tool in its documentation as a
   sandboxing option.
5. **No credible isolation incident** — no reported case of an agent reaching
   host filesystem, host network, or host credentials from within a sandbox.

## References

- libkrun: <https://github.com/containers/libkrun> (Apache-2.0, RedHat /
  containers org)
- libkrunfw: <https://github.com/containers/libkrunfw> (bundled Linux kernel)
- smolvm: <https://github.com/smol-machines/smolvm> (libkrun-based comparator)
- Apple Containerization framework: <https://github.com/apple/containerization>
  (considered, rejected)
- Apple `container` CLI: <https://github.com/apple/container>
- apple/container#66 — Docker Engine API request, closed not planned
- gVisor: <https://gvisor.dev>
- CodeRunner (InstaVM): comparable VM-isolated agent sandbox built on Apple's
  container runtime
- Docker Sandboxes: <https://www.docker.com/products/docker-sandboxes/>
- Cline incident, February 2026 — prompt-injection chain exfiltrating npm
  release tokens
- "What's the best code execution sandbox for AI agents in 2026?" — Northflank
