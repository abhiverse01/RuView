# Soul Signature — References

**Status:** Research Specification (Pre-Implementation)
**Date:** 2026-05-24
**Author:** ruv

---

## 1. Internal Architecture Decision Records

All ADRs are located at `docs/adr/ADR-XXX-*.md` in this repository.

| ADR | Title | Relevance to soul signature |
|---|---|---|
| ADR-003 | RVF Cognitive Containers for CSI Data | RVF container format used by soul signature |
| ADR-004 | HNSW Vector Search for Signal Fingerprinting | HNSW index for person_track embedding search |
| ADR-005 | SONA Self-Learning Pose Estimation | LoRA adaptation, EWC regularization, environment profiles |
| ADR-007 | Post-Quantum Cryptography Secure Sensing | PQC cryptographic context; foundation for ADR-108/109 |
| ADR-010 | Witness Chains Audit Trail Integrity | Witness chain design; Ed25519 over frame bundles |
| ADR-014 | SOTA Signal Processing Algorithms | RuvSense pipeline: conjugate multiplication, Hampel filter, spectrogram, BVP |
| ADR-021 | Vital Sign Detection via rvdna Pipeline | Cardiac HR / respiratory extraction; bandpass filters; ADR-039 vitals packet |
| ADR-023 | Trained DensePose Model with RuVector Pipeline | CsiToPoseTransformer backbone; MPJPE baseline 91.7 mm |
| ADR-024 | Project AETHER — Contrastive CSI Embedding Model | Primary soul signature identity channel; 128-dim L2-normalized embedding; HNSW person_track index (>80% mAP target at 5 subjects) |
| ADR-027 | Project MERIDIAN — Cross-Environment Domain Generalization | Environment-disentangled embeddings; HardwareNormalizer; multi-room portability |
| ADR-029 | RuvSense Multistatic Sensing Mode | Multi-node mesh; 20 Hz DensePose; <30 mm jitter; person separation |
| ADR-030 | RuvSense Persistent Field Model | Field normal modes; SVD eigenstructure; perturbation extraction; longitudinal drift; adversarial detection; cross-room continuity |
| ADR-039 | ESP32-S3 Edge Intelligence Pipeline | Vitals packet wire format (magic `0xC511_0002`); HR/BR on-device extraction |
| ADR-075 | MinCut Person Separation | ruvector-mincut for multi-person track assignment |
| ADR-079 | Camera Ground-Truth Training | Paired camera + CSI training; skeletal proportions accuracy |
| ADR-082 | Pose Tracker Confirmed Output Filter | Pose tracker output confidence filtering |
| ADR-100 | Cog Packaging Specification | Ed25519 firmware signing; supply chain integrity |
| ADR-105 | Federated CSI Training | Federated AETHER fine-tuning; secure aggregation |
| ADR-106 | DP-SGD and Primitive Isolation | Differential privacy at training; biometric primitive isolation; (ε, δ)-DP budget |
| ADR-107 | Cross-Installation Federation | Cross-installation secure aggregation; DH key exchange |
| ADR-108 | Kyber Post-Quantum Key Exchange | Kyber-768 (NIST FIPS 203); hybrid X25519 + Kyber during migration |
| ADR-109 | Dilithium PQC Signatures | Dilithium-3 (NIST FIPS 204); hybrid Ed25519 + Dilithium; cog signing |
| ADR-110 | ESP32-C6 Firmware Extension | Wi-Fi 6 HE-LTF CSI (242 subcarriers); 802.15.4 time-sync; TWT; Ed25519 witness chain per-frame |
| ADR-113 | Multistatic Placement Strategy | Node placement geometry; coverage analysis |
| ADR-115 | Home Assistant Integration (HA-DISCO + HA-MIND) | Privacy mode; MQTT auto-discovery; semantic primitives layer under which soul signature operates |

---

## 2. AETHER and Contrastive Embedding Foundations

- Chen, T., Kornblith, S., Norouzi, M., & Hinton, G. (2020). **A Simple Framework for Contrastive Learning of Visual Representations** (SimCLR). *ICML 2020*. arXiv:2002.05709.
- Chen, T., Kornblith, S., Sohl-Dickstein, J., & Hinton, G. (2020). **Big Self-Supervised Models are Strong Semi-Supervised Learners** (SimCLR v2). *NeurIPS 2020*. arXiv:2006.10029.
- Bardes, A., Ponce, J., & LeCun, Y. (2022). **VICReg: Variance-Invariance-Covariance Regularization for Self-Supervised Learning**. *ICLR 2022*. arXiv:2105.04906.
- Grill, J.-B., et al. (2020). **Bootstrap Your Own Latent: A New Approach to Self-Supervised Learning** (BYOL). *NeurIPS 2020*. arXiv:2006.07733.
- Wang, T. & Isola, P. (2020). **Understanding Contrastive Representation Learning through Alignment and Uniformity on the Hypersphere**. *ICML 2020*. arXiv:2005.10242.

---

## 3. WiFi CSI Biometric Identification (Prior Art)

- **IdentiFi** (2025): Self-supervised WiFi-based identity recognition in multi-user smart environments. Contrastive pretraining in the signal domain produces identity-discriminative embeddings without spatial labels. *PMC:12115556*.
- **WhoFi** (2025): Transformer-based WiFi CSI encoding for person re-identification. 95.5% accuracy on NTU-Fi (18 subjects). Validates transformer backbones for CSI re-ID. arXiv:2507.12869.
- **Wi-PER81** (2025): Benchmark dataset of 162K wireless packets for WiFi-based person re-identification using Siamese networks. *Nature Scientific Data*, 2025. doi:10.1038/s41597-025-05804-0.
- **CAPC** (Context-Aware Predictive Coding, 2024): CPC + Barlow Twins for WiFi sensing. 24.7% accuracy improvement on unseen environments. arXiv:2410.01825.
- **SSL for WiFi HAR Survey** (2025): Comprehensive evaluation of SimCLR, VICReg, Barlow Twins, SimSiam on WiFi CSI. arXiv:2506.12052.

---

## 4. WiFi Sensing SOTA (Pose, Vitals, Gait)

- Geng, J., Huang, D., & De la Torre, F. (2022). **DensePose From WiFi**. *CMU*. arXiv:2301.00250.
- Adib, F., Kabelac, Z., Katabi, D., & Miller, R.C. (2015). **3D Tracking via Body Radio Reflections** (WiTrack). *NSDI 2015*.
- Wang, J., Gao, X., Zhang, K., & Liu, X. (2019). **Widar 3.0: Zero-Effort Cross-Domain Gesture Recognition with Wi-Fi**. *MobiSys 2019*.
- Zhao, M., Li, T., Abu Alsheikh, M., Tian, Y., Zhao, H., Torralba, A., & Katabi, D. (2018). **Through-Wall Human Pose Estimation Using Radio Signals**. *CVPR 2018*.
- Zhao, M., Adib, F., & Katabi, D. (2016). **Emotion Recognition Using Wireless Signals** (EQ-Radio). *MobiCom 2016*. (HRV from WiFi; cardiac biometric baseline)
- **PerceptAlign** (Chen et al., 2026): Geometry-conditioned cross-layout WiFi pose estimation. >60% cross-domain error reduction. Dataset: 21 subjects, 5 scenes, 18 actions. arXiv:2601.12252.
- **Person-in-WiFi 3D** (Yan et al., 2024): Multi-person 3D pose from WiFi. 91.7 mm MPJPE (single-person). *CVPR 2024*.
- **DGSense** (Zhou et al., 2025): Domain-invariant features for WiFi/mmWave/acoustic sensing. arXiv:2502.08155.
- **X-Fi** (Chen & Yang, 2025): Modality-invariant foundation model for human sensing. 24.8% MPJPE improvement on MM-Fi. *ICLR 2025*. arXiv:2410.10167.
- **AM-FM** (2026): First WiFi foundation model, pretrained on 9.2M CSI samples, 20 device types, 439 days. arXiv:2602.11200.
- Ma, Y., Zhou, G., Wang, S., Zhao, H., & Jung, W. (2018). **SignFi: Sign Language Recognition Using WiFi**. *ACM IMWUT*. arXiv:1806.04583.

---

## 5. Training Datasets Referenced

- **MM-Fi** (2022): Multi-Modal Non-Intrusive 4D Human Dataset — WiFi CSI, mmWave, LiDAR, RGB-D. 27 subjects, 40 actions, 5 environments, 320K samples. 56-subcarrier CSI, 17 COCO keypoints. [github.com/ybhbingo/MMFi_dataset]
- **Wi-Pose** (2022): WiFi-based 3D pose estimation dataset. Used in ADR-015.
- **NTU-Fi** (2022): 56 activities, WiFi CSI, 75 Hz sampling. Used for WhoFi evaluation.

---

## 6. Differential Privacy

- Abadi, M., Chu, A., Goodfellow, I., McMahan, H.B., Mironov, I., Talwar, K., & Zhang, L. (2016). **Deep Learning with Differential Privacy**. *CCS 2016*. [Moments Accountant; DP-SGD formulation used in ADR-106]
- Mironov, I. (2017). **Rényi Differential Privacy**. *CSF 2017*. [Alternative DP accounting; referenced in ADR-106 as future enhancement]
- Shokri, R., Stronati, M., Song, C., & Shmatikov, V. (2017). **Membership Inference Attacks Against Machine Learning Models**. *IEEE S&P 2017*. [Motivation for DP-SGD in ADR-106]

---

## 7. Cryptographic Standards

- **RFC 8032** (2017): Edwards-Curve Digital Signature Algorithm (EdDSA). [Ed25519; used in ADR-110 witness chain]
- **RFC 8439** (2018): ChaCha20 and Poly1305 for IETF Protocols. [At-rest encryption primitive specified in security.md §5]
- **RFC 9106** (2021): Argon2 Memory-Hard Function. [KDF for soul signature at-rest key derivation]
- **NIST FIPS 203** (2024): Module-Lattice-Based Key-Encapsulation Mechanism Standard (ML-KEM / Kyber). [ADR-108; post-quantum key exchange]
- **NIST FIPS 204** (2024): Module-Lattice-Based Digital Signature Standard (ML-DSA / Dilithium). [ADR-109; post-quantum signatures]
- **NIST SP 800-132 Draft** (2024): Recommendation for Password-Based Key Derivation. [Argon2id parameter guidance]

---

## 8. Biometric Standards (for Standards Awareness)

The soul signature is not currently certified to any of these standards but the
specification is designed with awareness of the relevant frameworks.

- **ISO/IEC 19794-1:2011**: Biometric data interchange formats — Part 1: Framework.
  [Top-level; soul signature's node/edge schema follows the typed-attribute-record
  philosophy of this standard]
- **ISO/IEC 19794-2:2011**: Biometric data interchange formats — Part 2: Finger
  minutiae data. [Structural analog for how the soul signature encodes per-channel
  discriminative features]
- **ISO/IEC 19794-4:2011**: Biometric data interchange formats — Part 4: Finger image data.
  [Image-container analog; soul signature extends the concept to vector-valued
  multi-channel templates]
- **ISO/IEC 29794-1:2016**: Biometric sample quality — Part 1: Framework.
  [Quality scoring framework; soul signature's per-node `confidence` field
  is conceptually analogous to ISO 29794 quality scores]
- **ISO/IEC 30107-3:2023**: Biometric presentation attack detection — Part 3:
  Testing and reporting. [Presentation attack (anti-spoofing) framework;
  the adversarial.rs module is the soul signature's PAD implementation]

---

## 9. Reading List for RF Biometrics Newcomers

Ordered from most accessible to most technical.

1. Adib, F. (2017). **Using Radio Reflections to See the World**. MIT PhD thesis. [Most accessible introduction to using RF for human sensing; covers WiVi, WiTrack, EQ-Radio]
2. Ma, Y., et al. (2019). **WiFi Sensing with Channel State Information: A Survey**. *ACM Computing Surveys*. doi:10.1145/3310194. [Comprehensive survey of CSI-based sensing approaches through 2019]
3. Wang, X., et al. (2023). **A Survey on WiFi Sensing: From Signal to Action**. *IEEE Internet of Things Journal*. [Updated survey through 2023; covers contrastive learning approaches]
4. Chen, T., et al. (2020). **A Simple Framework for Contrastive Learning** (SimCLR). arXiv:2002.05709. [Best starting point for understanding the contrastive learning approach used in AETHER]
5. Geng, J., et al. (2022). **DensePose From WiFi**. arXiv:2301.00250. [Direct ancestor of this codebase; describes the cross-modal CSI → DensePose mapping]
6. Abadi, M., et al. (2016). **Deep Learning with Differential Privacy**. CCS 2016. [Essential reading before any deployment collecting biometric data at training time]
