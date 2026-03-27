---
title: CVE-2025-14174 The ANGLE Graphics Bug That Let DarkSword Escape Safari's Sandbox
date: 2026-03-24 00:00:00 +/-TTTT
categories: [security]
tags: [darksword, xpc, mediaplaybackd, ios ipc, privilege escalation, CVE-2025-43510, applem2scalercscdriver, sandbox escape, kernel, process injection]
---


## Executive Summary

Stage 4 of DarkSword is the most architecturally clever piece of the entire chain — and the least discussed. While earlier stages (CVE-2025-31277/43529 for RCE, CVE-2026-20700 for PAC bypass, CVE-2025-14174 for GPU sandbox escape) get most of the attention, **Stage 4 is the bridge that connects the GPU process to the iOS kernel**.

After Stage 3 escapes Safari's WebContent sandbox into the GPU process, DarkSword still needs to reach the kernel. But the GPU process cannot directly access the kernel driver interfaces needed for Stage 5's kernel memory corruption (CVE-2025-43510). The solution: **XPC injection into `mediaplaybackd`** — a higher-privilege system daemon that *can* reach those interfaces.

Stage 4 involves:
1. From the compromised GPU process, crafting **malicious XPC messages** targeting `mediaplaybackd`'s exposed inter-process communication interfaces
2. Using those messages to **inject code** — including loading a full JavaScriptCore runtime — into the `mediaplaybackd` daemon process
3. Establishing **arbitrary memory read/write and function call primitives** within `mediaplaybackd`
4. Using those primitives to then execute CVE-2025-43510 against the `AppleM2ScalerCSCDriver` kernel driver (Stage 5)

This stage has no dedicated CVE — it exploits the **architectural trust model of XPC itself**, combined with the elevated context gained from prior stages. It is a testament to how carefully DarkSword's developers mapped the iOS privilege hierarchy to find the optimal pivot path.

---

## Understanding XPC: iOS's Inter-Process Communication Backbone

### What XPC Is

**XPC (XNU inter-Process Communication)** is Apple's primary mechanism for communication between processes on iOS and macOS. It is built on top of **Mach ports** — the low-level kernel primitive for IPC on Apple platforms — but provides a higher-level, type-safe, asynchronous API.

Every significant system service on iOS communicates via XPC:

```
launchd (PID 1)
    │
    ├── Manages lifecycle of all XPC services
    ├── Launches services on-demand
    └── Restarts crashed services automatically

Example XPC service ecosystem:
    ├── mediaplaybackd  ← media playback
    ├── nsurlsessiond   ← network
    ├── imagent         ← iMessage
    ├── SpringBoard     ← UI/home screen
    ├── locationd       ← GPS
    └── trustd          ← code signing
```

### The XPC Security Model

XPC's security model relies on several layers:

**1. Client Validation**
When a client connects to an XPC service, the service's delegate can (and should) validate the caller:
```objc
- (BOOL)listener:(NSXPCListener *)listener 
    shouldAcceptNewConnection:(NSXPCConnection *)newConnection {
    
    // Validate client's code signature
    SecCodeRef clientCode = NULL;
    SecCodeCreateWithXPCMessage(newConnection, &clientCode);
    
    // Check that client has required entitlement
    SecRequirementRef requirement = NULL;
    SecRequirementCreateWithString(
        CFSTR("entitlement[\"com.apple.media.playback.client\"] = TRUE"),
        kSecCSDefaultFlags, &requirement);
    
    OSStatus result = SecCodeCheckValidity(clientCode, 
        kSecCSDefaultFlags, requirement);
    
    return (result == noErr);  // Only accept if entitlement matches
}
```

**2. Sandbox Entitlements**
XPC services run in their own sandbox. A service only has access to what its sandbox profile and entitlements explicitly permit.

**3. Mach Port Naming**
XPC connections use named Mach ports. Only processes that know the port name (or are given it by launchd) can connect.

### The Trust Assumption XPC Makes

The critical security assumption: **if a client process connects to an XPC service, the XPC framework assumes that process has appropriate authorization** (entitlements, code signature, sandbox profile).

The weakness: **if an attacker compromises a process that legitimately has access to an XPC service, they inherit that process's trust level**. The GPU process on iOS has XPC connections to multiple system daemons — including `mediaplaybackd` — because it needs to coordinate GPU resource management during media playback.

---

## `mediaplaybackd`: The Target Daemon

### What mediaplaybackd Does

`mediaplaybackd` is an iOS system daemon (`/usr/libexec/mediaplaybackd`) responsible for:
- Coordinating hardware-accelerated media playback (video decoding, audio routing)
- Managing AVFoundation pipeline on behalf of apps
- Interfacing with GPU and media hardware subsystems
- Scheduling media processing across multiple processes

Its key characteristic for DarkSword's purposes: **it runs with significantly higher system privileges than the GPU process**, and it has XPC interfaces that expose kernel driver functionality — specifically the `AppleM2ScalerCSCDriver` used in Stage 5.

### mediaplaybackd's Privilege Level

```
iOS Process Privilege Hierarchy (simplified):

Root / kernel
    │
    ├── System Daemons (launchd, configd, etc.)  ← High privilege
    │       └── mediaplaybackd  ← STAGE 4 TARGET
    │
    ├── GPU Process  ← STAGE 3 (where we are after Stage 3)
    │
    ├── WKWebContent Process  ← STAGE 1/2 (where we started)
    │
    └── App processes  ← Lowest user privilege

DarkSword's traversal:
  WKWebContent → GPU → mediaplaybackd → kernel
```

### Why mediaplaybackd Specifically?

DarkSword's developers chose `mediaplaybackd` for a precise technical reason: it exposes XPC interfaces that interact with `AppleM2ScalerCSCDriver` — the kernel driver containing CVE-2025-43510.

```
GPU process
    │
    │ Has XPC connection to mediaplaybackd
    │ (legitimate: GPU needs to coordinate with media subsystem)
    ▼
mediaplaybackd
    │
    │ Has privileged XPC interface to kernel drivers
    │ including AppleM2ScalerCSCDriver
    ▼
AppleM2ScalerCSCDriver (kernel)
    │
    │ CVE-2025-43510: COW race condition
    ▼
Arbitrary kernel memory R/W
```

The GPU process cannot directly reach `AppleM2ScalerCSCDriver`. But `mediaplaybackd` can. And the GPU process has a legitimate XPC pathway to `mediaplaybackd`. Stage 4 exploits this chain.

---

## Stage 4 Technical Mechanics

### Step 1: From GPU Process to mediaplaybackd XPC Connection

After Stage 3 provides code execution in the GPU process, DarkSword uses that execution context to:

1. **Enumerate existing XPC connections** in the GPU process — the GPU process already has live XPC connections to system daemons (established by legitimate iOS media operations)

2. **Locate the `mediaplaybackd` XPC connection** among the existing connections using memory reading (available from Stage 2's arbitrary R/W)

3. **Craft malicious XPC messages** targeting `mediaplaybackd`'s exposed service interfaces

```javascript
// CONCEPTUAL — DarkSword's sbx1_main.js approach (illustrative)
// This runs in the GPU process after Stage 3

// Step 1: Find existing Mach port to mediaplaybackd
// GPU process has legitimate IPC connections to media subsystem
let mediaPlaybackPort = findXPCConnection("com.apple.mediaplayback.xpc");

// Step 2: Craft message that triggers CVE-2025-43510 indirectly
// via mediaplaybackd's exposed XPC interface to AppleM2ScalerCSCDriver
let maliciousMessage = craftXPCMessage({
    selector: 1,  // AppleM2ScalerCSCDriver selector
    // Payload designed to trigger COW race condition
    // that provides kernel memory R/W
    payload: buildCOWRacePayload()
});

// Step 3: Send crafted XPC message
sendXPCMessage(mediaPlaybackPort, maliciousMessage);
```

### Step 2: XPC Message Injection and Code Loading

DarkSword goes further than just sending XPC messages — it **injects a full JavaScriptCore runtime into `mediaplaybackd`**. This is a sophisticated technique:

```
Why load JavaScriptCore into mediaplaybackd?
─────────────────────────────────────────────
DarkSword is entirely JavaScript-based.
Its Stage 5 exploit (pe_main.js) is JavaScript code.
It needs a JavaScript runtime to execute Stage 5.

Solution:
1. Load JavaScriptCore.framework into mediaplaybackd
   (using XPC to trigger dlopen/NSBundle loading)
2. Execute the JavaScript Stage 5 payload within that runtime
3. Stage 5 (pe_main.js) runs in mediaplaybackd's context
   with mediaplaybackd's privileges and access
```

This means even the kernel exploitation happens through JavaScript — an extraordinary achievement that keeps the entire DarkSword chain pure JS from beginning to end.

```javascript
// CONCEPTUAL — JavaScript executed within mediaplaybackd context
// This is pe_main.js (Stage 5) running inside mediaplaybackd

// mediaplaybackd has access to AppleM2ScalerCSCDriver via XPC
// Use this to call CVE-2025-43510's vulnerable selector
let driver = openAppleM2ScalerCSCDriver();

// Trigger CVE-2025-43510: COW race condition
// Get kernel arbitrary R/W from mediaplaybackd context
let kernelRW = triggerCOWRace(driver);

// Now proceed to Stage 6: kernel privilege escalation
// With kernel R/W from mediaplaybackd's elevated context
```

### Step 3: Establishing Memory Primitives in mediaplaybackd

Within `mediaplaybackd`, DarkSword establishes the same exploit primitives used in earlier stages (addrof/fakeobj → arbitrary R/W), but this time with `mediaplaybackd`'s broader system access:

```
mediaplaybackd's memory access capabilities:
─────────────────────────────────────────────
✓ Read/write to its own process memory
✓ IPC channels to kernel drivers
✓ Access to hardware-accelerated media subsystem
✓ Interaction with AppleM2ScalerCSCDriver (kernel)
✓ Higher sandbox entitlements than GPU process
✓ Access to system memory-mapped regions

→ These capabilities make CVE-2025-43510 accessible
→ The COW bug can be triggered from this context
```

### Step 4: The XPC-to-Kernel Bridge

The architectural beauty of Stage 4 is how it uses XPC as a privilege elevator:

```
Privilege Escalation via XPC:
──────────────────────────────

GPU Process (lower privilege)
    │
    │ XPC message (legitimate channel)
    │ but with malicious payload
    ▼
mediaplaybackd (higher privilege)
    │
    │ Processes malicious message
    │ Executes injected JavaScript (JSCore loaded)
    │ pe_main.js runs here
    ▼
AppleM2ScalerCSCDriver (kernel driver)
    │
    │ CVE-2025-43510: COW race condition
    │ Triggered by mediaplaybackd's privileged access
    ▼
Kernel arbitrary memory R/W
    │
    ▼
CVE-2025-43520: Kernel privilege escalation → ROOT
```

---

## The iOS Process Architecture DarkSword Mapped

Stage 4 reveals the depth of DarkSword's architectural analysis. The developers mapped the complete iOS privilege hierarchy and identified the optimal "hop" path from the GPU process to the kernel:

```
iOS Privilege Hierarchy (DarkSword's path):
────────────────────────────────────────────

KERNEL
  ↑ CVE-2025-43520 (Stage 6)
  
AppleM2ScalerCSCDriver
  ↑ CVE-2025-43510 (Stage 5)
  
mediaplaybackd daemon
  ↑ XPC Injection (Stage 4) ← no CVE — exploits trust model
  
GPU Process
  ↑ CVE-2025-14174 + CVE-2026-20700 (Stage 3)
  
WebContent Process
  ↑ CVE-2025-31277/43529 (Stage 1)

Internet (attacker's website)
```

Each "hop" requires different exploitation techniques:
- Stages 1-3: Memory corruption, type confusion, OOB writes
- Stage 4: **Architectural exploitation** — using XPC's trust model as a privilege elevator
- Stages 5-6: Memory corruption at kernel level

Stage 4 is the only stage that requires **no memory corruption at all**. It exploits the architecture, not a specific bug.

---

## Why This Approach Is Architecturally Significant

### The "No CVE" Stage

Stage 4 is noteworthy because it has no assigned CVE. It doesn't exploit a specific bug — it exploits **how iOS processes are connected by design**:

1. GPU process legitimately communicates with `mediaplaybackd` (by design)
2. `mediaplaybackd` legitimately communicates with kernel drivers (by design)
3. XPC validates connections based on process identity (by design)
4. A compromised GPU process inherits its real identity's trust (architectural property)

This is not a bug — it is **privilege escalation through trusted channels**. The "fix" is not a CVE patch; it is architectural hardening of XPC validation to detect compromise indicators.

### Loading JavaScriptCore Into mediaplaybackd

The decision to load a full JSC runtime into `mediaplaybackd` is an extraordinary engineering choice. It means:

- No stage-specific shellcode is needed for Stage 5
- The entire Stage 5 kernel exploit is written in JavaScript
- DarkSword maintains its pure-JavaScript architecture throughout
- The same JavaScript execution infrastructure works across all stages

This makes DarkSword more maintainable, portable, and resistant to signature-based detection — shellcode triggers many AV/EDR signatures, but loading a legitimate Apple framework (JavaScriptCore) into a media daemon is harder to flag.

---

## Detection

### Behavioral Indicators

Stage 4 is the hardest stage to detect because it uses legitimate IPC channels:

**Process-Level:**
```
High-priority detection:
- mediaplaybackd spawning unexpected child processes
- Unusual dynamic library loads into mediaplaybackd:
  → JavaScriptCore.framework loaded by mediaplaybackd ← ANOMALOUS
  → MediaPlayer.framework loaded outside normal media context
- mediaplaybackd accessing kernel interfaces outside normal media operations
- Unusual memory mapping in mediaplaybackd's address space
```

**XPC/IPC Level:**
```
- Unusually large or complex XPC messages from GPU process to mediaplaybackd
- XPC messages to mediaplaybackd with non-standard message types
- GPU process accessing mediaplaybackd interfaces during non-media activity
  (e.g., during Safari browsing with no active media playback)
```

**Temporal Pattern:**
```
Stage 4 occurs very rapidly (< 1 second) after Stage 3.
Look for:
- Rapid sequential anomalies across GPU process → mediaplaybackd → kernel
- Burst of XPC activity from Safari GPU process during page load (no video playing)
- JavaScriptCore.framework loaded into multiple processes simultaneously
```

**iVerify Detection:**
- iVerify's kernel behavioral monitoring can detect post-Stage-4 anomalies
- The moment CVE-2025-43510 fires from Stage 5 creates detectable kernel artifacts
- Abnormal kernel memory access patterns from mediaplaybackd context

### MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Privilege Escalation | Process Injection via IPC | T1055 |
| Privilege Escalation | Exploitation for Privilege Escalation | T1068 |
| Defense Evasion | Process Injection | T1055 |
| Defense Evasion | Masquerading: Match Legitimate Name | T1036 |
| Lateral Movement | Exploitation of Remote Services (IPC) | T1210 |

---

## The Full DarkSword Architecture with Stage 4 in Context

```
┌─────────────────────────────────────────────────────────────────┐
│                  DARKSWORD COMPLETE CHAIN                       │
│                                                                 │
│  [Attacker's watering hole website]                             │
│          │ Hidden iframe loads DarkSword JS                     │
│          ▼                                                      │
│  Stage 1a/b: CVE-2025-31277 / CVE-2025-43529                    │
│    JavaScriptCore JIT/GC → Arbitrary R/W in WebContent          │
│          │                                                      │
│  Stage 2: CVE-2026-20700                                        │
│    dyld PAC/TPRO bypass → Arbitrary code exec in WebContent     │
│          │                                                      │
│  Stage 3: CVE-2025-14174                                        │
│    ANGLE OOB write → GPU process sandbox escape                 │
│          │                                                      │
│  ★ Stage 4: XPC Injection (no CVE)  ★                           │
│    GPU process → XPC → mediaplaybackd injection                 │
│    Load JavaScriptCore into mediaplaybackd                      │
│    pe_main.js executes in mediaplaybackd context                │
│          │                                                      │
│  Stage 5: CVE-2025-43510                                        │
│    AppleM2ScalerCSCDriver COW race → kernel R/W                 │
│    (triggered from mediaplaybackd's privileged context)         │
│          │                                                      │
│  Stage 6: CVE-2025-43520                                        │
│    Kernel memory corruption → full ROOT privileges              │
│          │                                                      │
│  [GHOSTBLADE / GHOSTKNIFE / GHOSTSABER deployed]                │
│  [Data exfiltration in seconds → self-deletion]                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Mitigation

Stage 4 has no dedicated patch because it exploits the architecture, not a specific vulnerability. However:

**Indirect mitigation:**
- Patching **Stage 1 CVEs** (CVE-2025-31277/43529) prevents the chain from ever starting
- Patching **Stage 3 CVE** (CVE-2025-14174) stops the chain before Stage 4
- **Lockdown Mode** blocks Stage 1 entirely → Stage 4 is never reached

**Apple's architectural response:**
Apple has been hardening XPC validation across iOS releases — adding more granular process identity checks that would make Stage 4's GPU-to-mediaplaybackd pivot harder to accomplish even with code execution in the GPU process.

**Full DarkSword protection:**
```
Update to iOS 18.7.6 or iOS 26.3.1
→ Patches all 6 CVEs in the chain
→ Stage 4's XPC pivot becomes irrelevant (chain broken at Stage 1)
```

---

## Conclusion

DarkSword Stage 4's XPC injection into `mediaplaybackd` is perhaps the most elegant and architecturally sophisticated piece of the entire chain. It requires no CVE — no memory corruption, no type confusion, no buffer overflow. It simply maps the iOS privilege hierarchy, identifies that `mediaplaybackd` is a reachable trust boundary with kernel driver access, and uses the GPU process's legitimate XPC channel to traverse that boundary.

The decision to load JavaScriptCore into `mediaplaybackd` and execute Stage 5 as pure JavaScript within that context reveals a developer philosophy of maintaining consistency and minimizing detection surface. No shellcode, no foreign binaries — just Apple's own frameworks being used in unexpected ways.

This stage teaches an important lesson about the limits of process isolation: **if an attacker compromises a process that legitimately communicates with a privileged service, they inherit that process's trust relationships**. Sandboxes protect against untrusted processes. They provide weaker guarantees when a trusted (but compromised) process is the attacker.

---

## References

- [Google Threat Intelligence — DarkSword iOS Exploit Chain](https://cloud.google.com/blog/topics/threat-intelligence/darksword-ios-exploit-chain)
- [iVerify — DarkSword Explained](https://iverify.io/blog/darksword-ios-exploit-kit-explained)
- [Apple Developer — XPC Services](https://developer.apple.com/documentation/xpc)
- [Apple Developer — Creating XPC Services](https://developer.apple.com/documentation/xpc/creating-xpc-services)
- [MITRE ATT&CK — XPC Services Abuse (T1559.003)](https://attack.mitre.org/techniques/T1559/003/)
- [NVD — CVE-2025-43510](https://nvd.nist.gov/vuln/detail/CVE-2025-43510)
- [NSHipster — Inter-Process Communication on iOS/macOS](https://nshipster.com/inter-process-communication/)
- [The Register — DarkSword Exploit Kit](https://www.theregister.com/2026/03/18/darksword_exploit_kit_steals_iphone/)
- [Security Week — DarkSword Coverage](https://www.securityweek.com/darksword-ios-exploit-kit-used-by-state-sponsored-hackers-spyware-vendors/)
- [Lookout — DarkSword Threat Intelligence](https://www.lookout.com/threat-intelligence/article/darksword)

---

*This post is intended for security researchers, iOS security engineers, and mobile threat analysts. All technical details are based on publicly disclosed research.*
