

## 📅 Week 8 — Object Detection

### Exercises

**1. Manual Labeling & Annotation Quality Discussion**

**Labeling guidelines for 10 images:**
1. **Tightness:** Box edges touch the outermost pixels of the object. No extra background.
2. **Consistency:** Same class always labeled the same way (e.g., "person" includes backpack or not? Decide and document).
3. **Occlusion handling:** If 50%+ visible, label. If less, mark as `occluded` in metadata.
4. **Truncation:** If object extends beyond frame, box goes to image edge; mark `truncated`.
5. **Ambiguity:** When uncertain between "car" and "truck," choose the class with higher confidence or flag for review.

**Annotation quality issues:**
- **Inter-annotator disagreement:** Two people draw different boxes for the same person. Solution: IoU > 0.7 between annotators required.
- **Small object omission:** Annotators miss distant pedestrians. Solution: Zoom tools + explicit "nothing to label" confirmation.
- **Class confusion:** "Dog" vs. "cat" for blurry animals. Solution: "Unknown" class option + confidence score.

**2. Run Pretrained Detector & Compare Mistakes**

Using a pretrained YOLOv8 model:

```python
from ultralytics import YOLO

model = YOLO("yolov8n.pt")  # Nano variant

# Run on three test images
results = model(["day_street.jpg", "night_street.jpg", "crowded_mall.jpg"])

for i, r in enumerate(results):
    print(f"\nImage {i+1}: {r.path}")
    print(f"Detections: {len(r.boxes)}")
    for box in r.boxes:
        print(f"  {model.names[int(box.cls)]}: {box.conf.item():.3f} at {box.xyxy[0].tolist()}")
```

**Typical mistakes observed:**

| Image | False Positives | False Negatives | Cause |
|-------|----------------|-----------------|-------|
| Day street | Fire hydrant as "person" (0.42) | None | Low confidence threshold; shape ambiguity |
| Night street | Three "cars" that are shadows | Two pedestrians in dark clothing | Poor illumination; training data bias toward lit objects |
| Crowded mall | Merged pair labeled as one "person" | Child partially behind adult | Occlusion; NMS threshold too aggressive |

**3. Confidence Threshold Sweep**

```python
thresholds = [0.1, 0.3, 0.5, 0.7, 0.9]
results_table = []

for thresh in thresholds:
    detections = model(image, conf=thresh)[0]
    results_table.append({
        "threshold": thresh,
        "detections": len(detections.boxes),
        "avg_confidence": detections.boxes.conf.mean().item() if len(detections.boxes) > 0 else 0
    })

# Typical output:
# threshold=0.1 → 47 detections (many false positives)
# threshold=0.3 → 23 detections (reasonable)
# threshold=0.5 → 12 detections (conservative)
# threshold=0.7 → 5 detections (high precision)
# threshold=0.9 → 1 detection (only sure things)
```

**Observation:** Lower thresholds increase recall (find more objects) but decrease precision (more false alarms). The optimal threshold depends on the application: security screening favors high recall (low threshold); autonomous driving favors high precision (high threshold) because false positives trigger dangerous braking.

**4. IoU Calculation by Hand**

```
Box A (Ground Truth):  x1=100, y1=100, x2=300, y2=400
Box B (Prediction):    x1=150, y1=120, x2=320, y2=380

Step 1: Calculate areas
Area A = (300-100) × (400-100) = 200 × 300 = 60,000
Area B = (320-150) × (380-120) = 170 × 260 = 44,200

Step 2: Find intersection
Intersect x1 = max(100, 150) = 150
Intersect y1 = max(100, 120) = 120
Intersect x2 = min(300, 320) = 300
Intersect y2 = min(400, 380) = 380

Intersection width  = 300 - 150 = 150
Intersection height = 380 - 120 = 260
Intersection area   = 150 × 260 = 39,000

Step 3: Find union
Union = Area A + Area B - Intersection
Union = 60,000 + 44,200 - 39,000 = 65,200

Step 4: Compute IoU
IoU = Intersection / Union = 39,000 / 65,200 = 0.598 ≈ 0.60
```

**Interpretation:** IoU = 0.60 means moderate overlap. In COCO evaluation, this counts as a "detection" (IoU ≥ 0.50) but not as "high quality" (IoU ≥ 0.75).

**5. Real-Time Detection Tradeoffs**

**Why accuracy is traded for speed:**

| Technique | Speed Gain | Accuracy Cost | How It Works |
|-----------|-----------|---------------|--------------|
| **Smaller input resolution** | 2-4× faster | Miss small objects | 640×640 → 320×320 halves compute |
| **Fewer feature layers** | 1.5× faster | Poor localization | Use P3-P4 only, skip P5 |
| **Lighter backbone** | 2× faster | Weak features | MobileNet vs. ResNet-50 |
| **INT8 quantization** | 1.5-3× faster | <2% mAP drop | 8-bit weights instead of 32-bit |
| **NMS simplification** | 10-20% faster | Duplicate boxes | Faster NMS approximations |
| **Batch size = 1** | Required for stream | No batching efficiency | Process each frame individually |

**The fundamental tension:** Real-time systems (30+ FPS) have ~33ms per frame. A high-accuracy model might take 100ms. To meet the deadline, you must reduce model capacity, input resolution, or post-processing complexity—all of which reduce accuracy. The optimal point is application-dependent: a traffic camera can tolerate missing a distant car; an autonomous vehicle cannot.

---

### Homework: Failure Analysis Table

| Failure Type | Example | Root Cause | Mitigation Strategy |
|--------------|---------|------------|---------------------|
| **False Positive** | Shadow labeled as "person" (conf=0.67) | Shape similarity between vertical shadow and human silhouette | Increase training data with hard negatives; add temporal consistency check (shadow doesn't move like person) |
| **False Positive** | Billboard face detected as "person" | 2D image of person triggers same features as real person | Add depth information or context (faces on billboards don't have bodies below) |
| **False Negative** | Child in stroller missed | Small object size; training data underrepresents children | Lower confidence threshold for small boxes; add data augmentation with small objects |
| **False Negative** | Person in dark clothing at night | Low illumination reduces contrast; model trained on well-lit images | Add low-light augmentation; apply preprocessing from Week 7 |
| **False Negative** | Occluded pedestrian (50% behind car) | Occlusion handling not robust | Use part-based detectors; leverage temporal tracking across frames |
| **Localization Error** | Box too large, includes background | Anchor box aspect ratio mismatch | Use anchor-free detector (FCOS, CenterNet) or optimize anchor sizes |
| **Class Confusion** | "Bus" detected as "truck" | Similar visual features for large vehicles | Fine-tune on domain-specific data; add semantic context (buses stop at bus stops) |
| **Duplicate Detection** | One person with two overlapping boxes | NMS threshold too lenient (IoU=0.5) | Lower NMS IoU threshold to 0.3; use soft-NMS |
| **Temporal Flicker** | Person appears/disappears across frames | Frame-by-frame detection inconsistency | Add object tracking (SORT/DeepSORT); apply temporal smoothing |
| **Scale Sensitivity** | Distant cars missed, nearby cars detected | Feature pyramid doesn't resolve small objects well | Increase input resolution; use specialized small-object detection head |

**Summary statistics to track:**
- **mAP@0.50:** Overall detection quality at loose localization
- **mAP@0.75:** Strict localization quality
- **mAP small/medium/large:** Where the model struggles by object size
- **False positive rate per image:** How much noise the system produces
- **Per-class AP:** Which classes need more data or architectural attention

---