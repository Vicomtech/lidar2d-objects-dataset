# lidar2d-objects-dataset

Real, synthetic and hybrid 2D LiDAR scans (SICK TiM571, 572 beams) with beam-wise labels and object poses for six indoor objects, plus their high-resolution 3D meshes used for synthetic dataset generation.

The dataset used in this project is available in the [dataset repository](https://opendatasets.vicomtech.org/di11-lidar2d-objects/4e1e1fb2).

---

## Files

| File | Scans | 1 object | 2 objects | Other | Poses | Content |
|---|---:|---:|---:|---:|---|---|
| `real_train.json` | 87,404 | 41,376 | 46,021 | 7 background-only | no | augmented real training scans |
| `real_val.json` | 812 | 384 | 428 | – | yes | real validation scans (not augmented) |
| `real_test.json` | 1,223 | 544 | 679 | – | yes | real test scans (not augmented) |
| `synth_train.json` | 93,996 | 26,856 | 67,140 | – | yes | 4,476 scenes × 21 object combinations |
| `synth_val.json` | 882 | 252 | 630 | – | yes | 42 scenes × 21 |
| `synth_test.json` | 1,491 | 426 | 1,065 | – | yes | 71 scenes × 21 |
| `hyb_train.json` | 181,400 | 68,232 | 113,161 | 7 background-only | synthetic part | hybrid training set: real and synthetic training scans |

Models trained on the hybrid set use the real validation set (`real_val.json`) for validation and the real test set (`real_test.json`) for testing.

Each synthetic scene (wall layout and sensor pose) is rendered with every object combination: the 6 single objects and the 15 pairs. As a result, every class appears in 6 of the 21 scans of each scene, which gives 426 instances per class in `synth_test`.

`meshes/` — the six object meshes, STL format, units in millimetres.

## Object classes

| `value` | `name` in files | Class | Yaw-symmetry period |
|---:|---|---|---|
| 0 | – (implicit) | background | – |
| 1 | `carrier` | carrier | none |
| 2 | `ladder` | ladder | 180° |
| 3 | `pallet` | pallet | 180° |
| 4 | `rotChair` | rotating chair | 72° |
| 5 | `table` | table | 180° |
| 6 | `trashCan` | trash can | continuous (yaw not evaluated) |

Scans contain one object or two objects of different classes. One physical object per class was used.

## File format

Each file is one JSON object with a single key, `Measurements`, holding the list of scans:

```json
{
  "Measurements": [
    {
      "data":    [3.112, 3.108, ...],
      "label":   [{"value": 1, "name": "carrier0", "indices": [289, 290, ...]}],
      "objects": [{"name": "carrier0",
                   "position": {"x": 3.11244, "y": 0.62526},
                   "orientation": {"yaw": 140.35583}}],
      "id": 70
    }
  ]
}
```

| Field | Present in | Description |
|---|---|---|
| `data` | all | 572 ranges in metres, ordered by beam index: beam 0 is at −95° (left) and beam 571 at +95° (right), in steps of 1/3°. See *Angle convention*. |
| `label` | all | One entry per object. `indices` are the beam indices (0–571) hit by the object, and `value` is its class. Beams not listed are background (0). Entries never share beams. |
| `label[].name` | all except `real_train` | Object name; matches `objects[].name`. |
| `objects` | real val/test, all synthetic | One pose per labelled object, in the same order as `label`. |
| `position.x`, `position.y` | | Object position in metres, in the LiDAR reference frame (see *Angle convention*). The reference point on the object is described under *Meshes*. |
| `orientation.yaw` | | Object yaw in **degrees** about the vertical axis. It is not wrapped to a fixed interval (values range from negative to above 180°). |
| `id` | synthetic only | Scene index. IDs restart in each split, so they are not unique across files. |

### Range values

- **Real validation and test:** raw sensor values with 1 mm resolution (3 decimals). Beams without a valid return are stored as exactly `25.0` (0.47 % of test beams).
- **Real training:** augmented. Zero-mean Gaussian range noise (σ = 0.011 m) was added, so values have 5 decimals. A few values are slightly negative (≥ −0.07 m) or above 25 m.
- **Synthetic:** virtual LiDAR in Unreal Engine 5.5 with zero-mean Gaussian range noise (σ = 0.011 m). Every beam hits a surface (maximum about 20 m).
- In the paper, ranges are clamped to `[0, r_max]` with r_max = 5 m before use.

## Sensor and acquisition

- SICK TiM571, scanning plane parallel to the floor, mounted at 0.12 m.
- 572 beams over 190° (−95° to +95°), angular resolution 1/3°.

### Angle convention

- 0° points straight ahead of the LiDAR.
- Angles are negative to the left and positive to the right.
- With beam angle φ and range r, `x = r·cos φ` points forward and `y = r·sin φ` points to the right.
- Real data were recorded in a laboratory with movable walls. Objects were placed individually or in pairs.
- Scans were captured with the robot stationary, moving at 0.5 m/s or moving at 1.5 m/s, with no motion compensation.
- The real data were split into train, validation and test sets by configuration. Augmentation was applied to the training set only, after splitting: horizontal reflection, shifting the object to the leftmost or rightmost feasible position, displacing the background by 1, 2 or 3 m, and range noise.

## Meshes

Captured with a Scantech TrackScan-P550 optical 3D scanner. STL files, units in millimetres. The bounding boxes below are given in metres.

| Mesh | Faces | Bounding box L × W × H [m] |
|---|---:|---|
| carrier | 3,959,968 | 1.287 × 1.378 × 1.563 |
| ladder | 17,516,924 | 0.874 × 1.690 × 1.263 |
| pallet | 2,505,320 | 1.295 × 0.278 × 0.843 |
| rotating chair | 5,616,452 | 1.029 × 1.140 × 0.854 |
| table | 2,173,600 | 1.629 × 0.953 × 0.742 |
| trash can | 4,521,220 | 0.430 × 0.358 × 0.417 |

Simplified meshes with 0 < H < 0.14 are also included in the dataset.

### Mesh origin and symmetry centre

The origin of a mesh does not, in general, coincide with the physical symmetry centre of the object. For symmetry-aware pose evaluation, the paper estimates each class's symmetry centre from its mesh:

- **Objects with a rotational symmetry period:** the centre that best aligns the model with copies of itself rotated by that period.
- **Circular objects (trash can):** a least-squares circle fit.

TODO: say whether `position` in the JSON files is the mesh origin or the symmetry centre, and list each class's centre `[x, y]` in metres in the mesh frame (for example as `meshes/symmetry_centers.json`).

## Loading example (Python)

```python
import json, numpy as np

with open("real_test.json") as f:
    scans = json.load(f)["Measurements"]

s = scans[0]
r = np.asarray(s["data"], dtype=np.float32)          # (572,) ranges [m]
y = np.zeros(572, dtype=np.int64)                    # beam-wise labels
for lab in s["label"]:
    y[lab["indices"]] = lab["value"]
phi = np.deg2rad(np.linspace(-95, 95, 572))          # beam 0 at -95 deg (left)
xy = np.stack([r * np.cos(phi), r * np.sin(phi)], 1)  # x forward, y right
poses = [(o["name"], o["position"]["x"], o["position"]["y"], o["orientation"]["yaw"])
         for o in s.get("objects", [])]
```

`hyb_train.json` is about 920 MB; loading it with `json.load` needs several GB of RAM. Use `ijson` to stream it.

## Citation - TODO

```bibtex
@article{TODO,
  author  = {Mendizabal-Arrieta, I{\~n}igo and Barberena, Xuban and S{\'a}nchez, Jairo R. and Torres, Fernando},
  title   = {Semantic Segmentation and Model-Based Object Pose Estimation Using 2D LiDAR},
  journal = {TODO},
  year    = {TODO},
  doi     = {TODO}
}
```

## License

The data and meshes are released under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). You may share and adapt them for any purpose, including commercial use, provided you cite the paper above.

## Contact

imendizabal@vicomtech.org, jrsanchez@vicomtech.org
