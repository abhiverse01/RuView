# BFLD SOTA Survey — Beamforming Feedback: State of the Art

## 1. BFI vs CSI: Physical-Layer Differences and Leakage Profiles

### 1.1 Channel State Information (CSI)

CSI is the raw complex channel frequency response (CFR) measured at the receiver across
all subcarriers and antenna pairs. Extracting CSI requires either (a) firmware
modifications on the receiving NIC (Atheros CSI Tool, Nexmon CSI patch for BCM43455c0
on Raspberry Pi 4/5) or (b) a specialized radio (software-defined radio with 802.11
decoders). The resulting matrix is typically Ntx × Nrx × Nsubcarrier complex floats —
dense, high-dimensional, and not transmitted over the air in standard operation.

This project's existing rvCSI runtime (`vendor/rvcsi/`) captures CSI via the Nexmon
firmware patch on Raspberry Pi hardware (ADR-095/096). The ESP32-S3 on COM9 cannot
produce CSI in the format needed for the full pipeline — it lacks the antenna count
and the firmware support for per-subcarrier phase extraction at the fidelity rvcsi
expects.

### 1.2 Beamforming Feedback Information (BFI)

BFI is fundamentally different: it is the compressed representation of the channel that
a STA (station/client) sends back to an AP (access point) so the AP can steer its beam
toward the client. The standard (IEEE 802.11ac/ax, section 9.4.1.52) defines the
compressed beamforming format as:

1. The AP transmits a Null Data Packet (NDP) sounding frame.
2. The STA measures the channel from the NDP, computes the singular-value decomposition
   V = U Sigma V^H, then compresses the right singular vectors using a series of Givens
   rotations.
3. The Givens rotation produces a set of angles: Phi (φ) angles in [0, 2π) and Psi (ψ)
   angles in [0, π/2). In 802.11ac these are quantized to 7 and 5 bits respectively; in
   802.11ax the default is 4 bits for φ and 2 bits for ψ.
4. The STA transmits a VHT/HE Compressed Beamforming frame (CBFR) containing those
   quantized angles, one set per active subcarrier (or per compressed subcarrier group),
   plus an SNR field per stream.

The CBFR is a **management-plane 802.11 frame, not an 802.3 data frame**. It is
transmitted before association encryption is negotiated; in WPA2/WPA3 deployments, the
beamforming sounding and feedback exchange happens in the clear because WPA2/WPA3
encrypt data frames only. Even 802.11ax (Wi-Fi 6/6E) with Protected Management Frames
(PMF) enabled does NOT encrypt action frames in the beamforming exchange by default on
commodity APs as of 2025 (NDSS 2025 finding, "Lend Me Your Beam",
https://www.ndss-symposium.org/ndss-paper/lend-me-your-beam-privacy-implications-of-plaintext-beamforming-feedback-in-wifi/).

**Key asymmetry**: extracting CSI requires physical access to a device and firmware
modification; extracting BFI requires only a WiFi adapter in monitor mode and a parser
for the CBFR frame format. Wi-BFI (Haque, Meneghello, Restuccia; ACM WiNTECH 2023,
https://arxiv.org/abs/2309.04408) is an open-source pip-installable tool that does
exactly this.

### 1.3 Why BFI Is Uniquely Dangerous

CSI is a research instrument — accessing it requires deliberate effort. BFI is a
production protocol artifact that any 802.11ac/ax STA broadcasts periodically as a
matter of course. The attack-surface implications:

- **No firmware modification needed** on the target device or AP.
- **Passive capture** is sufficient. Frames are broadcast in all directions, not
  beamformed, so a nearby attacker receives them at essentially the same SNR as the AP.
- **Structured leakage**: the Phi/Psi angle matrices encode a compressed but
  non-trivially-invertible representation of the spatial channel, which includes
  multipath geometry that is body-shaped — the human body is a dielectric obstacle whose
  shape and movement modulate the channel.
- **Regularity**: sounding happens at the AP's request, typically at 5–40 Hz in modern
  802.11ax deployments. A 60-second capture at 10 Hz produces 600 CBFR frames —
  sufficient for the BFId classifier to achieve >90% re-identification accuracy (ACM CCS
  2025, https://dl.acm.org/doi/10.1145/3719027.3765062).

---

## 2. Compressed Angle Matrices: The Identity Surface

### 2.1 Givens Rotation Reconstruction

The Phi/Psi angles encode a unitary matrix via the Givens rotation decomposition:

    V = G(N, N-1, φ_{N,N-1}, ψ_{N,N-1}) · G(N, N-2, ...) · ... · G(2,1, φ_{2,1}, ψ_{2,1}) · D

where D is a diagonal phase matrix. For a 2×2 MIMO system this is two angles; for a
4×4 system this is 12 angles. Each "column" in the BFI payload corresponds to one
subcarrier group (or every 4th subcarrier in 802.11ax, every 2nd in 802.11ac).

The resulting per-subcarrier angle sequence is a time-varying signature of the spatial
channel. Because the human body modulates the multipath channel, this sequence encodes
body-specific geometry. The BFId paper (https://dl.acm.org/doi/10.1145/3719027.3765062)
demonstrates that a supervised classifier trained on these sequences achieves identity
recognition on a 197-person dataset.

### 2.2 The AI/ML Compression Feedback Loop

IEEE 802.11 standardization is actively exploring AI/ML-based compression for
beamforming feedback (IEEE 802.11bn / Wi-Fi 8 study group, "Toward AIML Enabled WiFi
Beamforming CSI Feedback Compression", https://arxiv.org/html/2503.00412v1). This work
proposes neural codebooks that reduce feedback overhead. An important side effect: the
learned latent space of a neural BFI compressor may be *more* identity-discriminative
than the raw angles, because neural compression tends to preserve class-discriminative
variance. BFLD must be designed to handle compressed BFI encodings, not just the raw
Phi/Psi format.

---

## 3. Tooling Landscape

### 3.1 Wi-BFI

- **Source**: https://arxiv.org/abs/2309.04408 / https://github.com/kfoysalhaque/MU-MIMO-Beamforming-Feedback-Extraction-IEEE802.11ac
- **Capabilities**: real-time and offline extraction of BFAs from 802.11ac and 802.11ax;
  20/40/80/160 MHz; SU-MIMO and MU-MIMO; pip-installable.
- **Relevance to BFLD**: the BFLD extractor module (`extractor.rs`) must produce
  semantically equivalent output to Wi-BFI — i.e., per-subcarrier Phi/Psi angle arrays
  plus per-stream SNR — so that research results from the Wi-BFI ecosystem can be
  replicated on BFLD captures.

### 3.2 PicoScenes

- **Source**: https://www.semanticscholar.org/paper/Eliminating-the-Barriers-Demystifying-Wi-Fi-Baseband-Jiang-Zhou/...
- **Capabilities**: cross-NIC CSI and CBFR measurement platform; supports Intel AX200,
  AX210, Atheros AR9300, QCA6174; runs on Linux with custom kernel modules.
- **Relevance to BFLD**: PicoScenes can simultaneously capture CSI and BFI from the
  same frame sequence, enabling the CSI+BFI fusion path described in the BFLD spec
  (`csi_matrix` optional input). The rvcsi adapter layer (`vendor/rvcsi/`) already
  handles the Nexmon PCap format; a PicoScenes adapter is a future extension.

### 3.3 Nexmon CSI (BCM43455c0)

- **Source**: https://github.com/seemoo-lab/nexmon_csi
- **Hardware**: Raspberry Pi 4/5 with BCM43455c0 chip — the same hardware used in
  `cognitum-v0` (Pi 5 appliance in this fleet, see CLAUDE.local.md).
- **Capabilities**: per-subcarrier complex CSI in monitor mode; 4×4 MIMO on Pi 5 with
  BCM43456.
- **Relevance to BFLD**: the rvcsi nexmon adapter already routes PCap frames from this
  hardware into the wifi-densepose pipeline. BFI extraction on the same hardware requires
  an additional sniffer for CBFR frames alongside the CSI sniffer.

### 3.4 Atheros CSI Tool / iwlwifi CSI

- Legacy tools for Intel and Atheros NICs; require kernel module injection. Not relevant
  to the current hardware fleet (ESP32-S3 + Raspberry Pi 5), but documented here for
  completeness and for future Intel AX210-based deployments.

---

## 4. Identity Inference Attacks

### 4.1 BFId (ACM CCS 2025)

**Reference**: Todt, Morsbach, Strufe; KIT. ACM CCS 2025.
https://dl.acm.org/doi/10.1145/3719027.3765062
https://publikationen.bibliothek.kit.edu/1000185756
Dataset: https://ps.tm.kit.edu/english/bfid-dataset/index.php

BFId is the first published identity-inference attack that uses BFI exclusively (no
CSI). The methodology:

1. **Dataset**: 197 individuals, multiple sessions, multiple AP angles. Each subject
   walked a defined path while their STA continuously triggered BFI exchanges. CSI
   was also recorded simultaneously for comparison.
2. **Feature extraction**: temporal sequences of Phi/Psi angle matrices, windowed at
   varying lengths. Basic statistical features (mean, variance, cross-subcarrier
   correlation) fed a shallow classifier.
3. **Results**: re-identification accuracy >90% with as little as 5 seconds of BFI.
   Performance was robust to different walking styles and viewing angles — consistent
   with the hypothesis that anthropometric body shape (torso width, stride, limb
   geometry) rather than gait phase is the primary discriminator.
4. **Comparison to CSI**: BFI-only accuracy was comparable to CSI-only accuracy for
   identity tasks, despite BFI being a compressed representation. This confirms that
   the Givens angle compression preserves identity-discriminative variance.

### 4.2 LeakyBeam (NDSS 2025)

**Reference**: Xiao, Chen, He, Han, Han; Zhejiang U., NTU, KAIST. NDSS 2025.
https://www.ndss-symposium.org/ndss-paper/lend-me-your-beam-privacy-implications-of-plaintext-beamforming-feedback-in-wifi/

LeakyBeam targets occupancy detection (is a person present?) rather than identity.
Key findings:

- BFI is detectable through walls at 20 m range with commodity hardware.
- True positive rate 82.7%, true negative rate 96.7% in real-world evaluation.
- The attack works because BFI encodes motion-induced channel perturbations even through
  obstacles — the Phi/Psi angle variance changes measurably when a body enters the room.
- The defense (obfuscating BFI before transmission) requires minimal hardware changes.

**Implication for BFLD**: if a passive attacker with no relationship to the AP can
detect occupancy, then the BFLD node is implicitly broadcasting presence information
unless active obfuscation is deployed at the STA firmware level. BFLD cannot prevent
this passive attack — it can only ensure the *node's own output* does not additionally
leak identity.

### 4.3 Prior RF-Based Gait and Biometric Inference

Before BFI-specific attacks, the threat landscape was already established through
CSI-based attacks:

- **Gait from CSI**: WiGait (2017), Wi-Gait (ScienceDirect 2023,
  https://www.sciencedirect.com/science/article/abs/pii/S1389128623001962),
  Gait+Respiration ID (IEEE Xplore 2021,
  https://ieeexplore.ieee.org/document/9488277) all demonstrate >90% gait-based
  re-identification from standard WiFi.
- **Breathing biometrics**: Respiration rate and depth are person-specific at a
  population level. IEEE 802.11 CSI captures breathing as amplitude oscillations at
  0.1–0.5 Hz.
- **Anthropometric inference**: Hand size, torso width, and limb geometry modulate the
  channel; classifiers trained on activity data have been shown to leak anthropometrics
  as a side effect.

The BFId finding that BFI achieves comparable accuracy to CSI for identity is consistent
with this prior body of work — it simply demonstrates the attack is achievable with a
lower barrier to entry.

---

## 5. Privacy-Preserving Sensing: Current State of the Art

### 5.1 Differential Privacy on RF Embeddings

"Differentially Private Feature Release for Wireless Sensing: Adaptive Privacy Budget
Allocation on CSI Spectrograms" (https://arxiv.org/pdf/2512.20323) applies Laplace/
Gaussian mechanisms to CSI spectrograms, calibrating epsilon per subcarrier based on
empirical sensitivity. Results show meaningful reduction in identity-inference accuracy
while preserving activity-recognition utility at epsilon = 1.0–4.0.

BFLD's `identity_risk_score` could be used as an adaptive epsilon selector: high-risk
frames receive a tighter privacy budget (more noise), low-risk frames pass unmodified.
This is a forward-looking integration not in the current spec.

### 5.2 Federated / Local-Only Inference

The consensus across 2024–2025 literature on wireless federated learning
(https://arxiv.org/pdf/2603.19040, https://arxiv.org/pdf/2109.09142) is that
local differential privacy (LDP) with gradient perturbation is achievable on resource-
constrained edge devices. For BFLD's use case the critical property is simpler: the
identity embedding never needs to leave the node. There is no federated learning step
for identity. The risk score is a local computation whose output is published; the
embedding that produced it is not.

### 5.3 ZK Attestation for Sensing

ZK-SenseLM (https://arxiv.org/pdf/2510.25677) proposes zero-knowledge proofs that a
sensing model's output derives from legitimate data. This is architecturally close to
ADR-028's witness-bundle approach. Future BFLD work could use ZK proofs to attest that
the identity_risk_score was computed from the claimed input without revealing the input.

### 5.4 "Protecting Human Activity Signatures in Compressed IEEE 802.11 CSI Feedback"

(https://arxiv.org/pdf/2512.18529) — This 2024 paper directly addresses activity-
signature leakage in CBFR frames and proposes perturbation of Phi/Psi angles at the STA
before transmission. The defense is the dual of BFLD's approach: BFLD detects leakage
at the receiver; this paper proposes suppression at the transmitter. Both approaches
are complementary.

---

## 6. Relationship to Existing Project ADRs

**ADR-027 (MERIDIAN cross-environment generalization)**: BFLD's cross-room hash
rotation directly instantiates the "no cross-site correlation" invariant that MERIDIAN
assumes for privacy-safe multi-room deployment.

**ADR-028 (ESP32 capability audit + witness verification)**: The deterministic-proof
pattern (`verify.py` + SHA-256 expected hash) is the template for BFLD's own acceptance
test. BFLD must produce a deterministic frame hash given the same input — acceptance
criterion 6 in the spec.

**ADR-024 (AETHER contrastive CSI embedding)**: BFLD reuses the AETHER embedding
infrastructure for its identity_risk measurement. The risk score is a function of how
separable the current embedding is from the population of known embeddings.

**ADR-029/030 (RuvSense multistatic + field model)**: BFLD's `cross_perspective_
consistency` component of the risk formula requires correlation across multiple sensor
viewpoints — the multistatic infrastructure from ADR-029 provides this.

**ADR-032 (multistatic mesh security hardening)**: The BFLD threat model is a
superset of the security model in ADR-032. ADR-032 covers mesh compromise; BFLD adds
the passive sniffing threat at the management-plane layer.

---

## 7. Open Technical Questions

1. **BFI capture on ESP32-S3**: The ESP32-S3's `esp_wifi_csi_set_config` API provides
   CSI via the vendor-specific Espressif HT20 format. It does not expose VHT/HE CBFR
   frames. BFI capture on this hardware likely requires host-side sniffing (Pi 5 +
   Nexmon in monitor mode, already available on cognitum-v0).

2. **Quantization resolution degradation**: At 4 bits for φ and 2 bits for ψ (802.11ax
   defaults), the angle resolution is coarser than in 802.11ac (7/5 bits). The BFId
   paper used 802.11ac hardware. BFLD must validate that the identity_risk_score
   calibration remains valid at lower quantization.

3. **WiFi 7 (802.11be) changes**: 802.11be introduces multi-link operation (MLO) and
   may change the sounding/feedback cadence. BFLD's frame format (magic 0xBF1D_0001,
   version byte) is designed to accommodate future protocol versions.
