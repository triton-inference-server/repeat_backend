<!--
# Copyright 2026, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#  * Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#  * Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#  * Neither the name of NVIDIA CORPORATION nor the names of its
#    contributors may be used to endorse or promote products derived
#    from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ``AS IS'' AND ANY
# EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
# PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
# PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
# OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
-->

# Security Policy

## Reporting a Vulnerability

NVIDIA takes the security of its software seriously. To report a potential
security vulnerability in this project or any NVIDIA product, use one of the
following channels. **Do not open a public GitHub issue for a security
vulnerability.**

1. **NVIDIA Vulnerability Disclosure Program (preferred):**
   <https://www.nvidia.com/en-us/security/>
2. **Email:** [psirt@nvidia.com](mailto:psirt@nvidia.com). Please encrypt
   sensitive reports with the
   [NVIDIA PGP key](https://www.nvidia.com/en-us/security/pgp-key).
3. **GitHub Private Vulnerability Reporting:** use the "Report a
   vulnerability" button on the Security tab of this repository.

**OEM partners should contact their NVIDIA Customer Program Manager.**

Please include:

1. Product name and version or branch that contains the vulnerability
2. Type of vulnerability (for example denial of service or memory safety)
3. Instructions to reproduce the vulnerability
4. Proof-of-concept or exploit code, if available
5. Potential impact, including how an attacker could exploit it

NVIDIA PSIRT acknowledges reports, assesses severity, coordinates a fix and
disclosure timeline with the reporter, and publishes advisories. See
<https://www.nvidia.com/en-us/security/> for past security bulletins and
notices.

## Security Architecture and Context

**Project:** Triton Inference Server Repeat Backend (`repeat_backend`).

**Software classification:** Library. It is a shared object
(`libtriton_repeat.so`) loaded in-process by Triton Inference Server through
the TRITONBACKEND C API.

**Purpose:** An example backend that demonstrates decoupled model execution,
where each request produces zero or more responses. For each element of input
`IN` it sends one response containing `OUT` and `IDX` after a per-element
`DELAY` (milliseconds). Input `WAIT` controls how long the backend holds the
request before releasing it. It is intended for testing and demonstration, not
for production workloads.

**Primary security responsibility:** Validate the model configuration and
per-request tensor shapes, and manage the lifetime of per-request buffers,
response factories and response threads, so that requests cannot corrupt
memory or exhaust the host process.

**Key security boundaries and interfaces:**

- **TRITONBACKEND API boundary.** The backend has no network listener, file
  input, authentication or persistent storage. All data arrives through
  Triton as tensors (`IN`, `DELAY`, `WAIT`) and the model configuration
  (`src/repeat.cc`). Authentication, authorization, TLS and request
  size limits are the responsibility of the hosting Triton server.
- **In-process trust.** The backend runs in the Triton server process with
  the server's privileges. A defect in the backend affects the whole server.

**Repository Exposure Classification:** Public. Basis: the GitHub repository
visibility is public.

**Service Exposure Classification:** Internal-Isolated (medium confidence).
Basis: example and test backend with no network exposure of its own, handling
no sensitive data. If it is deployed inside a service exposed to untrusted
clients, the exposure of that service governs.

## Threat Model

Ordered by assessed severity and likelihood.

1. **Resource exhaustion through unbounded delays and per-request threads:**
   `ModelInstanceState::ProcessRequest` starts one detached thread per request
   and `ResponseThread` sleeps for the client-supplied `DELAY` values. The
   `WAIT` value likewise blocks `TRITONBACKEND_ModelInstanceExecute`. No upper
   bound is enforced, so a client able to send inference requests can tie up
   threads and instance execution slots, degrading availability of the server.
2. **Memory-management defects in request buffers:** `IN` and `DELAY` copies
   are allocated as arrays but held in `std::unique_ptr<int32_t>`, which
   releases them with a non-array deleter. This is undefined behavior
   (CWE-762) on every request. The element count is derived from the `IN`
   byte size and is also used for `DELAY`, so correctness depends on the
   shape check performed per request in `ProcessRequest`.
3. **Unload hang from an unbalanced in-flight counter:** `ResponseThread`
   returns early on a response-creation error without decrementing
   `inflight_thread_count_`. `~ModelInstanceState` waits for that counter to
   reach zero, so model unload or server shutdown can block indefinitely
   after such an error.
4. **Negative or oversized delay values:** `DELAY` is read as a signed 32-bit
   value and converted to a duration. Negative values are not rejected, and
   very large values keep a thread alive for a long time.
5. **Information exposure through logs:** `ValidateModelConfig` writes the
   full model configuration to the server log at INFO level, and per-response
   log lines are emitted for every request. Operators who place sensitive
   values in model configuration parameters should be aware they appear in
   logs.
6. **Supply chain:** The build fetches the `backend`, `core` and `common`
   repositories by branch or tag at configure time (`CMakeLists.txt`). A
   mutable branch reference means the resulting binary depends on upstream
   state at build time.

## Critical Security Assumptions

- **The hosting Triton server authenticates and authorizes clients.** This
  backend implements no access control of its own.
- **Request sizes and rates are limited upstream.** The backend does not cap
  the number of elements in `IN`, the values in `DELAY` or `WAIT`, or the
  number of concurrent requests.
- **Clients are trusted not to supply hostile timing values.** Delay values
  are used as given.
- **Model configuration is trusted.** Anyone who can load a model into the
  model repository can load code into the Triton process, and the backend
  only checks that the configuration matches the expected tensor names,
  shapes and datatypes.
- **The TRITONBACKEND API behaves as documented.** Buffers returned by Triton
  are assumed valid for the sizes requested, and the backend only handles
  output buffers in CPU memory.
- **This backend is not a production component.** It is provided as a
  reference and test fixture and is not hardened for untrusted workloads.

## Supported Versions

Security fixes are applied to the default branch and to the current Triton
release branch. Use the release of this backend that matches your Triton
Inference Server release.
