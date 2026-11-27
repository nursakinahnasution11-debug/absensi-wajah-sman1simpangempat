
<!DOCTYPE html>
<html>
<head>
  <base target="_top">
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <script defer src="https://cdn.jsdelivr.net/npm/face-api.js@0.22.2/dist/face-api.min.js"></script>

  <style>
    :root {
      font-family: Arial, sans-serif;
      color: #172033;
      background: #f3f6fb;
    }

    * {
      box-sizing: border-box;
    }

    body {
      margin: 0;
      padding: 20px;
    }

    .wrap {
      max-width: 980px;
      margin: auto;
    }

    .head {
      background: #123b68;
      color: white;
      padding: 22px;
      border-radius: 16px;
    }

    .head h1 {
      margin: 0 0 8px;
      font-size: 24px;
    }

    .grid {
      display: grid;
      grid-template-columns: 1.4fr 1fr;
      gap: 16px;
      margin-top: 16px;
    }

    .card {
      background: white;
      border-radius: 16px;
      padding: 18px;
      box-shadow: 0 5px 20px #102e5010;
    }

    .camera {
      width: 100%;
      aspect-ratio: 16/9;
      object-fit: cover;
      background: #111;
      border-radius: 12px;
    }

    .row {
      display: flex;
      gap: 10px;
      flex-wrap: wrap;
      margin-top: 12px;
    }

    button,
    select,
    input {
      font-size: 15px;
      padding: 11px;
      border: 1px solid #cbd5e1;
      border-radius: 9px;
    }

    button {
      cursor: pointer;
      background: #1d5b95;
      color: white;
      border: 0;
      font-weight: 700;
    }

    button.secondary {
      background: #e8eef5;
      color: #123b68;
    }

    button:disabled {
      opacity: .5;
      cursor: wait;
    }

    .status {
      padding: 12px;
      border-radius: 10px;
      background: #edf3fa;
      margin-top: 12px;
      min-height: 44px;
    }

    .small {
      font-size: 12px;
      color: #64748b;
      line-height: 1.5;
    }

    .admin {
      border-top: 1px solid #e2e8f0;
      margin-top: 16px;
      padding-top: 16px;
    }

    .hide {
      display: none;
    }

    @media(max-width:700px) {
      body {
        padding: 10px;
      }

      .grid {
        grid-template-columns: 1fr;
      }

      .head h1 {
        font-size: 20px;
      }
    }
  </style>
</head>

<body>
  <div class="wrap">

    <div class="head">
      <h1>ABSENSI WAJAH SISWA</h1>
      <div>
        SMA Negeri 1 Simpang Empat • Kedatangan pagi
      </div>
    </div>

    <div class="grid">

      <!-- BAGIAN ABSENSI -->
      <section class="card">

        <video
          id="video"
          class="camera"
          autoplay
          muted
          playsinline>
        </video>

        <canvas id="overlay" class="hide"></canvas>

        <div class="row">
          <button id="start">
            Aktifkan Kamera
          </button>

          <button id="scan" disabled>
            Mulai Absensi
          </button>
        </div>

        <div id="result" class="status">
          Memuat aplikasi…
        </div>

        <p class="small">
          Pastikan hanya satu wajah terlihat, pencahayaan cukup,
          dan siswa berdiri menghadap kamera.
          Jika tidak dikenali, arahkan ke petugas absensi.
        </p>

      </section>

      <!-- BAGIAN PENDAFTARAN WAJAH -->
      <section class="card">

        <h2 style="margin-top:0">
          Pendaftaran Wajah
        </h2>

        <p class="small">
          Bagian ini hanya untuk petugas admin.
          Data template wajah disimpan di spreadsheet sekolah
          dan tidak ditampilkan ke layar absensi.
        </p>

        <div class="row">
          <select id="student">
            <option value="">
              Memuat daftar siswa…
            </option>
          </select>
        </div>

        <div class="row">
          <input
            id="pin"
            type="password"
            placeholder="PIN admin"
            autocomplete="off">
        </div>

        <div class="row">
          <button
            id="enroll"
            class="secondary"
            disabled>
            Daftarkan Wajah Terpilih
          </button>
        </div>

        <div class="small">
          Daftarkan wajah dengan persetujuan yang diperlukan.
          Lakukan dari perangkat dan akun sekolah yang dikelola.
        </div>

      </section>

    </div>

    <div class="card admin">
      <b>Catatan pengelolaan</b>

      <p class="small">
        Absensi ini merupakan prototipe awal.
        Sebelum dipakai resmi, uji dengan data terbatas,
        atur akses spreadsheet, ganti PIN admin,
        isi persetujuan orang tua, dan siapkan jalur absensi manual.
      </p>
    </div>

  </div>

  <script>
    const MODEL_URL =
      'https://cdn.jsdelivr.net/gh/justadudewhohacks/face-api.js@master/weights';

    let cameraOn = false;
    let modelsReady = false;
    let busy = false;

    const $ = id => document.getElementById(id);
    const video = $('video');

    function say(msg) {
      $('result').textContent = msg;
    }

    // MEMUAT MODEL PENGENALAN WAJAH
    async function loadModels() {
      if (modelsReady) return;

      say('Memuat model pengenalan wajah…');

      await Promise.all([
        faceapi.nets.tinyFaceDetector.loadFromUri(MODEL_URL),
        faceapi.nets.faceLandmark68Net.loadFromUri(MODEL_URL),
        faceapi.nets.faceRecognitionNet.loadFromUri(MODEL_URL)
      ]);

      modelsReady = true;

      say('Model siap. Aktifkan kamera.');
    }

    // MENGAKTIFKAN KAMERA
    $('start').onclick = async () => {
      try {
        await loadModels();

        const stream =
          await navigator.mediaDevices.getUserMedia({
            video: {
              facingMode: 'user',
              width: { ideal: 1280 },
              height: { ideal: 720 }
            },
            audio: false
          });

        video.srcObject = stream;

        await video.play();

        cameraOn = true;

        $('scan').disabled = false;
        $('enroll').disabled = false;

        say('Kamera aktif. Siswa dapat mulai absensi.');

      } catch (e) {
  console.error("Camera error:", e);

  if (e.name === "NotAllowedError") {
    say(
      "Akses kamera ditolak. " +
      "Klik ikon pengaturan situs di Chrome, " +
      "ubah Kamera menjadi Izinkan, " +
      "kemudian muat ulang halaman."
    );
  } else if (e.name === "NotFoundError") {
    say(
      "Webcam tidak ditemukan. " +
      "Pastikan kamera terhubung ke laptop."
    );
  } else if (e.name === "NotReadableError") {
    say(
      "Kamera sedang digunakan aplikasi lain. " +
      "Tutup Zoom, Meet, atau aplikasi kamera lainnya."
    );
  } else {
    say(
      "Kamera gagal dibuka: " +
      e.name + " - " + e.message
    );
  }
}
    };

    // MENGAMBIL TEMPLATE WAJAH
    async function getDescriptor() {
      if (!cameraOn || !modelsReady) {
        throw new Error(
          'Aktifkan kamera terlebih dahulu.'
        );
      }

      const detections =
        await faceapi
          .detectAllFaces(
            video,
            new faceapi.TinyFaceDetectorOptions({
              inputSize: 320,
              scoreThreshold: 0.55
            })
          )
          .withFaceLandmarks()
          .withFaceDescriptors();

      if (detections.length !== 1) {
        throw new Error(
          detections.length === 0
            ? 'Wajah belum terdeteksi.'
            : 'Pastikan hanya satu wajah di depan kamera.'
        );
      }

      return Array.from(
        detections[0].descriptor
      );
    }

    // PROSES ABSENSI
    $('scan').onclick = async () => {
      if (busy) return;

      busy = true;
      $('scan').disabled = true;

      try {
        say('Memindai wajah…');

        const descriptor =
          await getDescriptor();

        say('Memeriksa kecocokan…');

        google.script.run
          .withSuccessHandler(r => {
            say(
              r.message +
              (r.whatsapp
                ? ' • WA: ' + r.whatsapp
                : '')
            );

            busy = false;
            $('scan').disabled = false;
          })
          .withFailureHandler(e => {
            say(
              'Gagal menyimpan: ' +
              e.message
            );

            busy = false;
            $('scan').disabled = false;
          })
          .matchAndRecord(descriptor);

      } catch (e) {
        say(e.message);

        busy = false;
        $('scan').disabled = false;
      }
    };

    // MEMUAT DAFTAR SISWA
    function loadStudents() {
      google.script.run
        .withSuccessHandler(list => {
          const s = $('student');

          s.innerHTML =
            '<option value="">Pilih siswa…</option>';

          list.forEach(x => {
            const o =
              document.createElement('option');

            o.value = x.nisn;

            o.textContent =
              x.nama +
              ' — ' +
              x.kelas +
              ' (' +
              x.nisn +
              ')';

            s.appendChild(o);
          });
        })
        .withFailureHandler(e => {
          say(
            'Tidak dapat memuat siswa: ' +
            e.message
          );
        })
        .getStudentsForRegistration();
    }

    // PENDAFTARAN WAJAH SISWA
    $('enroll').onclick = async () => {
      try {
        const nisn = $('student').value;
        const pin = $('pin').value;

        if (!nisn || !pin) {
          throw new Error(
            'Pilih siswa dan masukkan PIN admin.'
          );
        }

        say('Mengambil template wajah…');

        const descriptor =
          await getDescriptor();

        google.script.run
          .withSuccessHandler(r => {
            say(r.message);
          })
          .withFailureHandler(e => {
            say(
              'Pendaftaran gagal: ' +
              e.message
            );
          })
          .registerFace(
            nisn,
            descriptor,
            pin
          );

      } catch (e) {
        say(e.message);
      }
    };

    // SAAT APLIKASI DIBUKA
    window.addEventListener('load', () => {
      if (window.google && google.script) {
        loadStudents();
      }

      say(
        'Klik Aktifkan Kamera untuk memulai.'
      );
    });
  </script>
</body>
</html>
