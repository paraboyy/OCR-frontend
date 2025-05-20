<template>
  <div class="container">
    <h1>Deteksi Kode Batch Obat-2</h1>
    <input
      type="text"
      v-model="userInput"
      placeholder="Nama Pengirim (opsional)"
      class="input"
    />
    <div class="mode-group">
      <label><input type="radio" v-model="mode" value="barcode" /> Barcode</label>
      <label><input type="radio" v-model="mode" value="qr" /> QR Code</label>
      <label><input type="radio" v-model="mode" value="ocr" /> Teks (OCR)</label>
    </div>
    <div class="preview-area">
      <video ref="video" width="100%" height="auto" autoplay style="display: block"></video>
      <canvas ref="canvas" style="display: none"></canvas>
      <img
        ref="imgPreview"
        class="img-preview"
        style="display: none"
        alt="Preview"
      />
    </div>
    <div class="action-group">
      <button @click="captureFromCamera">
        <span class="icon">&#128247;</span> Ambil Gambar dari Kamera
      </button>
      <label for="fileInput" class="upload-label">
        <span class="icon">&#128206;</span> Upload Gambar
      </label>
      <input
        type="file"
        id="fileInput"
        accept="image/*"
        @change="onFileChange"
        style="display: none"
      />
    </div>
    <div id="result" v-html="result" class="result"></div>
    <div id="ocrText" class="ocr-text">{{ ocrText }}</div>
  </div>
</template>

<script setup>
import { ref } from "vue";

const userInput = ref("");
const mode = ref("barcode");
const result = ref("");
const ocrText = ref("");

const video = ref(null);
const canvas = ref(null);
const imgPreview = ref(null);

function getMode() {
  return mode.value;
}

function captureFromCamera() {
  navigator.mediaDevices
    .getUserMedia({ video: true })
    .then((stream) => {
      video.value.srcObject = stream;
      video.value.onloadedmetadata = () => {
        canvas.value.width = video.value.videoWidth;
        canvas.value.height = video.value.videoHeight;

        imgPreview.value.style.display = "none";
        video.value.style.display = "block";
        const context = canvas.value.getContext("2d");
        context.drawImage(video.value, 0, 0, canvas.value.width, canvas.value.height);
        result.value = "Memproses gambar...";
        ocrText.value = "";

        const currentMode = getMode();
        const imageData = context.getImageData(0, 0, canvas.value.width, canvas.value.height);

        if (currentMode === "qr") {
          const qr = window.jsQR(imageData.data, imageData.width, imageData.height);
          if (qr && qr.data) {
            tampilkanHasilBatch(qr.data, "QR Code");
          } else {
            result.value = '<span class="error">QR Code tidak ditemukan.</span>';
          }
        } else if (currentMode === "barcode") {
          const dataUrl = canvas.value.toDataURL("image/png");
          detectBarcodeFromImageData(dataUrl, canvas.value.width, canvas.value.height, function (barcode) {
            if (barcode) {
              tampilkanHasilBatch(barcode, "Barcode");
            } else {
              result.value = '<span class="error">Barcode tidak ditemukan.</span>';
            }
          });
        } else {
          window.Tesseract.recognize(canvas.value, "eng", { logger: (m) => console.log(m) })
            .then(({ data: { text } }) => {
              handleOCRResult(text);
            })
            .catch((error) => {
              console.error("Error OCR:", error);
              result.value = '<span class="error">Error memproses gambar.</span>';
            });
        }
      };
    })
    .catch((err) => {
      console.error("Error mengakses kamera:", err);
      if (location.protocol !== "https:" && location.hostname !== "localhost" && location.hostname !== "127.0.0.1") {
        result.value = '<span class="error">Gagal mengakses kamera.<br>Browser hanya mengizinkan akses kamera pada <b>HTTPS</b> atau <b>localhost</b>.<br>Silakan akses web ini melalui HTTPS atau localhost.</span>';
      } else {
        result.value = '<span class="error">Gagal mengakses kamera: ' + err.message + "</span>";
      }
    });
}

function onFileChange(e) {
  const file = e.target.files[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = function (ev) {
      imgPreview.value.src = ev.target.result;
      imgPreview.value.style.display = "block";
      imgPreview.value.style.maxWidth = "100%";
      imgPreview.value.style.height = "auto";
      video.value.style.display = "none";
      result.value = "Memproses gambar...";
      ocrText.value = "";

      const img = new window.Image();
      img.onload = function () {
        canvas.value.width = img.width;
        canvas.value.height = img.height;
        const ctx = canvas.value.getContext("2d");
        ctx.drawImage(img, 0, 0, img.width, img.height);

        const currentMode = getMode();
        const imageData = ctx.getImageData(0, 0, img.width, img.height);

        if (currentMode === "qr") {
          const qr = window.jsQR(imageData.data, imageData.width, imageData.height);
          if (qr && qr.data) {
            tampilkanHasilBatch(qr.data, "QR Code");
          } else {
            result.value = '<span class="error">QR Code tidak ditemukan.</span>';
          }
        } else if (currentMode === "barcode") {
          const dataUrl = canvas.value.toDataURL("image/png");
          detectBarcodeFromImageData(dataUrl, canvas.value.width, canvas.value.height, function (barcode) {
            if (barcode) {
              tampilkanHasilBatch(barcode, "Barcode");
            } else {
              result.value = '<span class="error">Barcode tidak ditemukan.</span>';
            }
          });
        } else {
          window.Tesseract.recognize(ev.target.result, "eng", { logger: (m) => console.log(m) })
            .then(({ data: { text } }) => {
              handleOCRResult(text);
            })
            .catch((error) => {
              console.error("Error OCR:", error);
              result.value = '<span class="error">Error memproses gambar.</span>';
            });
        }
      };
      img.src = ev.target.result;
    };
    reader.readAsDataURL(file);
  }
}

function handleOCRResult(text) {
  ocrText.value = "Hasil OCR:\n" + text;
  const lines = text
    .split("\n")
    .map((line) => line.trim())
    .filter((line) => line.length > 0);
  let codeFound = false;

  const regexSameLine = /(batch\s*(no|number)?[\s\.:;\-]*)+([A-Za-z0-9\-]{3,})/i;
  const regexBatchOnly = /(batch\s*(no|number)?[\s\.:;\-]*)$/i;
  const regexCodeOnly = /^([A-Za-z0-9\-]{3,})$/;

  for (let i = 0; i < lines.length; i++) {
    let match = lines[i].match(regexSameLine);
    if (match && match[3]) {
      const code = match[3].trim();
      tampilkanHasilBatch(code, "OCR");
      codeFound = true;
      break;
    }
    if (regexBatchOnly.test(lines[i]) && i + 1 < lines.length) {
      let nextLine = lines[i + 1].trim();
      let codeMatch = nextLine.match(regexCodeOnly);
      if (codeMatch && codeMatch[1]) {
        const code = codeMatch[1].trim();
        tampilkanHasilBatch(code, "OCR");
        codeFound = true;
        break;
      }
    }
  }
  if (!codeFound) {
    result.value = '<span class="error">Kode tidak ditemukan. Coba lagi.</span>';
  }
}

function tampilkanHasilBatch(code, sumber) {
  result.value = `<span class="success">Kode terdeteksi (${sumber}): ${code}</span>`;
  fetch("http://127.0.0.1:3000/save-code", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ code: code, user: userInput.value }),
  })
    .then((response) => response.json())
    .then((data) => {
      if (data.message) {
        result.value += '<br><span class="success">' + data.message + "</span>";
      }
    })
    .catch((error) => {
      console.error("Error:", error);
      result.value += '<br><span class="error">Gagal mengirim kode ke server.</span>';
    });
}

// Fungsi deteksi barcode 1D (QuaggaJS)
function detectBarcodeFromImageData(imageData, width, height, callback) {
  window.Quagga.decodeSingle(
    {
      src: imageData,
      inputStream: {
        size: width,
        singleChannel: false,
      },
      locator: { patchSize: "medium", halfSample: false },
      decoder: { readers: ["code_128_reader", "ean_reader", "ean_8_reader", "code_39_reader", "upc_reader", "upc_e_reader"] },
      locate: true,
      numOfWorkers: 0,
    },
    function (result) {
      if (result && result.codeResult && result.codeResult.code) {
        callback(result.codeResult.code);
      } else {
        callback(null);
      }
    }
  );
}
</script>

<style scoped>
body {
  font-family: "Segoe UI", Arial, sans-serif;
  background: linear-gradient(135deg, #e0e7ff 0%, #f7f7f7 100%);
  margin: 0;
  padding: 0;
}
.container {
  background: #fff;
  margin: 40px auto;
  padding: 32px 24px 40px 24px;
  border-radius: 18px;
  box-shadow: 0 8px 32px rgba(44, 62, 80, 0.12);
  max-width: 480px;
}
h1 {
  color: #2563eb;
  margin-bottom: 18px;
  letter-spacing: 1px;
}
.input {
  padding: 10px;
  border-radius: 8px;
  border: 1px solid #b6c3d1;
  font-size: 16px;
  margin-bottom: 12px;
  width: 100%;
  box-sizing: border-box;
}
.mode-group {
  margin-bottom: 12px;
  display: flex;
  gap: 18px;
  align-items: center;
}
.mode-group label {
  font-size: 15px;
  color: #2563eb;
  font-weight: 600;
  cursor: pointer;
}
.preview-area {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 10px;
}
.img-preview {
  display: block;
  margin: 0 auto 12px auto;
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(44, 62, 80, 0.06);
  max-width: 100%;
  height: auto;
  object-fit: contain;
  background: #f3f4f6;
}
.action-group {
  display: flex;
  gap: 10px;
  margin-bottom: 10px;
  flex-wrap: wrap;
}
button,
.upload-label {
  background: linear-gradient(90deg, #2563eb 60%, #60a5fa 100%);
  color: #fff;
  border: none;
  padding: 12px 24px;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  margin: 8px 0 0 0;
  transition: background 0.2s;
  display: inline-flex;
  align-items: center;
  gap: 8px;
}
button:active,
.upload-label:active {
  background: linear-gradient(90deg, #1e40af 60%, #2563eb 100%);
}
.upload-label {
  cursor: pointer;
}
#fileInput {
  display: none;
}
.result {
  margin-top: 18px;
  font-size: 18px;
  color: #222;
  min-height: 32px;
  word-break: break-word;
}
.ocr-text {
  margin-top: 10px;
  font-size: 14px;
  color: #555;
  background: #f3f4f6;
  border-radius: 8px;
  padding: 10px;
  word-break: break-all;
}
.success {
  color: #2563eb;
  font-weight: 600;
}
.error {
  color: #e11d48;
  font-weight: 600;
}
.icon {
  font-size: 18px;
  vertical-align: middle;
}
@media (max-width: 600px) {
  .container {
    padding: 12px 2vw;
  }
  h1 {
    font-size: 22px;
  }
  .action-group {
    flex-direction: column;
    gap: 8px;
  }
}
</style>