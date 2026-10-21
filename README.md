# Video-Enhancer

Video-Enhancer is a project to enhance video delivery quality: making videos look better (restoration and enhancement) and reach viewers in the best quality their bandwidth allows (efficient encoding and delivery).

> **Status:** early stage. This README covers the project goals, a survey of existing methods and research, and the planned pipeline. Sections marked *planned* are not implemented yet.

---

## Table of contents

1. [What "video enhancement" means](#what-video-enhancement-means)
2. [Methods by task](#methods-by-task)
3. [Delivery-side quality](#delivery-side-quality)
4. [Measuring quality](#measuring-quality)
5. [Key papers and reports](#key-papers-and-reports)
6. [Open-source tools](#open-source-tools)
7. [Quick start with FFmpeg](#quick-start-with-ffmpeg)
8. [Proposed pipeline (planned)](#proposed-pipeline-planned)
9. [Roadmap](#roadmap)
10. [Contributors](#contributors)
11. [Contributing](#contributing)

---

## What "video enhancement" means

Video enhancement is a family of tasks. Each one fixes a different problem:

| Problem | Task | Typical approach |
|---|---|---|
| Low resolution / blurry | **Super-resolution (VSR)** | Deep networks that use neighbouring frames (BasicVSR++, VRT, Real-ESRGAN) |
| Grain, sensor noise | **Denoising** | Temporal filters, FastDVDnet, VRT |
| Blocky / banded compression artifacts | **Compression artifact removal** | Real-ESRGAN, deblocking filters |
| Shaky footage | **Stabilisation** | Motion estimation + smoothing (vid.stab, DIFRINT) |
| Choppy motion / low FPS | **Frame interpolation** | Optical-flow networks (RIFE, FILM, DAIN) |
| Dark / underexposed | **Low-light enhancement** | Zero-DCE, curve estimation, histogram methods |
| Motion blur | **Deblurring** | EDVR, VRT, DeblurGAN |
| Damaged faces | **Face restoration** | GFPGAN, CodeFormer |
| Black-and-white footage | **Colourisation** | DeOldify |
| SDR → HDR | **Inverse tone mapping** | HDRTVNet, learned tone curves |

---

## Methods by task

### 1. Classical (non-AI) methods
Fast, predictable and good for real-time use:
- **Interpolation upscaling:** bicubic, Lanczos.
- **Sharpening:** unsharp mask.
- **Denoising:** non-local means, BM3D/VBM4D, temporal filters such as `hqdn3d`.
- **Contrast / colour:** histogram equalisation, CLAHE, gamma correction.
- **Stabilisation:** feature tracking + trajectory smoothing (`vid.stab`).
- **Frame-rate conversion:** motion-compensated interpolation (`minterpolate`).

### 2. Deep-learning video super-resolution
- **EDVR (2019):** deformable-convolution alignment plus fusion; won the NTIRE 2019 video restoration challenges.
- **BasicVSR / BasicVSR++ (2021):** bidirectional recurrent propagation with flow-guided deformable alignment. A strong, efficient baseline.
- **VRT / RVRT (2022):** transformer-based models that handle SR, deblurring and denoising in one framework.
- **Real-ESRGAN (2021):** trained on synthetic "real-world" degradations (blur, noise, JPEG). Works per frame and is very practical for low-quality web video.
- **Diffusion-based VSR (2023+):** e.g. Upscale-A-Video. Creates convincing detail but is slow and can hallucinate.

### 3. Frame interpolation
- **DAIN (2019):** depth-aware flow.
- **RIFE (2020/2022):** real-time intermediate-flow estimation. Popular for 30→60 fps.
- **FILM (2022):** handles large motion well.

### 4. Denoising and low-light
- **FastDVDnet (2020):** real-time video denoising without explicit flow.
- **Zero-DCE (2020):** learns light-enhancement curves without paired training data.

### 5. Face restoration and colourisation
- **GFPGAN (2021)** and **CodeFormer (2022)** for faces.
- **DeOldify** for colourising old footage.

---

## Delivery-side quality

Making the video *look better* is only half of it. The other half is getting the best possible quality *to the viewer*:

- **Modern codecs:** H.264 → HEVC (H.265) → **AV1** / VVC (H.266). AV1 saves roughly 30% bitrate compared with HEVC at similar quality.
- **Per-title / per-shot encoding:** choose the bitrate ladder for each video (or scene) based on its complexity instead of using one fixed ladder (Netflix, 2015; "Dynamic Optimizer", 2018).
- **Adaptive bitrate streaming (ABR):** HLS / MPEG-DASH with algorithms such as BOLA or MPC that switch quality based on network conditions.
- **Content-aware encoding settings:** CRF / constant-quality modes, film-grain synthesis in AV1 (strip the grain before encoding and re-synthesise it on playback).
- **Client-side enhancement:** upscale lower-bitrate streams on the device, e.g. NVIDIA RTX Video Super Resolution, or NAS (Neural Adaptive Streaming, OSDI 2018).

---

## Measuring quality

| Metric | Type | Notes |
|---|---|---|
| **PSNR** | Full-reference | Simple. Correlates poorly with perception. |
| **SSIM / MS-SSIM** | Full-reference | Structural similarity. |
| **VMAF** | Full-reference | Netflix's perceptual metric and the industry standard for streaming. |
| **LPIPS** | Full-reference | Learned perceptual distance. |
| **NIQE / BRISQUE** | No-reference | When there is no original to compare against. |
| **DOVER / FAST-VQA** | No-reference (deep) | Modern learned video quality assessment. |

---

## Key papers and reports

**Surveys**
- Liu et al., *Video Super-Resolution Based on Deep Learning: A Comprehensive Survey*, 2020. [arXiv:2007.12928](https://arxiv.org/abs/2007.12928)
- NTIRE / AIM workshop challenge reports (CVPR / ECCV), published every year for video SR, deblurring and compressed-video enhancement. A good snapshot of the state of the art.

**Super-resolution and restoration**
- Wang et al., *EDVR: Video Restoration with Enhanced Deformable Convolutional Networks*, CVPRW 2019. [arXiv:1905.02716](https://arxiv.org/abs/1905.02716)
- Chan et al., *BasicVSR++: Improving Video Super-Resolution with Enhanced Propagation and Alignment*, CVPR 2022. [arXiv:2104.13371](https://arxiv.org/abs/2104.13371)
- Liang et al., *VRT: A Video Restoration Transformer*, 2022. [arXiv:2201.12288](https://arxiv.org/abs/2201.12288)
- Wang et al., *Real-ESRGAN: Training Real-World Blind Super-Resolution with Pure Synthetic Data*, ICCVW 2021. [arXiv:2107.10833](https://arxiv.org/abs/2107.10833)
- Liang et al., *SwinIR: Image Restoration Using Swin Transformer*, 2021. [arXiv:2108.10257](https://arxiv.org/abs/2108.10257)
- Zhou et al., *Upscale-A-Video: Temporal-Consistent Diffusion Model for Real-World Video Super-Resolution*, CVPR 2024. [arXiv:2312.06640](https://arxiv.org/abs/2312.06640)

**Frame interpolation**
- Bao et al., *DAIN: Depth-Aware Video Frame Interpolation*, CVPR 2019. [arXiv:1904.00830](https://arxiv.org/abs/1904.00830)
- Huang et al., *RIFE: Real-Time Intermediate Flow Estimation for Video Frame Interpolation*, ECCV 2022. [arXiv:2011.06294](https://arxiv.org/abs/2011.06294)
- Reda et al., *FILM: Frame Interpolation for Large Motion*, ECCV 2022. [arXiv:2202.04901](https://arxiv.org/abs/2202.04901)

**Denoising, low-light, faces**
- Tassano et al., *FastDVDnet: Towards Real-Time Deep Video Denoising Without Flow Estimation*, CVPR 2020. [arXiv:1907.01361](https://arxiv.org/abs/1907.01361)
- Guo et al., *Zero-DCE: Zero-Reference Deep Curve Estimation for Low-Light Image Enhancement*, CVPR 2020. [arXiv:2001.06826](https://arxiv.org/abs/2001.06826)
- Wang et al., *GFPGAN: Towards Real-World Blind Face Restoration with Generative Facial Prior*, CVPR 2021. [arXiv:2101.04061](https://arxiv.org/abs/2101.04061)
- Zhou et al., *CodeFormer: Towards Robust Blind Face Restoration with Codebook Lookup Transformer*, NeurIPS 2022. [arXiv:2206.11253](https://arxiv.org/abs/2206.11253)

**Delivery and quality**
- Netflix Tech Blog, *Per-Title Encode Optimization* (2015) and *Toward a Practical Perceptual Video Quality Metric* (VMAF, 2016).
- Yeo et al., *Neural Adaptive Content-aware Internet Video Delivery (NAS)*, OSDI 2018.

---

## Open-source tools

| Tool | Use |
|---|---|
| [FFmpeg](https://ffmpeg.org/) | Decode, filter and encode everything |
| [Real-ESRGAN](https://github.com/xinntao/Real-ESRGAN) | Practical AI upscaling + artifact removal |
| [BasicSR](https://github.com/XPixelGroup/BasicSR) | Training/inference toolbox (EDVR, BasicVSR, …) |
| [MMagic](https://github.com/open-mmlab/mmagic) | OpenMMLab toolbox incl. BasicVSR++ |
| [VRT](https://github.com/JingyunLiang/VRT) | Transformer video restoration |
| [RIFE](https://github.com/hzwer/ECCV2022-RIFE) | Frame interpolation |
| [FILM](https://github.com/google-research/frame-interpolation) | Large-motion interpolation |
| [GFPGAN](https://github.com/TencentARC/GFPGAN) / [CodeFormer](https://github.com/sczhou/CodeFormer) | Face restoration |
| [DeOldify](https://github.com/jantic/DeOldify) | Colourisation |
| [Video2X](https://github.com/k4yt3x/video2x) | End-to-end upscaling/interpolation wrapper |
| [VMAF](https://github.com/Netflix/vmaf) | Perceptual quality measurement |

---

## Quick start with FFmpeg

Classical enhancements you can try today:

```bash
# Upscale to 1080p with Lanczos and light sharpening
ffmpeg -i input.mp4 -vf "scale=-2:1080:flags=lanczos,unsharp=5:5:0.8" -c:a copy upscaled.mp4

# Denoise (temporal + spatial)
ffmpeg -i input.mp4 -vf "hqdn3d=4:3:6:4" -c:a copy denoised.mp4

# Brighten and boost contrast/saturation
ffmpeg -i input.mp4 -vf "eq=brightness=0.05:contrast=1.15:saturation=1.2" -c:a copy graded.mp4

# Interpolate to 60 fps (motion-compensated)
ffmpeg -i input.mp4 -vf "minterpolate=fps=60:mi_mode=mci" -c:a copy smooth60.mp4

# Stabilise (two passes, requires libvidstab)
ffmpeg -i input.mp4 -vf vidstabdetect -f null -
ffmpeg -i input.mp4 -vf vidstabtransform=smoothing=30,unsharp=5:5:0.8 -c:a copy stable.mp4

# Encode efficiently with AV1 (SVT-AV1)
ffmpeg -i enhanced.mp4 -c:v libsvtav1 -crf 30 -preset 6 -c:a copy out_av1.mp4

# Measure quality against the original with VMAF
ffmpeg -i enhanced.mp4 -i reference.mp4 -lavfi libvmaf -f null -
```

AI upscaling with Real-ESRGAN (after installing it from its repo):

```bash
python inference_realesrgan_video.py -i input.mp4 -n realesr-animevideov3 -s 2 -o results/
```

---

## Proposed pipeline (planned)

```
            ┌──────────┐   ┌─────────────┐   ┌──────────────┐   ┌───────────┐   ┌──────────────┐
 input ───▶ │ Analyse  │──▶│  Restore    │──▶│  Enhance     │──▶│  Encode   │──▶│  Evaluate    │
 video      │ (probe,  │   │ denoise,    │   │ super-res,   │   │ AV1/HEVC, │   │ VMAF, PSNR,  │
            │ quality) │   │ deblock,    │   │ interpolate, │   │ per-title │   │ SSIM, report │
            └──────────┘   │ stabilise   │   │ colour/light │   │ ABR ladder│   └──────────────┘
                           └─────────────┘   └──────────────┘   └───────────┘
```

1. **Analyse:** read resolution, fps and bitrate with `ffprobe`, and estimate quality with a no-reference metric to decide which steps are needed.
2. **Restore:** remove noise, compression artifacts and shake *before* upscaling, so artifacts don't get magnified.
3. **Enhance:** AI super-resolution (Real-ESRGAN / BasicVSR++), optional frame interpolation (RIFE) and low-light/colour correction.
4. **Encode:** content-aware AV1/HEVC encoding and an adaptive-bitrate ladder (HLS/DASH).
5. **Evaluate:** compare against the source with VMAF/SSIM and write a quality report.

---

## Roadmap

- [ ] CLI wrapper around FFmpeg for classical enhancement presets
- [ ] Real-ESRGAN / BasicVSR++ super-resolution stage (GPU)
- [ ] RIFE frame interpolation stage
- [ ] Automatic analysis step that picks the right enhancements
- [ ] Per-title AV1 encoding + HLS/DASH ladder generation
- [ ] VMAF-based evaluation and HTML quality report
- [ ] Simple web UI for uploading and comparing before/after

---

## Contributors

| | Name | GitHub |
|---|---|---|
| <img src="https://github.com/riteshtummalawork.png" width="40"> | Ritesh Tummala | [@riteshtummalawork](https://github.com/riteshtummalawork) |
| <img src="https://github.com/krtk06.png" width="40"> | Kartheek Nistala | [@krtk06](https://github.com/krtk06) |

## Contributing

1. Fork or clone the repo and create a feature branch.
2. Keep each enhancement stage in its own module with a clear input/output contract (frames or video file in, frames or video file out).
3. Include before/after samples and VMAF numbers in pull requests that change output quality.

## License

No license selected yet. Add a `LICENSE` file (e.g. MIT or Apache-2.0) before making the repository public.
