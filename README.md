# YOLOv8 Detection on Jetson Nano

ระบบตรวจจับด้วย YOLOv8n เทรนบน PC แล้ว Deploy บน Jetson Nano 4GB

---

## 📋 สิ่งที่ต้องเตรียม

- PC (Windows) สำหรับเทรนโมเดล + GPU (CUDA)
- Jetson Nano 4GB (JetPack 4.6.1)
- กล้อง USB
- Dataset ขนม

---

## 📊 ผลการเทรน

| Class | Images | mAP@50 | mAP@50-95 |
|---|---|---|---|
| **All** | 57 | **0.995** | **0.819** |
| Candy | 2 | 0.995 | 0.895 |
| Chocolate | 20 | 0.995 | 0.856 |
| Gummy | 17 | 0.995 | 0.749 |
| Halls | 18 | 0.995 | 0.775 |

---

## 💻 ฝั่ง PC (Windows)

### Environment
- Python 3.12
- PyTorch (CUDA 12.6)
- Ultralytics YOLOv8

### 1. สร้าง Workspace และ Virtual Environment
–เปิด cmd เเล้วคลิ๊กขวาเลือก Run as Administrator
```bash
mkdir yolov8
cd yolov8
python -m venv venv
venv\Scripts\activate.bat
```

### 2. ติดตั้ง Package
```bash
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu126
pip install ultralytics
```

### 3. เปิด VS Code
```bash
code .
```

### 4. โครงสร้าง Dataset
```
dataset/
├── train/
│   ├── images/
│   └── labels/
├── valid/
│   ├── images/
│   └── labels/
└── data.yaml
```

### 5. ไฟล์ data.yaml
```yaml
path: dataset
train: train/images
val: valid/images

nc: 4
names: ["Candy", "Chocolate", "Gummy", "Halls"]
```

### 6. ไฟล์ train.py
```python
from ultralytics import YOLO

model = YOLO("yolov8n.pt")

model.train(
    data="dataset/data.yaml",
    epochs=100,
    imgsz=640,
    batch=16,
    device=0
)
```

### 7. รัน Train
```bash
python train.py
```

โมเดลจะถูกบันทึกที่:
```
runs/detect/train/weights/best.pt
```

---

## 🤖 ฝั่ง Jetson Nano

### Environment
- JetPack: 4.6.1 (R32.6.1)
- Ubuntu: 20.04
- Python: 3.8.10
- CUDA: 10.2
- PyTorch: 1.11.0
- torchvision: 0.12.0
- Ultralytics: 8.0.196

### 1. สร้าง Folder
```bash
mkdir -p ~/yolov8
cd ~/yolov8
```

### 2. ติดตั้ง Dependencies
```bash
sudo apt-get update
sudo apt-get install -y python3-pip libopenblas-base libopenmpi-dev \
libjpeg-dev zlib1g-dev libpython3-dev \
libavcodec-dev libavformat-dev libswscale-dev

sudo -H pip3 install future testresources Cython gdown
sudo -H pip3 install setuptools==58.3.0
sudo pip3 install -U --user wheel mock pillow
```

### 3. ติดตั้ง PyTorch 1.11.0 (Python 3.8)

> ⚠️ ต้องโหลด wheel จาก Google Drive บน PC แล้ว copy มาด้วย SCP
> Link: https://drive.google.com/uc?id=1AQQuBS9skNk1mgZXMp0FmTIwjuxc81WY

```bash
# รันบน PC เพื่อ copy ไฟล์มา Jetson
scp torch-1.11.0a0+gitbc2c6ed-cp38-cp38-linux_aarch64.whl er@<JETSON_IP>:/home/er/yolov8/

# ติดตั้งบน Jetson
sudo -H pip3 install torch-1.11.0a0+gitbc2c6ed-cp38-cp38-linux_aarch64.whl
```

เช็คว่าสำเร็จ:
```bash
python3 -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"
# ผลที่ควรได้: 1.11.0a0+gitbc2c6ed / True
```

### 4. Build torchvision 0.12.0

```bash
cd ~/yolov8
git clone --branch v0.12.0 https://github.com/pytorch/vision torchvision
cd torchvision
export BUILD_VERSION=0.12.0
sudo -H python3 setup.py install
cd ..
```

> ⏱️ ใช้เวลาประมาณ 15-30 นาที

เช็คว่าสำเร็จ:
```bash
cd ~/yolov8
python3 -c "import torchvision; print(torchvision.__version__)"
# ผลที่ควรได้: 0.12.0a0+9b5a3fe
```

### 5. ติดตั้ง Ultralytics
```bash
sudo -H pip3 install ultralytics
```

### 6. Copy โมเดลจาก PC
```bash
# รันบน PC (CMD)
scp D:\yolov8\runs\detect\train\weights\best.pt er@<JETSON_IP>:/home/er/yolov8/
```

### 7. ไฟล์ detect.py
```python
from ultralytics import YOLO
import cv2
import time

model = YOLO("best.pt")
cap = cv2.VideoCapture(0)

cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)

TARGET_FPS = 10
frame_delay = 1.0 / TARGET_FPS

while True:
    start = time.time()

    ret, frame = cap.read()
    if not ret:
        print("ไม่พบกล้อง")
        break

    results = model(frame, imgsz=320, conf=0.5)
    annotated = results[0].plot()

    fps = 1.0 / (time.time() - start)
    cv2.putText(annotated, f"FPS: {fps:.1f}", (10, 30),
                cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)

    cv2.imshow("Candy Detection", annotated)

    elapsed = time.time() - start
    wait = max(1, int((frame_delay - elapsed) * 1000))
    if cv2.waitKey(wait) == ord("q"):
        break

cap.release()
cv2.destroyAllWindows()
```

### 8. รัน Detection
```bash
cd ~/yolov8
python3 detect.py
```

---

## ⚠️ ปัญหาที่พบและวิธีแก้

### 1. Wheel ไม่ตรง Platform
```
ERROR: not a supported wheel on this platform
```
**แก้:** เช็ค Python version ให้ตรงกับ wheel
```bash
python3 --version  # ต้องได้ 3.8.x
```

### 2. Google Drive Access Denied
```
Access denied with the following error
```
**แก้:** โหลดบน PC ผ่าน Browser แล้วใช้ SCP copy มา Jetson
```bash
scp <filename> er@<JETSON_IP>:/home/er/yolov8/
```

### 3. CUDA Not Available
```python
torch.cuda.is_available()  # False
```
**แก้:** ตรวจสอบว่าติดตั้ง PyTorch wheel ของ NVIDIA โดยตรง ไม่ใช่จาก PyPI

### 4. venv/Scripts/activate ไม่ได้ (Windows)
```
'venv' is not recognized
```
**แก้:** ใช้ backslash `\` ไม่ใช่ `/`
```bash
venv\Scripts\activate.bat
```

---

## 📁 โครงสร้าง Project

```
# PC
yolov8/
├── venv/
├── dataset/
│   ├── train/
│   ├── valid/
│   └── data.yaml
├── train.py
└── runs/detect/train/weights/
    └── best.pt ← copy ไป Jetson

# Jetson Nano
~/yolov8/
├── best.pt
├── detect.py
└── torchvision/
```

---

## 🔧 Parameter ที่ควรปรับ

| Parameter | PC | Jetson Nano |
|---|---|---|
| imgsz | 640 | 320 |
| batch | 16 | - |
| TARGET_FPS | - | 10 |
| conf | - | 0.5 |
