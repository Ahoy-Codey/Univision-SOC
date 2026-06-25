

## 📅 Week 7 — Preprocessing & Frame Efficiency

### Exercises

**1. Apply Blur: When It Helps vs. Hurts**

```python
import cv2
import numpy as np

image = cv2.imread("noisy.jpg")

# Gaussian blur (kernel size must be odd)
blurred = cv2.GaussianBlur(image, (5, 5), sigmaX=1.5)

# Median blur (better for salt-and-pepper noise)
median_blurred = cv2.medianBlur(image, 5)
```

**When blur helps:**
- **Noise reduction:** Gaussian blur smooths high-frequency sensor noise before edge detection, preventing Canny from detecting noise as edges.
- **Downsampling anti-aliasing:** Blur before resize prevents moiré patterns and aliasing artifacts.
- **Preprocessing for OCR:** Slight blur improves text recognition by merging fragmented strokes.

**When blur hurts:**
- **Edge detection:** Excessive blur rounds sharp corners and merges close edges. Two nearby objects become one blob.
- **Small object detection:** A blur kernel larger than the object itself can erase it entirely. A 5×5 kernel on a 640×640 image affects a 28×28 pixel object significantly.
- **Fine texture analysis:** Blur destroys texture patterns used for material classification or defect detection.
- **Metric accuracy:** PSNR/SSIM comparisons against ground truth penalize blur as distortion.

**Rule of thumb:** Blur kernel size should be smaller than the smallest feature you need to preserve. For a 640×640 detector looking for 20×20 objects, keep kernel ≤ 3×3.

**2. Thresholding for Object Separation**

```python
import cv2

image = cv2.imread("dark_background.jpg", cv2.IMREAD_GRAYSCALE)

# Simple binary thresholding
_, binary = cv2.threshold(image, 127, 255, cv2.THRESH_BINARY)

# Adaptive thresholding (handles uneven lighting)
adaptive = cv2.adaptiveThreshold(
    image, 255, 
    cv2.ADAPTIVE_THRESH_GAUSSIAN_C, 
    cv2.THRESH_BINARY, 
    blockSize=11, 
    C=2
)

# Otsu's automatic thresholding
_, otsu = cv2.threshold(image, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)
```

**Explanation:** Thresholding converts grayscale to binary by comparing each pixel to a threshold. Pixels above the threshold become white (object); below become black (background). Adaptive thresholding computes a local threshold for each pixel based on its neighborhood, which handles uneven illumination where global thresholding fails.

**3. Canny Edge Detection: Low vs. High Thresholds**

```python
import cv2

image = cv2.imread("scene.jpg", cv2.IMREAD_GRAYSCALE)
blurred = cv2.GaussianBlur(image, (5, 5), 1.0)

# Low thresholds: more edges, more noise
edges_low = cv2.Canny(blurred, threshold1=50, threshold2=100)

# High thresholds: fewer edges, cleaner result
edges_high = cv2.Canny(blurred, threshold1=150, threshold2=300)
```

**Comparison:**

| Threshold Pair | Result | Use Case |
|----------------|--------|----------|
| 50 / 100 | Many edges, includes noise and texture | Finding all possible contours; texture analysis |
| 150 / 300 | Few edges, only strong boundaries | Object boundary detection; clean line extraction |

**How Canny works:** (1) Gaussian blur reduces noise, (2) Sobel operators find gradient magnitude/direction, (3) Non-maximum suppression thins edges to 1 pixel, (4) Hysteresis thresholding keeps pixels above high threshold, and keeps pixels above low threshold only if connected to strong edges. Low thresholds let weak edges survive; high thresholds demand strong evidence.

**4. Duplicate Frame Removal Saves Compute**

**The problem:** A security camera at 30 FPS sends 1,800 frames per minute. If nothing moves, 1,799 frames are identical (or nearly so).

**Perceptual hashing approach:**

```python
import cv2
import imagehash
from PIL import Image

def is_duplicate(frame1, frame2, threshold=5):
    """Returns True if frames are near-duplicates."""
    hash1 = imagehash.phash(Image.fromarray(frame1))
    hash2 = imagehash.phash(Image.fromarray(frame2))
    # Hamming distance: 0 = identical, >10 = different
    return (hash1 - hash2) < threshold

# Pipeline integration
last_hash = None
for frame in video_stream:
    current_hash = imagehash.phash(Image.fromarray(frame))
    if last_hash and (current_hash - last_hash) < 5:
        continue  # Skip duplicate
    last_hash = current_hash
    process(frame)  # Only run detection on changed frames
```

**Compute savings:**
- **Without deduplication:** 30 FPS × 60s × 10ms inference = 18s of compute per minute (30% of real-time capacity).
- **With deduplication (static scene):** 1 frame × 10ms = 10ms per minute (0.0017% capacity).
- **Realistic (10% motion):** 3 FPS effective × 10ms = 180ms per minute (0.3% capacity).

**Tradeoffs:** Perceptual hashing adds ~2ms overhead per frame. False positives (missed motion) can occur with very slow movement. Tuning the threshold balances sensitivity vs. savings.

**5. Preprocessing Chain for Low-Light Footage**

Based on current research, effective low-light preprocessing follows a two-stage approach: algorithmic brightness normalization followed by learned residual correction .

```
┌─────────────────────────────────────────────────────────────┐
│           LOW-LIGHT PREPROCESSING PIPELINE                  │
├─────────────────────────────────────────────────────────────┤
│  1. DENOISING                                               │
│     ├─ Input: Raw low-light image (noisy, dark)             │
│     ├─ Method: DnCNN or bilateral filter                   │
│     └─ Output: Noise-reduced image                           │
│                                                             │
│  2. BRIGHTNESS ENHANCEMENT                                  │
│     ├─ Method: CPGA-Net (frozen) or Gamma correction        │
│     ├─ Also apply: CLAHE on L-channel (local contrast)     │
│     └─ Output: Brightness-normalized views (multi-channel)   │
│                                                             │
│  3. CONTRAST OPTIMIZATION                                   │
│     ├─ Method: Histogram equalization or curve adjustment   │
│     └─ Output: Full dynamic range utilized                   │
│                                                             │
│  4. COLOR CORRECTION (learned)                              │
│     ├─ Input: Concatenated preprocessed views (9 channels)    │
│     ├─ Method: Lightweight depthwise-separable U-Net         │
│     └─ Output: Final enhanced image                          │
│                                                             │
│  5. VALIDATION                                              │
│     ├─ Check: Mean brightness in [80, 180] (8-bit)         │
│     ├─ Check: No clipping (>5% pixels at 0 or 255)        │
│     └─ Fallback: Re-run with gentler enhancement           │
└─────────────────────────────────────────────────────────────┘
```

**Why this order matters:** Denoising must come first because brightness enhancement amplifies noise. Brightness correction must precede contrast operations because histogram equalization on a dark image produces poor results. The learned color corrector receives normalized inputs, so it focuses on residual color shifts rather than extreme brightness recovery .

---

### Homework: Low-Light Preprocessing Policy

**Policy Document:**

```
LOW-LIGHT PREPROCESSING POLICY v1.0
====================================

1. TRIGGER CONDITIONS
   Apply enhanced preprocessing when:
   - Mean brightness < 50 (8-bit scale)
   - >20% pixels below intensity 30
   - Detection confidence drops >30% vs. daytime baseline

2. PIPELINE STAGES (in order)

   Stage 1: Noise Reduction
   ├─ Algorithm: Bilateral filter (d=9, sigmaColor=75, sigmaSpace=75)
   ├─ Rationale: Edge-preserving unlike Gaussian blur
   └─ Skip if: Image already denoised by camera ISP

   Stage 2: Brightness Normalization
   ├─ Algorithm: Gamma correction (γ=0.4) + CLAHE (clipLimit=2.0, tileGrid=8×8)
   ├─ Rationale: Algorithmic methods normalize distribution without training data
   └─ Output: 3 complementary views concatenated to 9 channels

   Stage 3: Learned Color Correction
   ├─ Model: Depthwise-separable U-Net (338K params, fp16)
   ├─ Input: 9-channel preprocessed tensor
   ├─ Output: 3-channel RGB
   └─ Fallback: If model unavailable, use Stage 2 output directly

   Stage 4: Quality Validation
   ├─ Check mean brightness ∈ [80, 180]
   ├─ Check clipping ratio < 5%
   ├─ Check no-reference quality score (NIQE) < 5.0
   └─ On failure: Log warning, apply gentler gamma (γ=0.6), retry once

3. PERFORMANCE BUDGET
   - Total preprocessing latency: <50ms per frame
   - Memory overhead: <5MB for 1920×1080 input
   - If budget exceeded: Skip Stage 3, use Stage 2 only

4. MONITORING
   - Log preprocessing stage timings
   - Track detection accuracy with/without preprocessing
   - Alert if >10% frames fail quality validation
```
