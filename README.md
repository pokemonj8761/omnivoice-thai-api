# 🎙️ OmniVoice Thai API

> 🇹🇭 **Zero-shot Thai TTS** — สังเคราะห์เสียงภาษาไทยด้วย AI ไม่ต้องเทรนเพิ่ม
>
> Web UI + REST API รองรับ Voice Cloning, Voice Design, Auto Voice

Powered by [hotdogs/omnivoice-thai](https://huggingface.co/hotdogs/omnivoice-thai) — fine-tuned บนข้อมูลเสียงภาษาไทย 20,000 ประโยค (~12.6 ชั่วโมง)

---

## ✨ Features / ความสามารถ

| Mode / โหมด | Description / คำอธิบาย | Input |
|-------------|------------------------|-------|
| 🎤 **Voice Cloning** | โคลนเสียงจากไฟล์อ้างอิง 3–10 วินาที | `ref_audio` + `text` |
| 🎨 **Voice Design** | ออกแบบเสียงด้วยคำสั่ง (เพศ, อายุ, สำเนียง) | `instruct` + `text` |
| 🤖 **Auto Voice** | ให้ AI เลือกเสียงที่เหมาะสมให้อัตโนมัติ | `text` อย่างเดียว |

---

## 🚀 ติดตั้ง — คำสั่งเดียว

```bash
curl -fsSL https://raw.githubusercontent.com/nanofatdog/omnivoice-thai-api/main/install.sh | bash
```

ติดตั้งอัตโนมัติทุกขั้นตอน:
1. ตรวจสอบ Python 3.9+, พื้นที่ดิสก์
2. ติดตั้ง PyTorch + CUDA, omnivoice, FastAPI, uvicorn
3. โหลดโมเดล (~4.4GB): `hf download hotdogs/omnivoice-thai --local-dir ~/omnivoice-thai`
4. โหลด `server.py` จาก GitHub
5. สตาร์ทเซิร์ฟเวอร์ที่ port `7860`

### หรือติดตั้งทีละขั้นตอน

```bash
# ติดตั้ง dependencies
pip install "huggingface_hub[cli]" omnivoice fastapi "uvicorn[standard]" soundfile python-multipart

# ติดตั้ง PyTorch (ถ้ายังไม่มี)
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cu121

# โหลดโมเดล
hf download hotdogs/omnivoice-thai --local-dir ~/omnivoice-thai

# โหลด server
curl -fsSL -o ~/omnivoice-thai/server.py \
  https://raw.githubusercontent.com/nanofatdog/omnivoice-thai-api/main/server.py

# สตาร์ท
cd ~/omnivoice-thai && python3 server.py
```

### ตั้งค่าเพิ่มเติม

```bash
# เปลี่ยน port หรือที่อยู่โมเดล
OMNIVOICE_PORT=9000 OMNIVOICE_MODEL_DIR=/data/omnivoice curl -fsSL https://.../install.sh | bash
```

---

## 🐳 Docker

### วิธีที่ 1: Pull จาก Docker Hub (เร็วสุด)

```bash
docker pull aidogs/omnivoice-thai-api:latest
docker run --gpus all -p 7860:7860 aidogs/omnivoice-thai-api:latest
```

ขนาด image ~6GB (รวม CUDA 12.1 + PyTorch + OmniVoice + โมเดล)

### วิธีที่ 2: Build เองจากโค้ด

```bash
git clone https://github.com/nanofatdog/omnivoice-thai-api.git
cd omnivoice-thai-api
docker compose up -d --build
```

ครั้งแรกใช้เวลาโหลดโมเดลเข้า VRAM ~90 วิ

หรือใช้ image สำเร็จรูปใน `docker-compose.yml`:

```yaml
services:
  omnivoice-thai:
    image: aidogs/omnivoice-thai-api:latest   # pull จาก Hub
    # build: .                                 # หรือ build เอง
    ...
```

### วิธีที่ 3: ใช้โมเดลจากเครื่อง host (image เล็กลง)

ถ้ามีโมเดลอยู่แล้วที่ `~/omnivoice-thai/`:

```yaml
# แก้ docker-compose.yml — เพิ่ม volume mount:
volumes:
  - ~/omnivoice-thai:/app/model    # mount โมเดลจาก host
  - ./outputs:/app/outputs
```

```bash
# แล้ว comment out บรรทัดโหลดโมเดลใน Dockerfile:
# RUN mkdir -p /app/model ... && hf download ...
```

Image เหลือ ~1.5GB แทนที่จะเป็น ~6GB

### เปลี่ยน GPU

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          device_ids: ['1']    # เปลี่ยนเป็น GPU เบอร์ที่ต้องการ
          capabilities: [gpu]
```

---

## 📡 API

### `GET /api/health` — ตรวจสอบสถานะ

```bash
curl http://localhost:7860/api/health
```

```json
{
  "status": "ok",
  "model_loaded": true,
  "device": "cuda:0",
  "gpu_name": "NVIDIA GeForce RTX 3060",
  "vram_total_gb": 7.7,
  "vram_used_gb": 3.4
}
```

### `POST /api/generate` — สร้างเสียง (form-data)

ส่งข้อความ รับไฟล์ WAV กลับ

**🤖 Auto Voice** (ไม่ต้องใช้เสียงอ้างอิง):
```bash
curl -X POST http://localhost:7860/api/generate \
  -F "text=สวัสดีครับ วันนี้อากาศดีมาก" \
  -F "mode=auto" \
  -o output.wav
```

**🎤 Voice Cloning** (ใช้ไฟล์เสียงอ้างอิง):
```bash
curl -X POST http://localhost:7860/api/generate \
  -F "text=สวัสดีค่ะ ยินดีที่ได้รู้จัก" \
  -F "mode=clone" \
  -F "ref_audio=@my_voice.wav" \
  -F "ref_text=ข้อความในไฟล์อ้างอิง" \
  -o cloned.wav
```

**🎨 Voice Design** (ออกแบบเสียงด้วยคำสั่ง):
```bash
curl -X POST http://localhost:7860/api/generate \
  -F "text=สวัสดีค่ะ" \
  -F "mode=design" \
  -F "instruct=female, young adult, high pitch" \
  -o designed.wav
```

### `POST /api/generate/json` — สร้างเสียง (JSON)

```bash
curl -X POST http://localhost:7860/api/generate/json \
  -H "Content-Type: application/json" \
  -d '{"text":"สวัสดีครับ","mode":"auto"}'
```

```json
{
  "audio_b64": "UklGRiR...",
  "sample_rate": 24000,
  "duration_s": 2.14
}
```

สำหรับ Voice Cloning แบบ JSON — ส่ง `ref_audio_b64` (base64 ของไฟล์ WAV)

### Parameters / พารามิเตอร์

| Param | Type | จำเป็น? | คำอธิบาย |
|-------|------|---------|----------|
| `text` | string | ✅ | ข้อความที่ต้องการให้พูด (สูงสุด 2000 ตัวอักษร) |
| `mode` | string | - | `auto` (ค่าเริ่มต้น), `clone`, `design` |
| `ref_audio` | file | clone | ไฟล์เสียงอ้างอิง WAV/MP3/FLAC (3–10 วิ) |
| `ref_text` | string | - | บทถอดเสียงของไฟล์อ้างอิง (เว้นว่าง = ถอดเสียงอัตโนมัติ) |
| `instruct` | string | design | คำอธิบายเสียง (ดูตารางด้านล่าง) |

### 🎨 Voice Design — ค่าที่ใช้ได้

| หมวดหมู่ | ค่าที่ใช้ได้ |
|----------|-------------|
| **เพศ** | `female`, `male` |
| **อายุ** | `child`, `teenager`, `young adult`, `middle-aged`, `elderly` |
| **ระดับเสียง** | `very low pitch`, `low pitch`, `moderate pitch`, `high pitch`, `very high pitch` |
| **สำเนียง** | `american`, `australian`, `british`, `canadian`, `chinese`, `indian`, `japanese`, `korean`, `portuguese`, `russian` + `accent` |
| **สไตล์** | `whisper` |

ตัวอย่าง:
```bash
# หญิงสาวเสียงสูง
curl -X POST http://localhost:7860/api/generate \
  -F "text=สวัสดีค่ะ" -F "mode=design" \
  -F "instruct=female, young adult, high pitch"

# ผู้ชายวัยกลางคนเสียงทุ้ม
curl -F "text=ขอแนะนำสินค้าใหม่" -F "mode=design" \
  -F "instruct=male, middle-aged, low pitch"

# เสียงกระซิบสำเนียงอังกฤษ
curl -F "text=ความลับอยู่ที่นี่" -F "mode=design" \
  -F "instruct=male, whisper, british accent"
```

> ⚠️ ใส่เฉพาะค่าจากตารางด้านบนเท่านั้น — ค่าที่ไม่รองรับ (เช่น `cheerful`, `sad`, `deep`) จะถูกกรองออกอัตโนมัติ และแจ้งเตือนใน server log
>
> ใช้ภาษาจีนก็ได้ (เช่น `女，青年，高音调`)

---

## 🖥️ Web UI

เปิด `http://localhost:7860` ในเบราว์เซอร์ — มี 3 แท็บพร้อมเล่นเสียงและประวัติ

---

## 📋 ความต้องการของระบบ

| ทรัพยากร | ขั้นต่ำ | แนะนำ |
|----------|---------|-------|
| GPU VRAM | 4 GB | 8 GB |
| RAM | 8 GB | 16 GB |
| พื้นที่ดิสก์ | 8 GB | 15 GB |
| Python | 3.9+ | 3.11+ |
| CUDA | 11.8+ | 12.1+ |

> ใช้ CPU ได้ แต่ใช้เวลา 30–60 วินาทีต่อครั้ง (GPU ใช้ 3–10 วิ)

---

## 🛠️ การจัดการเซิร์ฟเวอร์

```bash
# สตาร์ท
bash ~/omnivoice-thai/start.sh

# หยุด
pkill -f server.py

# ดู log
tail -f ~/omnivoice-thai/server.log
```

---

## 🔧 ตัวแปรสภาพแวดล้อม (Environment Variables)

| ตัวแปร | ค่าเริ่มต้น | คำอธิบาย |
|--------|-----------|----------|
| `OMNIVOICE_MODEL_PATH` | `~/omnivoice-thai` | ที่ตั้งโฟลเดอร์โมเดล |
| `OMNIVOICE_PORT` | `7860` | พอร์ตเซิร์ฟเวอร์ |
| `OMNIVOICE_HOST` | `0.0.0.0` | IP ที่ใช้ bind |
| `OMNIVOICE_DEVICE` | `cuda:0` | อุปกรณ์ Torch |

---

## 📄 License

MIT

## 🙏 Credits

- [OmniVoice](https://github.com/k2-fsa/OmniVoice) โดย k2-fsa
- [omnivoice-thai](https://huggingface.co/hotdogs/omnivoice-thai) โดย hotdogs
