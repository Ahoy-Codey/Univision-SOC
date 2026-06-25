
## 📅 Week 6 — Images as Arrays

### Exercises

**1. Identify Height, Width, and Channels**

Using Python with OpenCV or PIL:

```python
import cv2

# Load a phone image
image = cv2.imread("phone_photo.jpg")  # BGR format by default
height, width, channels = image.shape

print(f"Shape: {image.shape}")
print(f"Height: {height} pixels")
print(f"Width: {width} pixels")  
print(f"Channels: {channels} (BGR)")
print(f"Total pixels: {height * width}")
print(f"Memory size: {image.nbytes / 1024:.1f} KB")

# Typical phone photo output:
# Shape: (3024, 4032, 3)
# Height: 3024 pixels
# Width: 4032 pixels
# Channels: 3 (BGR)
# Total pixels: 12,192,768
# Memory size: ~35.6 MB
```

**2. Convert to Grayscale & Information Lost**

```python
import cv2

color_image = cv2.imread("photo.jpg")
gray_image = cv2.cvtColor(color_image, cv2.COLOR_BGR2GRAY)

print(f"Original shape: {color_image.shape}")   # (H, W, 3)
print(f"Grayscale shape: {gray_image.shape}")   # (H, W)

# Grayscale conversion formula (luminance method):
# Y = 0.299*R + 0.587*G + 0.114*B
```

**Information lost:**
- **Color information:** A red fire truck and a green tree may have identical grayscale values if their luminance matches. The model can no longer distinguish "red object" from "green object."
- **Chromatic edges:** Boundaries defined by color contrast but not brightness contrast disappear. A blue object on a blue background of different hue becomes invisible.
- **Channel-specific features:** Some detectors use color histograms or skin-tone detection. These fail entirely on grayscale.

**What remains:** Spatial structure, texture, edges, and shape are preserved. For many detection tasks (faces, cars, pedestrians), grayscale is sufficient because shape matters more than color.

**3. Crop to Object Region**

```python
import cv2

image = cv2.imread("photo.jpg")
detection = {
    "box": {"x": 340, "y": 120, "width": 180, "height": 420}
}

x, y = detection["box"]["x"], detection["box"]["y"]
w, h = detection["box"]["width"], detection["box"]["height"]

# Crop using NumPy slicing: image[y:y+h, x:x+w]
cropped = image[y:y+h, x:x+w]

print(f"Cropped shape: {cropped.shape}")  # (420, 180, 3)
cv2.imwrite("cropped_object.jpg", cropped)
```

**Why crop:** Reduces input size for downstream processing, removes background clutter, and focuses compute on the region of interest.

**4. Resize to 640×640 & Discuss Distortion**

```python
import cv2

image = cv2.imread("photo.jpg")  # e.g., 3024 × 4032
resized = cv2.resize(image, (640, 640), interpolation=cv2.INTER_LINEAR)

print(f"Original aspect ratio: {4032/3024:.2f} (4:3)")
print(f"New aspect ratio: {640/640:.2f} (1:1)")
```

**Distortion analysis:**
- **Aspect ratio change:** 4:3 → 1:1 means horizontal compression. A person becomes thinner; a car becomes shorter.
- **Information loss:** Downscaling from 12M pixels to 409K pixels discards fine details. Small text becomes unreadable; distant faces lose features.
- **Interpolation artifacts:** Bilinear interpolation blends neighboring pixels, creating slight blurring. Nearest-neighbor preserves sharpness but introduces aliasing (jagged edges).
- **When it's acceptable:** Object detectors are trained on distorted images and learn aspect-ratio invariance. The tradeoff is acceptable if the model's input size is fixed (e.g., YOLO expects 640×640).
- **When it's harmful:** OCR, facial recognition, and measurement tasks require precise aspect ratios. Use letterboxing (padding with black bars) instead.

**5. Compare Original vs. Normalized Pixel Values**

```python
import numpy as np

# Original: uint8, range [0, 255]
original = cv2.imread("photo.jpg")
print(f"Original range: [{original.min()}, {original.max()}]")
print(f"Original dtype: {original.dtype}")  # uint8

# Normalization method 1: Scale to [0, 1]
normalized_01 = original.astype(np.float32) / 255.0
print(f"Normalized [0,1] range: [{normalized_01.min():.3f}, {normalized_01.max():.3f}]")

# Normalization method 2: ImageNet mean/std (for pretrained models)
mean = np.array([0.485, 0.456, 0.406])
std = np.array([0.229, 0.224, 0.225])
normalized_imagenet = (normalized_01 - mean) / std
print(f"ImageNet normalized range: [{normalized_imagenet.min():.3f}, {normalized_imagenet.max():.3f}]")

# Normalization method 3: Per-channel z-score
mean_per_channel = original.mean(axis=(0,1))
std_per_channel = original.std(axis=(0,1))
zscore = (original.astype(np.float32) - mean_per_channel) / std_per_channel
```

**Comparison:**

| Property | Original [0, 255] | Normalized [0, 1] | ImageNet Normalized |
|----------|-------------------|-------------------|---------------------|
| Range | 0 to 255 | 0.0 to 1.0 | ~-2.2 to +2.8 |
| Dtype | uint8 | float32 | float32 |
| Purpose | Storage, display | General ML input | Transfer learning |
| Gradient behavior | Large values, unstable | Stable, well-behaved | Zero-centered, unit variance |
| Memory | 1 byte/pixel | 4 bytes/pixel | 4 bytes/pixel |

**Why normalize:** Neural networks train faster and more stably when inputs are small and centered. Large values (255) can cause exploding gradients; non-zero-centered data shifts activation distributions.

---

### Homework: Before/After Images with Explanation

**Submission format:**

| Transformation | Before | After | Explanation |
|----------------|--------|-------|---------------|
| Grayscale | Color photo (3ch) | Gray photo (1ch) | Lost color, kept luminance. 2/3 data reduction. |
| Crop 640×640 | Full 12MP image | Object region only | Reduced from 3024×4032 to 420×180. 99.5% pixel reduction. |
| Resize 640×640 | 4:3 aspect | 1:1 square | Aspect ratio distortion. 97% pixel reduction. |
| Normalize | [0, 255] uint8 | [-2.2, +2.8] float32 | Zero-centered for model convergence. Values uninterpretable visually. |

**Written explanation (one paragraph):**
Images start as grids of unsigned 8-bit integers representing light intensity per color channel. Every preprocessing step transforms this representation for computational efficiency or model compatibility. Grayscale collapses three channels into one by luminance weighting, discarding chromatic information but preserving structure. Cropping reduces spatial dimensions to focus compute on relevant regions. Resizing standardizes input dimensions for neural networks but introduces geometric distortion unless letterboxing is used. Normalization rescales pixel values from [0, 255] to zero-centered distributions, which prevents gradient instability during training. These transformations form a pipeline: raw → grayscale → crop → resize → normalize → model input. Each step is a tradeoff between information preservation and computational tractability.

---