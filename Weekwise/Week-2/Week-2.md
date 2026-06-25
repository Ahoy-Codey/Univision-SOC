
## 📅 Week 2 — Python as a Tool

### Exercises

**1. Average Confidence Function**

```python
from typing import List

def average_confidence(scores: List[float]) -> float:
    """Calculate average of confidence scores."""
    if not scores:
        return 0.0
    return sum(scores) / len(scores)

# Example usage
scores = [0.92, 0.87, 0.95, 0.78]
print(average_confidence(scores))  # 0.88
```

**2. Detection as Dictionary**

```python
detection = {
    "label": "person",
    "confidence": 0.94,
    "box": {
        "x": 120,
        "y": 80,
        "width": 200,
        "height": 350
    }
}
```

**3. Filter by Confidence Threshold**

```python
from typing import List, Dict, Any

def filter_detections(
    detections: List[Dict[str, Any]], 
    threshold: float
) -> List[Dict[str, Any]]:
    """Return only detections with confidence >= threshold."""
    return [d for d in detections if d.get("confidence", 0) >= threshold]
```

**4. Test for Filter Function**

```python
import pytest
from detection_filter import filter_detections

def test_filter_removes_low_confidence():
    detections = [
        {"label": "person", "confidence": 0.9},
        {"label": "car", "confidence": 0.3},
        {"label": "dog", "confidence": 0.75},
    ]
    result = filter_detections(detections, threshold=0.5)
    assert len(result) == 2
    assert all(d["confidence"] >= 0.5 for d in result)
    assert result[0]["label"] == "person"
    assert result[1]["label"] == "dog"

def test_filter_empty_list():
    assert filter_detections([], 0.5) == []

def test_filter_all_below_threshold():
    detections = [{"confidence": 0.1}, {"confidence": 0.2}]
    assert filter_detections(detections, 0.5) == []
```

**5. Why Tests Matter Before Complexity**
Tests serve as executable documentation and safety nets. When a system is simple, bugs are obvious. But as you add threshold filtering, NMS (non-max suppression), tracking across frames, and different model backends, a change in one module can silently break another. Tests catch regressions immediately, let you refactor fearlessly, and force you to define expected behavior clearly. Writing tests *before* complexity means you're not scrambling to add them when you're already firefighting bugs.

---

### Homework: Tests for Threshold Filtering

The test suite above (Exercise 4) is your homework. Save it as `test_detection_filter.py` and run with:
```bash
pytest test_detection_filter.py -v
```

---