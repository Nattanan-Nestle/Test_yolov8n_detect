#YOLOv8 Detection on Jetson Nano

โปรเจคตรวจจับขนมด้วย YOLOv8n บน Jetson Nano 4GB

## 📋 สิ่งที่ต้องเตรียม
- PC สำหรับเทรนโมเดล
- Jetson Nano 4GB
- กล้อง USB
- Dataset ต่างๆ (Candy, Chocolate, Gummy, Halls)

---

## 💻 ฝั่ง PC (Windows)

### 1. สร้าง Environment
```bash
1. cd "C:\Users\<ชื่อ User>\Desktop"
2. mkdir yolov8
3. cd yolov8
4. python -m venv venv
5. venv\Scripts\activate.bat
```

### 2. ติดตั้ง Package
```bash
เมื่อเข้าไปที่ yolov8 เเล้วเปิดการใช้งาน Environment ด้วยคำสั่ง python -m venv venv จากนั้นใช้คำสั่งติดตั้ง
1. pip install torch torchvision --index-url https://download.pytorch.org/whl/cu126
2. pip install ultralytics
```

### 3. เทรนโมเดล
```bash
python train.py
```

โมเดลจะถูกบันทึกที่ `runs/detect/train/weights/best.pt`

---

## 🤖 ฝั่ง Jetson Nano

### ข้อมูล Environment
- JetPack: 4.6.1 (R32.6.1)
- Python: 3.8.10
- PyTorch: 1.11.0
- torchvision: 0.12.0

### 1. ติดตั้ง Dependencies
```bash
sudo apt-get install -y python3-pip libopenblas-base \
libopenmpi-dev libjpeg-dev zlib1g-dev
sudo -H pip3 install future testresources Cython
sudo -H pip3 install setuptools==58.3.0
```

### 2. ติดตั้ง PyTorch 1.11.0
```bash
# โหลด wheel จาก Google Drive แล้ว copy มาด้วย SCP
scp torch-1.11.0a0+gitbc2c6ed-cp38-cp38-linux_aarch64.whl \
    er@<JETSON_IP>:/home/er/yolov8/

sudo -H pip3 install torch-1.11.0a0+gitbc2c6ed-cp38-cp38-linux_aarch64.whl
```

### 3. Build torchvision
```bash
sudo apt-get install -y libjpeg-dev zlib1g-dev \
libpython3-dev libavcodec-dev libavformat-dev libswscale-dev

git clone --branch v0.12.0 https://github.com/pytorch/vision torchvision
cd torchvision
export BUILD_VERSION=0.12.0
sudo -H python3 setup.py install
cd ..
```

### 4. ติดตั้ง Ultralytics
```bash
sudo -H pip3 install ultralytics
```

### 5. Copy โมเดลจาก PC
```bash
# รันบน PC
scp runs/detect/train/weights/best.pt er@<JETSON_IP>:/home/er/yolov8/
```

### 6. รัน Detection
```bash
cd ~/yolov8
python3 detect.py
```

---

## 📊 ผลการเทรน

| Class | mAP@50 | mAP@50-95 |
|---|---|---|
| All | 0.995 | 0.819 |
| Candy | 0.995 | 0.895 |
| Chocolate | 0.995 | 0.856 |
| Gummy | 0.995 | 0.749 |
| Halls | 0.995 | 0.775 |

---

## ⚠️ ปัญหาที่พบและวิธีแก้

### PyTorch wheel ไม่รองรับ
