# OpenPilot Hybrid 交接 README

這份 README目標是讓你不用先讀完整程式碼，也能知道資料放哪裡、要跑哪個檔案、每個輸出會去哪裡，以及 OP-DeepDive / nuScenes-compatible 與 Comma2k19 兩套資料集要怎麼從「建立環境 → 訓練 → 推論 → 生成影片 → 畫 loss」完整跑起來。

---

## 0. 先記住一句話

最新版請以 **兼容(hybrid) 版本**為主。舊版只支援 OP-DeepDive / nuScenes-compatible，hybrid 版同時支援 `opdeepdive` 與 `comma2k19`。程式碼位置預設如下：

```bat
E:\3dvmlab\projects\openpilot
```

資料集預設如下：

```bat
E:\opdeepdiveDataset\data1
E:\comma2k19
E:\comma2k19_cache\frames
```

如果外接硬碟在別台電腦變成 `D:`、`F:` 或其他磁碟，請把指令中的 `E:/...` 改成實際磁碟代號。指令中建議用 `/`，例如 `E:/comma2k19`，不要混用太多反斜線。

---

## 1. 舊版與 hybrid 版差異

<table>
<thead>
<tr>
<th>項目</th>
<th>舊版非 hybrid</th>
<th>新版 hybrid</th>
</tr>
</thead>
<tbody>
<tr>
<td>支援資料集</td>
<td>只支援 OP-DeepDive / nuScenes-compatible</td>
<td>同時支援 OP-DeepDive / nuScenes-compatible 與 Comma2k19</td>
</tr>
<tr>
<td>資料讀取</td>
<td><code>dataset_tools.py</code></td>
<td><code>dataset_tools_hybrid.py</code></td>
</tr>
<tr>
<td>訓練入口</td>
<td><code>main.py</code>、<code>train.sh</code></td>
<td><code>main_hybrid.py</code>、<code>train_hybrid.sh</code></td>
</tr>
<tr>
<td>推論入口</td>
<td><code>bulk_inference.py</code></td>
<td><code>bulk_inference_hybrid.py</code></td>
</tr>
<tr>
<td>影片生成入口</td>
<td><code>demo.py</code></td>
<td><code>demo_hybrid.py</code></td>
</tr>
<tr>
<td>模型</td>
<td><code>model.py</code> 使用 EfficientNet</td>
<td><code>model.py</code> 仍使用 EfficientNet，但注意 <code>from_pretrained</code> 已改成 <code>from_name</code>，避免訓練時自動連 GitHub 下載權重</td>
</tr>
<tr>
<td>主要差異</td>
<td>資料來源固定，流程較單純</td>
<td>用 <code>--dataset</code> 切換資料集；Comma2k19 會從 <code>video.hevc</code> 解 frame，並用 cache 加速</td>
</tr>
</tbody>
</table>

---

## 2. 資料集整理

<table>
<thead>
<tr>
<th>資料集</th>
<th>正式名稱</th>
<th>本專案使用名稱</th>
<th>公開資料規模</th>
<th>本機預設位置</th>
<th>注意事項</th>
</tr>
</thead>
<tbody>
<tr>
<td>nuScenes / OP-DeepDive</td>
<td>nuTonomy scenes，也就是 nuScenes；本專案使用 OP-DeepDive 整理後的 nuScenes-compatible 前視角資料</td>
<td><code>opdeepdive</code></td>
<td>nuScenes 公開資料為 1000 個 scene，每個約 20 秒，總時長約 5.5 小時。完整資料容量依下載內容不同而不同，常見完整壓縮版約數百 GB；本專案實際大小請以本機資料夾為準。</td>
<td><code>E:\opdeepdiveDataset\data1</code></td>
<td>訓練時會讀 <code>sweeps/CAM_FRONT/split/&lt;case_id&gt;/&lt;case_id&gt;_culane.json</code> 與對應影像。</td>
</tr>
<tr>
<td>Comma2k19</td>
<td>comma.ai comma2k19 driving dataset</td>
<td><code>comma2k19</code></td>
<td>公開資料約 2019 段，每段約 1 分鐘，總時長超過 33 小時。官方下載約 100GB，但本機解壓、cache、實驗輸出會額外佔很多空間。</td>
<td><code>E:\comma2k19</code></td>
<td>訓練時會讀 <code>Chunk_*/&lt;drive&gt;/&lt;segment&gt;/video.hevc</code> 與 <code>global_pose</code>。強烈建議把 cache 放 SSD。</td>
</tr>
<tr>
<td>Comma2k19 frame cache</td>
<td>由本專案從 Comma2k19 的 HEVC 影片解出的 frame cache</td>
<td><code>--comma_cache_root</code></td>
<td>不是原始資料，會依訓練跑到的 segment / frame 逐步增加，可能達數十 GB 到數百 GB 以上。</td>
<td><code>E:\comma2k19_cache\frames</code></td>
<td>可以刪除重建；若 F 槽是 HDD，建議改放到內建 SSD，例如 <code>C:\comma2k19_cache\frames</code>。</td>
</tr>
</tbody>
</table>

查本機資料夾實際容量可用：

```bat
powershell -Command "(Get-ChildItem 'E:\comma2k19' -Recurse -File | Measure-Object Length -Sum).Sum / 1GB"
powershell -Command "(Get-ChildItem 'E:\opdeepdiveDataset' -Recurse -File | Measure-Object Length -Sum).Sum / 1GB"
powershell -Command "(Get-ChildItem 'E:\comma2k19_cache\frames' -Recurse -File | Measure-Object Length -Sum).Sum / 1GB"
```

---

## 3. 程式碼放哪裡、各檔案做什麼

<table>
<thead>
<tr>
<th>檔案</th>
<th>預設位置</th>
<th>功能</th>
<th>新手注意</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>ActivateEnv_auto_cuda_install_v4.bat</code></td>
<td><code>E:\3dvmlab\projects\openpilot</code></td>
<td>啟動 <code>openpilotEnv</code>，必要時建立 conda 環境，檢查 CUDA，安裝必要套件。</td>
<td>如果 NumPy 被升到 2.x 造成 matplotlib 錯誤，執行 <code>repair_numpy_matplotlib_openpilotEnv.bat</code>。</td>
</tr>
<tr>
<td><code>repair_numpy_matplotlib_openpilotEnv.bat</code></td>
<td><code>E:\3dvmlab\projects\openpilot</code></td>
<td>把 <code>numpy</code>、<code>matplotlib</code>、<code>opencv-python</code>、<code>pillow</code> 固定回相容版本。</td>
<td>遇到 <code>_ARRAY_API not found</code> 或 <code>numpy.core.multiarray failed to import</code> 時先跑它。</td>
</tr>
<tr>
<td><code>dataset_tools_hybrid.py</code></td>
<td><code>E:\3dvmlab\projects\openpilot</code></td>
<td>資料集讀取。<code>NuSceneDataset</code> 負責 OP-DeepDive；<code>Comma2k19Dataset</code> 負責 Comma2k19。</td>
<td>Comma2k19 會解 <code>video.hevc</code> 並把 jpg frame 寫到 cache。</td>
</tr>
<tr>
<td><code>main_hybrid.py</code></td>
<td><code>E:\3dvmlab\projects\openpilot</code></td>
<td>訓練主程式。用 <code>--dataset opdeepdive</code> 或 <code>--dataset comma2k19</code> 切換資料集。</td>
<td>權重檔 <code>epoch_*.pth</code> 存在 TensorBoard run 目錄，也就是 <code>--log_dir</code> 指定的位置。</td>
</tr>
<tr>
<td><code>train_hybrid.sh</code></td>
<td><code>E:\3dvmlab\projects\openpilot</code></td>
<td>訓練啟動腳本，設定單卡 DDP 需要的 <code>PORT</code>、<code>SLURM_PROCID</code>、<code>SLURM_NTASKS</code>。</td>
<td>Windows CMD 仍然透過 Git 的 <code>sh</code> 執行。</td>
</tr>
<tr>
<td><code>bulk_inference_hybrid.py</code></td>
<td><code>E:\3dvmlab\projects\openpilot</code></td>
<td>批次推論。OP-DeepDive 會輸出各 scene 的 <code>predicted_routes.json</code>；Comma2k19 會輸出各 segment 的 <code>predicted_routes.json</code> 與 <code>metadata.json</code>。</td>
<td>若檔名維持 <code>*_hybrid.py</code>，請確認 import 指向 hybrid 檔案，詳見第 11 節。</td>
</tr>
<tr>
<td><code>demo_hybrid.py</code></td>
<td><code>E:\3dvmlab\projects\openpilot</code></td>
<td>把推論結果畫成影片。OP-DeepDive 有 camera view + BEV；Comma2k19 主要是 video frame + BEV route。</td>
<td>若檔名維持 <code>*_hybrid.py</code>，請確認 import 與 <code>case_id</code> 變數沒有被改壞。</td>
</tr>
<tr>
<td><code>model.py</code></td>
<td><code>E:\3dvmlab\projects\openpilot</code></td>
<td>模型架構。主要使用 <code>SequencePlanningNetwork</code>。</td>
<td>請確認所有訓練用的 EfficientNet 都是 <code>EfficientNet.from_name(...)</code>，不要用 <code>from_pretrained(...)</code>。</td>
</tr>
<tr>
<td><code>draw_loss.py</code></td>
<td><code>E:\3dvmlab\projects\openpilot</code></td>
<td>讀取 loss log 並輸出 loss 曲線圖。</td>
<td>新版 hybrid 的 log 在 <code>data/openpilot/loss_log_28.txt</code>，舊版硬編碼路徑要改掉。</td>
</tr>
</tbody>
</table>

---

## 4. 第一次建立環境

先切到專案資料夾：

```bat
E:
cd \3dvmlab\projects\openpilot
```

啟動環境：

```bat
ActivateEnv_auto_cuda_install_v4.bat
```

如果出現 NumPy / matplotlib 相容性錯誤，執行：

```bat
repair_numpy_matplotlib_openpilotEnv.bat
```

確認套件與 CUDA：

```bat
python -c "import numpy, matplotlib, cv2, torch; print('numpy', numpy.__version__); print('matplotlib', matplotlib.__version__); print('cv2', cv2.__version__); print('torch', torch.__version__, torch.cuda.is_available())"
```

建議看到類似：

```text
numpy 1.26.4
matplotlib 3.5.3
cv2 4.10.0
torch 2.7.1+cu118 True
```

如果 `torch.cuda.is_available()` 是 `False`，先用 `nvidia-smi` 確認 NVIDIA GPU 是否可見。

---

## 5. OP-DeepDive / nuScenes-compatible 完整流程

### 5.1 訓練

建議明確指定 `--log_dir`，之後權重、TensorBoard log 才不會找不到：

```bash
sh train_hybrid.sh --dataset opdeepdive ^
  --opdeepdive_root E:/opdeepdiveDataset/data1 ^
  --log_dir E:/3dvmlab/projects/openpilot/runs/opdeepdive_full ^
  --tqdm True
```

如果你的 shell 不吃 `^` 換行，請改成一行：

```bash
sh train_hybrid.sh --dataset opdeepdive --opdeepdive_root E:/opdeepdiveDataset/data1 --log_dir E:/3dvmlab/projects/openpilot/runs/opdeepdive_full --tqdm True
```

### 5.2 推論

假設要用第 999 個 epoch 的權重：

```bash
python bulk_inference_hybrid.py --dataset opdeepdive ^
  --ckpt E:/3dvmlab/projects/openpilot/runs/opdeepdive_full/epoch_999.pth ^
  --opdeepdive_root E:/opdeepdiveDataset/data1 ^
  --op_output E:/3dvmlab/projects/openpilot/val/openpilot
```

只推論單一場景，例如 0272：

```bash
python bulk_inference_hybrid.py --dataset opdeepdive --case_id 0272 --ckpt E:/3dvmlab/projects/openpilot/runs/opdeepdive_full/epoch_999.pth --opdeepdive_root E:/opdeepdiveDataset/data1 --op_output E:/3dvmlab/projects/openpilot/val/openpilot
```

### 5.3 生成影片

```bash
python demo_hybrid.py 0272 --dataset opdeepdive ^
  --img_root E:/opdeepdiveDataset/data1 ^
  --op_pred_root E:/3dvmlab/projects/openpilot/val/openpilot ^
  --output E:/3dvmlab/projects/openpilot/val/demo
```

一行版：

```bash
python demo_hybrid.py 0272 --dataset opdeepdive --img_root E:/opdeepdiveDataset/data1 --op_pred_root E:/3dvmlab/projects/openpilot/val/openpilot --output E:/3dvmlab/projects/openpilot/val/demo
```

輸出會在：

```text
E:\3dvmlab\projects\openpilot\val\demo\0272\0272.mp4
```

---

## 6. Comma2k19 完整流程

### 6.1 先跑小測試

Comma2k19 正式資料量很大，不要一開始就全跑。先確認 dataloader、GPU、cache 都正常：

```bash
sh train_hybrid.sh --dataset comma2k19 ^
  --comma_root E:/comma2k19 ^
  --comma_cache_root E:/comma2k19_cache/frames ^
  --comma_max_segments 3 ^
  --comma_index_every_n 100 ^
  --epochs 1 ^
  --tqdm True ^
  --log_per_n_step 1 ^
  --val_per_n_epoch 999 ^
  --log_dir E:/3dvmlab/projects/openpilot/runs/comma2k19_smoke_test
```

一行版：

```bash
sh train_hybrid.sh --dataset comma2k19 --comma_root E:/comma2k19 --comma_cache_root E:/comma2k19_cache/frames --comma_max_segments 3 --comma_index_every_n 100 --epochs 1 --tqdm True --log_per_n_step 1 --val_per_n_epoch 999 --log_dir E:/3dvmlab/projects/openpilot/runs/comma2k19_smoke_test
```

### 6.2 正式訓練

```bash
sh train_hybrid.sh --dataset comma2k19 ^
  --comma_root E:/comma2k19 ^
  --comma_cache_root E:/comma2k19_cache/frames ^
  --log_dir E:/3dvmlab/projects/openpilot/runs/comma2k19_full ^
  --comma_index_every_n 30 ^
  --epochs 20 ^
  --val_per_n_epoch 5 ^
  --batch_size 6 ^
  --n_workers 4 ^
  --tqdm True
```

一行版：

```bash
sh train_hybrid.sh --dataset comma2k19 --comma_root E:/comma2k19 --comma_cache_root E:/comma2k19_cache/frames --log_dir E:/3dvmlab/projects/openpilot/runs/comma2k19_full --comma_index_every_n 30 --epochs 20 --val_per_n_epoch 5 --batch_size 6 --n_workers 4 --tqdm True
```

### 6.3 推論

先推論少量 segment 測試：

```bash
python bulk_inference_hybrid.py --dataset comma2k19 ^
  --ckpt E:/3dvmlab/projects/openpilot/runs/comma2k19_full/epoch_999.pth ^
  --comma_root E:/comma2k19 ^
  --comma_cache_root E:/comma2k19_cache/frames ^
  --comma_output E:/3dvmlab/projects/openpilot/val/comma2k19 ^
  --max_segments 1
```

一行版：

```bash
python bulk_inference_hybrid.py --dataset comma2k19 --ckpt E:/3dvmlab/projects/openpilot/runs/comma2k19_full/epoch_999.pth --comma_root E:/comma2k19 --comma_cache_root E:/comma2k19_cache/frames --comma_output E:/3dvmlab/projects/openpilot/val/comma2k19 --max_segments 1
```

輸出資料夾會長這樣：

```text
E:\3dvmlab\projects\openpilot\val\comma2k19\<segment_id>\predicted_routes.json
E:\3dvmlab\projects\openpilot\val\comma2k19\<segment_id>\metadata.json
```

`<segment_id>` 是由 Chunk、drive folder、segment number 組合成的安全檔名，例如：

```text
Chunk_4__99c94dc769b5d96e_2018-07-02--19-08-27__10
```

### 6.4 生成影片

```bash
python demo_hybrid.py <segment_id> --dataset comma2k19 ^
  --comma_pred_root E:/3dvmlab/projects/openpilot/val/comma2k19 ^
  --cache_root E:/comma2k19_cache/frames ^
  --output E:/3dvmlab/projects/openpilot/val/demo_comma2k19
```

一行版：
```bash
python demo_hybrid.py <segment_id> --dataset comma2k19 --comma_pred_root E:/3dvmlab/projects/openpilot/val/comma2k19 --cache_root E:/comma2k19_cache/frames --output E:/3dvmlab/projects/openpilot/val/demo_comma2k19
```

輸出會在：

```text
E:\3dvmlab\projects\openpilot\val\demo_comma2k19\<segment_id>\<segment_id>.mp4
```

---

## 7. 權重、log、loss curve、推論輸出放哪裡

<table>
<thead>
<tr>
<th>項目</th>
<th>OP-DeepDive / nuScenes-compatible</th>
<th>Comma2k19</th>
<th>由哪個程式產生</th>
</tr>
</thead>
<tbody>
<tr>
<td>每個 epoch 的權重檔</td>
<td><code>E:\3dvmlab\projects\openpilot\runs\opdeepdive_full\epoch_*.pth</code></td>
<td><code>E:\3dvmlab\projects\openpilot\runs\comma2k19_full\epoch_*.pth</code></td>
<td><code>main_hybrid.py</code>，實際位置是 <code>--log_dir</code>。</td>
</tr>
<tr>
<td>TensorBoard event log</td>
<td><code>E:\3dvmlab\projects\openpilot\runs\opdeepdive_full\events.out.tfevents*</code></td>
<td><code>E:\3dvmlab\projects\openpilot\runs\comma2k19_full\events.out.tfevents*</code></td>
<td><code>SummaryWriter</code>。</td>
</tr>
<tr>
<td>文字版 loss log</td>
<td colspan="2"><code>E:\3dvmlab\projects\openpilot\data\openpilot\loss_log_28.txt</code></td>
<td><code>main_hybrid.py</code>。注意：兩種資料集會寫進同一個檔案，但每行會有 <code>Dataset opdeepdive</code> 或 <code>Dataset comma2k19</code>。</td>
</tr>
<tr>
<td>loss 曲線圖</td>
<td><code>E:\3dvmlab\projects\openpilot\runs\opdeepdive_full\loss_curve.png</code></td>
<td><code>E:\3dvmlab\projects\openpilot\runs\comma2k19_full\loss_curve.png</code></td>
<td><code>draw_loss.py</code>，需指定 <code>--log</code> 與 <code>--output</code>。</td>
</tr>
<tr>
<td>OP-DeepDive 推論 JSON</td>
<td><code>E:\3dvmlab\projects\openpilot\val\openpilot\&lt;case_id&gt;\predicted_routes.json</code></td>
<td>不適用</td>
<td><code>bulk_inference_hybrid.py</code></td>
</tr>
<tr>
<td>Comma2k19 推論 JSON</td>
<td>不適用</td>
<td><code>E:\3dvmlab\projects\openpilot\val\comma2k19\&lt;segment_id&gt;\predicted_routes.json</code><br><code>E:\3dvmlab\projects\openpilot\val\comma2k19\&lt;segment_id&gt;\metadata.json</code></td>
<td><code>bulk_inference_hybrid.py</code></td>
</tr>
<tr>
<td>OP-DeepDive demo 影片</td>
<td><code>E:\3dvmlab\projects\openpilot\val\demo\&lt;case_id&gt;\&lt;case_id&gt;.mp4</code></td>
<td>不適用</td>
<td><code>demo_hybrid.py</code></td>
</tr>
<tr>
<td>Comma2k19 demo 影片</td>
<td>不適用</td>
<td><code>E:\3dvmlab\projects\openpilot\val\demo_comma2k19\&lt;segment_id&gt;\&lt;segment_id&gt;.mp4</code></td>
<td><code>demo_hybrid.py</code></td>
</tr>
</tbody>
</table>

---

## 8. 畫 loss curve

新版 loss log 位置是：

```text
E:\3dvmlab\projects\openpilot\data\openpilot\loss_log_28.txt
```

建議把修正版 `draw_loss.py` 放在：

```text
E:\3dvmlab\projects\openpilot\draw_loss.py
```

畫 OP-DeepDive：

```bat
python draw_loss.py --dataset opdeepdive --log E:/3dvmlab/projects/openpilot/data/openpilot/loss_log_28.txt --output E:/3dvmlab/projects/openpilot/runs/opdeepdive_full/loss_curve.png
```

畫 Comma2k19：

```bat
python draw_loss.py --dataset comma2k19 --log E:/3dvmlab/projects/openpilot/data/openpilot/loss_log_28.txt --output E:/3dvmlab/projects/openpilot/runs/comma2k19_full/loss_curve.png
```

如果要看 TensorBoard：

```bat
tensorboard --logdir E:\3dvmlab\projects\openpilot\runs
```

瀏覽器打開：

```text
http://localhost:6006
```

---

## 9. 常用參數速查

<table>
<thead>
<tr>
<th>參數</th>
<th>預設值</th>
<th>說明</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>--batch_size</code></td>
<td><code>6</code></td>
<td>訓練 batch size。4GB GPU 通常要改成 1。</td>
</tr>
<tr>
<td><code>--n_workers</code></td>
<td><code>4</code></td>
<td>Dataloader worker 數。Comma2k19 + HDD 若很慢或卡住，可改成 0。</td>
</tr>
<tr>
<td><code>--epochs</code></td>
<td><code>1000</code></td>
<td>總訓練 epoch 數。</td>
</tr>
<tr>
<td><code>--lr</code></td>
<td><code>5e-4</code></td>
<td>Learning rate。</td>
</tr>
<tr>
<td><code>--optimizer</code></td>
<td><code>sgd</code></td>
<td>可選 <code>sgd</code>、<code>adam</code>、<code>adamw</code>。</td>
</tr>
<tr>
<td><code>--M</code></td>
<td><code>5</code></td>
<td>多模態 trajectory 數量。</td>
</tr>
<tr>
<td><code>--num_pts</code></td>
<td><code>20</code></td>
<td>每條未來軌跡取 20 個點。</td>
</tr>
<tr>
<td><code>--comma_future_step_m</code></td>
<td><code>4.0</code></td>
<td>Comma2k19 future pose 每點間隔 4 公尺。</td>
</tr>
<tr>
<td><code>--comma_max_segments</code></td>
<td><code>0</code></td>
<td>0 代表用全部 segment。測試時建議設 3、20、50 逐步放大。</td>
</tr>
<tr>
<td><code>--comma_index_every_n</code></td>
<td><code>1</code></td>
<td>每幾個 frame 建一筆 sample。正式最密是 1；測試建議 50 或 100。</td>
</tr>
<tr>
<td><code>--log_dir</code></td>
<td>空字串</td>
<td>空白時自動寫到 <code>runs</code> 的時間戳資料夾。正式實驗建議手動指定。</td>
</tr>
</tbody>
</table>

---

## 10. 如果你折騰到一半遇到問題

<table>
<thead>
<tr>
<th>狀況</th>
<th>原因</th>
<th>處理方式</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>ModuleNotFoundError</code></td>
<td>新電腦環境缺套件</td>
<td>先執行 <code>ActivateEnv_auto_cuda_install_v4.bat</code>。</td>
</tr>
<tr>
<td><code>_ARRAY_API not found</code> 或 <code>numpy.core.multiarray failed to import</code></td>
<td>NumPy 2.x 和舊版 matplotlib compiled module 不相容</td>
<td>執行 <code>repair_numpy_matplotlib_openpilotEnv.bat</code>。</td>
</tr>
<tr>
<td>EfficientNet 一直嘗試下載權重</td>
<td><code>model.py</code> 還有 <code>from_pretrained</code></td>
<td>改成 <code>EfficientNet.from_name('efficientnet-b2', in_channels=6)</code>。</td>
</tr>
<tr>
<td>正式 Comma2k19 訓練卡很久，GPU 0%</td>
<td>HDD + HEVC 隨機 seek + cache 寫入太慢</td>
<td>先跑小測試；cache 放 SSD；不要一開始就跑完整資料。</td>
</tr>
<tr>
<td>CUDA 明明有 GPU，但 <code>CUDA_VISIBLE_DEVICES=1</code> 後看不到 GPU</td>
<td>Windows GPU 編號不等於 CUDA 編號；通常 NVIDIA 在 CUDA 裡是 <code>0</code></td>
<td>用 <code>nvidia-smi -L</code> 看 CUDA 編號，通常設 <code>CUDA_VISIBLE_DEVICES=0</code> 或不設。</td>
</tr>
<tr>
<td>找不到權重檔</td>
<td>不知道 <code>SummaryWriter</code> 的 <code>writer.log_dir</code> 是哪裡</td>
<td>正式訓練一律加 <code>--log_dir E:/3dvmlab/projects/openpilot/runs/實驗名稱</code>。</td>
</tr>
<tr>
<td><code>loss_log_28.txt</code> 沒有新資料</td>
<td>文字 log 要 epoch 結束才寫入</td>
<td>跑到一半先看 TensorBoard event；或開 <code>--tqdm True</code> 看進度。</td>
</tr>
</tbody>
</table>

---

## 11. hybrid 檔名與 import 檢查

如果你是把 hybrid 版本直接覆蓋成原本檔名：

```text
dataset_tools_hybrid.py → dataset_tools.py
main_hybrid.py          → main.py
bulk_inference_hybrid.py→ bulk_inference.py
demo_hybrid.py          → demo.py
```

那舊 import 通常不會有問題。

但如果你保留 `*_hybrid.py` 檔名，請檢查：

```python
# bulk_inference_hybrid.py
from main_hybrid import SequenceBaselineV1
from dataset_tools_hybrid import (
    NORMALIZE_MEAN,
    NORMALIZE_STD,
    discover_comma2k19_segments,
    split_comma2k19_segments,
    read_comma2k19_frame,
    safe_segment_id,
    load_npy_no_ext,
)
```

以及：

```python
# demo_hybrid.py
from dataset_tools_hybrid import read_comma2k19_frame
```

也請確認 `demo_hybrid.py` 裡的 function 參數是：

```python
def run_opdeepdive_demo(case_id: str, ...)
```

不要變成 `case_iF` 這種路徑取代造成的錯字。

---

## 12. 最短完整交接流程

### OP-DeepDive / nuScenes-compatible

```bat
cd /d E:\3dvmlab\projects\openpilot
ActivateEnv_auto_cuda_install_v4.bat
sh train_hybrid.sh --dataset opdeepdive --opdeepdive_root E:/opdeepdiveDataset/data1 --log_dir E:/3dvmlab/projects/openpilot/runs/opdeepdive_full --tqdm True
python bulk_inference_hybrid.py --dataset opdeepdive --case_id 0272 --ckpt E:/3dvmlab/projects/openpilot/runs/opdeepdive_full/epoch_999.pth --opdeepdive_root E:/opdeepdiveDataset/data1 --op_output E:/3dvmlab/projects/openpilot/val/openpilot
python demo_hybrid.py 0272 --dataset opdeepdive --img_root E:/opdeepdiveDataset/data1 --op_pred_root E:/3dvmlab/projects/openpilot/val/openpilot --output E:/3dvmlab/projects/openpilot/val/demo
python draw_loss.py --dataset opdeepdive --log E:/3dvmlab/projects/openpilot/data/openpilot/loss_log_28.txt --output E:/3dvmlab/projects/openpilot/runs/opdeepdive_full/loss_curve.png
```

### Comma2k19

```bat
cd /d E:\3dvmlab\projects\openpilot
ActivateEnv_auto_cuda_install_v4.bat
sh train_hybrid.sh --dataset comma2k19 --comma_root E:/comma2k19 --comma_cache_root E:/comma2k19_cache/frames --log_dir E:/3dvmlab/projects/openpilot/runs/comma2k19_full --tqdm True
python bulk_inference_hybrid.py --dataset comma2k19 --ckpt E:/3dvmlab/projects/openpilot/runs/comma2k19_full/epoch_999.pth --comma_root E:/comma2k19 --comma_cache_root E:/comma2k19_cache/frames --comma_output E:/3dvmlab/projects/openpilot/val/comma2k19 --max_segments 1
python demo_hybrid.py <segment_id> --dataset comma2k19 --comma_pred_root E:/3dvmlab/projects/openpilot/val/comma2k19 --cache_root E:/comma2k19_cache/frames --output E:/3dvmlab/projects/openpilot/val/demo_comma2k19
python draw_loss.py --dataset comma2k19 --log E:/3dvmlab/projects/openpilot/data/openpilot/loss_log_28.txt --output E:/3dvmlab/projects/openpilot/runs/comma2k19_full/loss_curve.png
```

---

## 13. 參考資料與本 README 的依據

這份 README 主要依據：

- `本機版教學.txt`
- `dataset_tools_hybrid.py`
- `main_hybrid.py`
- `bulk_inference_hybrid.py`
- `demo_hybrid.py`
- `draw_loss.py`
- 目前專案討論過的 F 槽 hybrid 路徑設定

公開資料集規格可參考：

- nuScenes official site / paper：1000 scenes，每段 20 秒，約 5.5 小時。
- comma2k19 official GitHub / paper：2019 segments，每段約 1 分鐘，超過 33 小時，官方下載約 100GB。

