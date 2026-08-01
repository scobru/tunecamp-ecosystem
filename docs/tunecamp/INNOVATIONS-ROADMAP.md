# TuneCamp Innovations & R&D Roadmap

This document outlines the current technical innovations operating within the TuneCamp ecosystem and details proposed R&D features designed to advance zero-server, privacy-first, and air-gapped audio distribution.

---

## 1. Existing Innovations in TuneCamp

TuneCamp already implements several cutting-edge architectural patterns across identity, P2P network distribution, and web audio performance:

### 1.1 FID Protocol (Fediverse-ID) & Zero-Knowledge Passports (`@scobru/fid`)
- **Zero-Knowledge Identity:** Rather than relying on central user databases or domain-locked passkeys (which fork identity per host), FID derives a deterministic `secp256k1` keypair directly from `alias:passphrase` using ZEN SEA (`scobru/zen`).
- **Cross-Instance Passport:** Users authenticate on independent TuneCamp instances (`/api/auth/zen/*`) via cryptographic challenge-response without transmitting raw secrets.

### 1.2 Isolated P2P Companion & Real-Time Audio Engine (Sidecamp & Graphofone)
- **Architectural Isolation:** Clean separation between the core self-hosted streaming server (TuneCamp node) and peer-to-peer acquisition (Soulseek, WebTorrent).
- **Graphofone DSP Threading:** Web Audio real-time DSP graph execution is completely decoupled from disk and network I/O to guarantee zero latency glitches during live performance setups.

### 1.3 Decentralized P2P Profile & Release Aggregation
- **Zero-Server Directory Indexing:** `tunecamp-website` uses decentralized ZEN P2P relays (`wss://delay.scobrudot.dev/zen`) to discover, aggregate, and play federated artist releases and passports dynamically without a central backend indexer.

### 1.4 Dynamic Lab App Micro-Frontend Ecosystem
- **Sandbox Integration:** DB-driven `lab_apps` system allowing complex WebGL/WebAudio applications (`Audiofabric` 3D visualizer, `4-Track Recorder` multitrack overdubbing) to run isolated in iFrames while consuming instance Subsonic/audio streams.

### 1.5 Built-in MCP Server with Zero-Trust FID Auth
- **AI Agent Native:** TuneCamp core integrates a Model Context Protocol (MCP) server authenticated via zero-trust FID headers (`Authorization: FID <zen_pub_key>`).

---

## 2. Proposed Innovations (R&D Roadmap)

Below are two upcoming innovation proposals for zero-server live audio drops and air-gapped optical file transfers.

### 2.1 Protocollo P2P per Drop Audio Effimeri (Zero-Server Live QR Drop)

#### Concept & Workflow
Instead of displaying a QR code that redirects audience smartphones to centralized cloud platforms (Spotify, SoundCloud, or external web servers), the QR code embeds a compressed **WebRTC signaling payload** and decentralized network identity.

1. **Live Performance Scenario:** During a gig or live set, a dynamic or static QR code is projected onto a venue screen or poster.
2. **Instant P2P Handshake:** Scanning the QR code does not open an external cloud website; instead, the browser decodes the signaling payload and opens a direct WebRTC DataChannel / AudioStream to the artist's local daemon (running on a laptop or Raspberry Pi).
3. **Zero-Server Audio Streaming:** The audio track or stem is served directly from the artist's local disk to audience devices in real-time.

#### Tech Stack
- **Client (Mobile / Browser):** TypeScript (WebRTC peer connection management, audio buffer sync, PWA player).
- **Daemon (Artist Host):** Lightweight Rust backend (high-concurrency zero-copy audio chunk streaming, memory-safe data channel management).

#### Impact
Completely decouples audio distribution from third-party servers, CDNs, or cellular internet requirements at live venues.

---

### 2.2 Air-Gapped Optical Audio & Data Transfer (Dynamic QR Stream)

#### Concept & Workflow
Enables secure, air-gapped transfer of small audio files (~2MB previews, stems, presets, sample packs) or SLM/LLM context memory states between isolated machines without active network cards, Wi-Fi, or Bluetooth.

1. **Optical Encoding:** Source machine compresses, serializes, and encrypts the payload into a high-density dynamic QR code video stream (low FPS visual transmission on screen).
2. **Optical Decoding via Webcam:** A second machine's webcam captures the visual QR stream in real-time, decrypts the payload chunks, applies Reed-Solomon error correction, and reconstructs the audio file or context data.
3. **In-Memory / System Injection:** The transferred payload (e.g. 2MB audio track or context JSON) is immediately ingested locally into the target application or system clipboard.

#### Stack & Reference
- **Reference:** [decimen-optical-transfer](https://github.com/bashalarmistalt/decimen-optical-transfer)
- **Tech Stack:** Lightweight Rust/C++ desktop utility or browser Lab App (`tunecamp-optical-transfer`), high-density QR video encoder/decoder, Reed-Solomon error correction library, optical camera reader pipeline.

#### Impact
Allows ultra-secure transfer of unreleased tracks, stems, sensitive presets, or local AI context memory between air-gapped studio computers and live performance rigs without network vulnerability.

---
