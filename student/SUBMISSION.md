# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Điền file này rồi commit. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Hoàng Trung Anh
- MSSV: 2A202602521
- Email: ht.anh00411@gmail.com
- Link repo (fork): https://github.com/trungKoiKa/K4-Track4-Day23-HoangTrungAnh-2A202602521-Sensor-Fusion-Student
- Commit hash nộp (`git rev-parse HEAD`): commit `CP6` cuối cùng trên `main` (hash 40 ký tự nộp trên LMS); artifacts chấm điểm sinh ở commit `86c1cf6` (CP5)

## Tóm tắt kết quả

- `fusion_mode` (bắt buộc `compare`), `frames`, `segment`, `seed`: `compare`, `[0, 198]` (199 frame), `training_segment-10094743350625019937_3420_000_3440_000_with_camera_labels.tfrecord`, `seed = 0`
- `detection.precision`, `detection.recall`, `detection.tp/fp/fn`: precision 0.9586, recall 0.4021, tp/fp/fn = 1019 / 44 / 1515
- `tracking.lidar.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 0.3462 m, 1000, 119.848, 0, 1534, 5.025
- `tracking.fused.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 0.3712 m, 1000, 137.753, 0, 1534, 5.025
- Giải thích khác biệt hai mode, đọc RMSE cùng số ghép và ghost/miss: xem phân tích ngay dưới.

**Về segment.** Máy em không có segment mặc định `1005081002024129653_5313_150_5333_150`,
chỉ có segment #2 (`10072…`) và #3 (`10094…`) trong danh sách khóa học. Chạy thử 25 frame
đầu của #2 cho `valid_gt = 0` ở mọi frame (không có xe trong cửa sổ BEV), RMSE = `null`,
nên em dùng segment #3 cho lần chấm điểm; mọi tham số khác giữ đúng quy định
(`frame_start: 0`, `frame_end: 198`, `--fusion compare --seed 0`).

**Phân tích hai mode.**

| Chỉ số | LiDAR | Fused | Ghi chú |
|---|---|---|---|
| RMSE (m) | 0.3462 | 0.3712 | fused − lidar = **+0.025 m** (≤ 0.05 m) |
| matches | 1000 | 1000 | cùng số cặp ghép → so sánh RMSE trên cùng tập cặp |
| ghost_track_frames | 0 | 0 | `precision_track` = 1000 / (1000 + 0) = **1.00** |
| missed_gt_frames | 1534 | 1534 | |
| mean_confirmed_tracks | 5.025 | 5.025 | |
| coverage = matches / det_tp | 1000 / 1019 = **0.981** | 0.981 | |

- `matches`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks` **giống hệt**
  giữa hai mode: camera không làm thay đổi tập track confirmed. Đúng như thiết kế
  track-then-fuse — chỉ lượt LiDAR tạo / chấm score / xoá track, camera chỉ update trạng thái EKF.
  Vì vậy khác biệt duy nhất nằm ở **độ chính xác vị trí** (`sum_sq_err` 119.85 → 137.75).
- Fused **kém hơn nhẹ** 0.025 m. Lý do: đo LiDAR có σ = 0.1 m trên cả 3 trục, rất chính xác;
  đo camera là tâm hộp 2D **có nhiễu** (σ = 5 px), chỉ 2 chiều (hướng nhìn), không có độ sâu.
  Tâm hộp 2D trên ảnh cũng không trùng hoàn toàn với hình chiếu tâm hộp 3D mà metric dùng làm
  GT, nên update camera kéo vị trí lệch nhẹ. Ví dụ frame 60 trong `grade_run.log`: cả hai mode
  đều 4 matches, nhưng `sum_sq_err` 0.284 (lidar) so với 0.335 (fused).
- `missed_gt_frames = 1534` lớn chủ yếu do **detector**: `det_fn = 1515` (recall 0.40 — xe xa
  hoặc bị che không được phát hiện). Tracker gần như không bỏ rơi detection nào
  (coverage 0.98); phần miss còn lại chủ yếu là các frame đầu trước khi track được xác nhận.
- Không kết luận chỉ từ RMSE: tracker có RMSE thấp mà nhiều ghost hoặc ít matches vẫn kém.
  Ở đây ghost = 0 và matches bằng nhau nên so sánh RMSE giữa hai mode là công bằng.

Chạy từ root repo:

```bash
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
```

`rmse = sqrt(sum_sq_err/matches)` trên vị trí 3D của confirmed tracks ghép
một-một với GT xe trong cửa sổ BEV, gate XY **2.0 m**; `null` nếu không có cặp.
Camera dùng tâm hộp 2D ground-truth FRONT có nhiễu seeded, **không** dùng camera
detector. Kết quả này không đo hiệu quả một perception system độc lập với GT.

`grade_run.log` là JSONL, mỗi `(mode,frame)` đúng một record với các trường:
`mode`, `frame`, `det_tp`, `det_fp`, `det_fn`, `valid_gt`, `confirmed`, `matches`,
`sum_sq_err`, `ghosts`, `misses`. Đảm bảo `matches+ghosts==confirmed` và
`matches+misses==valid_gt`; tổng/trung bình record phải khớp `metrics.json`.
File per-mode `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`,
`grade_run_fused.log` được giữ để đối chiếu.

## Giải thích ngắn (Parts E–H — tự viết)

1. Khác biệt đo lidar 3D và camera 2D trong EKF (`z`, `R`)?

   **Trả lời.** LiDAR: `z = (x, y, z)` mét (3×1), tâm hộp 3D trong hệ cảm biến;
   `h(x) = R·p + t` **tuyến tính** theo trạng thái nên `H` là ma trận hằng 3×6 (khối xoay,
   cột vận tốc bằng 0); `R = diag(0.1², 0.1², 0.1²)` m². Camera: `z = (u, v)` pixel (2×1);
   `h(x)` là phép chiếu pinhole **phi tuyến** `u = c_i − f_i·y_s/x_s`, `v = c_j − f_j·z_s/x_s`
   (`camera_measurement_prediction` trong `camera_fusion.py`), nên EKF dùng Jacobian `H` 2×6
   tính lại tại từng `x` (platform `Sensor.get_H`); `R = diag(5², 5²)` px²
   (`build_camera_measurement`). Camera không đo trực tiếp độ sâu — nó chỉ ràng buộc hướng
   nhìn. `ekf_update` trong `kalman.py` dùng chung `K = P Hᵀ S⁻¹` cho cả hai, chỉ khác
   `z`, `h(x)`, `H`, `R` do `meas.sensor` cung cấp.

2. Vì sao cần gating Mahalanobis trước khi gán?

   **Trả lời.** `d² = γᵀ S⁻¹ γ` (`mahalanobis_distance`) đo độ lệch đã chuẩn hoá theo bất định
   `S = H P Hᵀ + R`: cùng 1 m lệch là nhỏ với track mới (P lớn) nhưng lớn với track đã hội tụ.
   Nếu cặp đúng, `d²` theo phân phối χ² với bậc tự do = số chiều đo (3 cho LiDAR, 2 cho camera),
   nên `chi2_gate` loại cặp có `d² ≥ chi2.ppf(0.995, dim_meas)`. Gating ngăn update EKF bằng đo
   của **xe khác** hoặc false positive (track nhảy, đổi ID, sinh ghost) và giữ đo chưa khớp để
   tạo track mới. Trong `association_cost_matrix`, cặp ngoài FOV được đặt `inf` **trước** khi
   tính `d²`, vì với camera điểm phía sau / độ sâu ≈ 0 thì phép chiếu không xác định.

3. Pipeline là track-then-fuse hay fuse-then-track? Chỉ ra trên log `fusion-run-lab`.

   **Trả lời.** **Track-then-fuse.** Trong `platform/fusion_lab/scripts/run_lab.py`, mỗi frame:
   `KF.predict(track)` một lần cho mọi track → `assoc.associate_and_update(manager, observations, KF, lidar_sensor)`
   (AssocL) → nếu có camera thì `associate_and_update(..., camera_sensor)` (AssocC) → ghi record.
   Không có bước ghép đo LiDAR + camera thành một đo chung trước khi tracking. Trên log:
   `grade_run_lidar.log` và `grade_run_fused.log` có **cùng** `det_tp/det_fp/det_fn`, `confirmed`,
   `matches` ở mọi frame (cùng các track do LiDAR tạo), chỉ `sum_sq_err` khác (ví dụ frame 60:
   0.284 vs 0.335) — tức camera chỉ tinh chỉnh trạng thái của track đã có.

4. Nếu camera lệch calibration, triệu chứng gì trên innovation/residual?

   **Trả lời.** Extrinsic sai làm `h(x)` chiếu track vào sai chỗ trên ảnh một cách **hệ thống**:
   innovation `γ = z − h(x)` không còn trung bình 0 mà có **bias** cùng dấu (ví dụ lệch yaw →
   `u` lệch cùng chiều cho mọi xe), và `d²` trung bình vượt số bậc tự do (2). Lệch nhỏ: cặp vẫn
   qua cổng χ², update camera kéo track về sai chỗ → RMSE fused tăng so với LiDAR, lượt LiDAR
   frame sau kéo lại (track "giật" qua lại). Lệch lớn: `d²` vượt `chi2.ppf(0.995, 2) ≈ 10.6`
   → gating loại hết đo camera, fused tiến về kết quả chỉ-LiDAR. (Em chưa làm thí nghiệm bonus
   đo định lượng phần này.)

5. Vì sao `associate_and_update(..., sensor)` cần sensor tường minh ở frame rỗng?
   Giải thích vì sao lidar quyết định score/init/delete còn camera chỉ EKF update.

   **Trả lời.** Khi `meas_list` rỗng thì không suy ra được loại cảm biến từ `meas.sensor`, nhưng
   lượt LiDAR rỗng vẫn phải gọi `manager.manage_tracks(unassigned_tracks, [], sensor)` để trừ
   score các track trong FOV bị miss và xoá track hết score (test
   `test_empty_lidar_frame_scores_then_deletes_exhausted_track`). Bỏ qua frame rỗng thì track
   ma sống mãi; đoán sai sensor thì một lượt camera rỗng có thể bị xử lý như miss LiDAR.
   LiDAR quyết định tồn tại vì nó là detector thực, phủ 360°, cho vị trí 3D đủ để khởi tạo
   `x`, `P` (`init_track_state_from_meas`). Camera trong lab chỉ có FOV phía trước, 2D, không có
   độ sâu (không khởi tạo được track 3D) và lấy từ nhãn GT có nhiễu; nếu camera cộng/trừ score
   thì xe ngoài FOV camera bị trừ oan và mỗi frame track được tính hit hai lần. Vì vậy
   `TrackManager.handle_updated_track` chỉ gọi `update_track_score` khi `sensor.name == "lidar"`,
   còn `manage_tracks` return ngay ở lượt camera.

6. Nêu điều kiện xác nhận, giữ confirmed sau miss, và điều kiện xóa track.

   **Trả lời.** (`track_management.py`, `window = 6`.) Khởi tạo: `score = 1/6`, `state = initialized`.
   Mỗi lượt LiDAR: hit → `score = min(score + 1/6, 1)`; miss **trong FOV LiDAR** → `score − 1/6`.
   **Xác nhận** khi `score > confirmed_threshold = 0.8` (lớn hơn chặt; `score = 0.8` vẫn
   `tentative`), tức 5 frame LiDAR liên tiếp có đo (lần khởi tạo + 4 hit → 5/6); track chưa confirmed mà có hit → `tentative`.
   **Giữ confirmed**: đã `confirmed` thì không hạ trạng thái, một miss chỉ giảm 1.0 → 0.833.
   **Xoá** (OR): `P[0,0] > max_P` hoặc `P[1,1] > max_P` (= 3² = 9 m²) bất kể score; track
   confirmed có `score < delete_threshold = 0.6`; track chưa confirmed có `score ≤ 0`. Lượt camera
   không bao giờ xoá. So sánh dùng sai số 1e-9 để `0.8 − 1/6 + 1/6` không bị làm tròn thành > 0.8.

## Bonus (không bắt buộc)

Liệt kê phần bonus đã làm, file bằng chứng trong `student/bonus/` và kết quả chính
(xem [RUBRIC.md](../RUBRIC.md) mục 2). Không làm thì ghi "Không".

- Không.

## Khai báo sử dụng AI (bắt buộc)

Ghi rõ, kể cả khi không dùng ("Không dùng AI"). Xem [RULES.md](../RULES.md) mục 2.

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): Claude (Claude Code, model Claude Opus)
- Dùng cho phần nào (hàm, câu hỏi, debug): đọc đề và lập kế hoạch; viết code Part E (`kalman.py`), G (`camera_fusion.py`), F (`association.py`), H (`track_management.py`); debug lỗi `float()` trên ma trận 1×1 với NumPy mới và lỗi làm tròn ở ngưỡng xác nhận; chọn segment (thử segment #2 thấy không có xe); chạy `fusion-run-lab`; soạn nháp báo cáo và 6 câu trả lời E–H từ code và log.
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): chạy `pytest student/tests -q` sau từng Part (cuối cùng 128 passed, không xfailed); đối chiếu công thức với `docs/HUONG_DAN_KY_THUAT.md` và các gợi ý `# vi: TODO`; chạy Waymo `--fusion compare --seed 0` frame 0–198; số liệu trong báo cáo lấy trực tiếp từ `student/artifacts/metrics.json` và `grade_run*.log`, không sửa tay; chạy `python tools/check_submission.py`.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed`
- [x] Part A–D: không bắt buộc sửa (hoặc ghi chú nếu bạn đã sửa)
- [x] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`
- [x] Đã commit `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` (không sửa tay)
- [x] Đã điền đủ file này, gồm khai báo AI
- [x] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key
- [x] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP`
- [ ] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md))
