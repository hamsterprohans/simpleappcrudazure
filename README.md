# W11 - CI/CD Next.js CRUD ke Azure App Service

Aplikasi CRUD berbasis Next.js yang di-deploy otomatis ke **Azure App Service** menggunakan **GitHub Actions**, dengan database **Azure Cosmos DB (MongoDB API)**. Setiap `git push` ke branch `main` akan menjalankan build dan deployment tanpa langkah manual.

> Repository ini di-fork dari [ShakirAliSWE/nextjs-crud](https://github.com/ShakirAliSWE/nextjs-crud), dikembangkan menjadi [hamsterprohans/simpleappcrud](https://github.com/hamsterprohans/simpleappcrud) (W8: integrasi MongoDB Atlas), lalu disesuaikan untuk deployment Azure di repository ini (W11).

## Arsitektur

```
Developer ── git push ──▶ GitHub ──▶ GitHub Actions (CI/CD)
                                       │  checkout → setup Node → npm ci
                                       │  → build → buat package → deploy
                                       ▼
┌────────────────────── Resource Group ──────────────────────┐
│  App Service Plan (Linux, B1)                              │
│   └─ Web App (Node.js 22 LTS, Next.js)  ◀── MONGODB_URL    │
│                    │                                       │
│                    ▼                                       │
│        Azure Cosmos DB for MongoDB (RU)                    │
└────────────────────────────────────────────────────────────┘
```

| Komponen | Fungsi |
| --- | --- |
| Resource Group | Wadah yang mengelompokkan semua resource Azure proyek ini |
| App Service Plan + Web App | Hosting aplikasi Next.js (Node.js) |
| Cosmos DB (MongoDB API) | Database, kompatibel dengan Mongoose |
| GitHub Actions | Pipeline CI/CD: build sampai deploy otomatis |
| Publish Profile | Kredensial yang dipakai GitHub Actions untuk login ke Web App |

## Alur Pipeline

Didefinisikan di [.github/workflows/deploy.yml](.github/workflows/deploy.yml):

1. **Checkout code**: mengambil source code dari repository.
2. **Setup Node.js**: memasang Node.js 22.
3. **Install dependencies**: `npm ci`.
4. **Build Next.js**: `npm run build` (menghasilkan `.next/standalone`, sesuai `output: 'standalone'` di [next.config.js](next.config.js)).
5. **Buat deployment package**: menyalin `static` dan `public` ke folder standalone, lalu di-zip menjadi `release.zip`.
6. **Deploy ke Azure**: mengirim `release.zip` ke Web App memakai `azure/webapps-deploy` dan Publish Profile.

## Prasyarat

- Akun GitHub dan Git terpasang
- Akun MyITS (untuk Azure for Students)
- Node.js 22 dan npm (hanya jika ingin mencoba di lokal)
- VS Code

---

## Langkah demi Langkah

### 1. Pra-Implementasi: cek langganan Azure

1. Buka Portal Azure: https://portal.azure.com
2. Login dengan akun MyITS.
3. Cari **Subscriptions** pada search bar di atas.
4. Pastikan langganan **Azure for Students** berstatus **Active**.
5. Jika belum ada, aktivasi di https://azure.microsoft.com/en-us/free/students: klik **Start free**, pilih akun MyITS, lalu isi data pendukung (nama, email, alamat, dll).

### 2. Menyiapkan repository

1. **Fork** repository https://github.com/hamsterprohans/simpleappcrudazure.git melalui tombol **Fork** di GitHub. Beri nama repository sesuai kebutuhan, misalnya `pso-ci-cd-azure`.
2. Buka **VS Code**, lalu **clone repository hasil fork-mu** (bukan milik `hamsterprohans`). Ganti `<username-github-kamu>` dan `<nama-repo>`:

   ```bash
   git clone https://github.com/<username-github-kamu>/<nama-repo>.git
   cd <nama-repo>
   ```

3. Di GitHub, buka tab **Actions** pada repository fork-mu. Jika muncul pesan workflow dinonaktifkan, klik **I understand my workflows, go ahead and enable them**. GitHub menonaktifkan Actions pada fork secara default.

### 3. Membuat infrastruktur di Azure

Semua resource dikumpulkan dalam satu Resource Group agar mudah dipantau dan dihapus sekaligus.

**3.1 Resource Group**

1. Cari **Resource groups** di search bar, lalu klik **Create**.
2. Subscription: **Azure for Students**.
3. Isi nama, misalnya `rg-pso-azure`. Region bebas (contoh: Southeast Asia atau Japan West).
4. Klik **Review + create**, lalu **Create**.

**3.2 Cosmos DB (MongoDB API)**

1. Cari **Azure Cosmos DB**, klik **Create**, lalu pilih **Azure Cosmos DB for MongoDB** dengan model **Request unit (RU) database account**.
2. Isi:
   - Resource Group: `rg-pso-azure`
   - Account Name: nama unik, misalnya `pso-azure-db-<nama>`
   - Capacity mode: **Provisioned throughput** (aktifkan **Free tier** jika tersedia) atau **Serverless**
3. Klik **Review + create**, lalu **Create**. Proses pembuatan memakan waktu beberapa menit.

> Region yang tersedia untuk Azure for Students bisa dibatasi. Jika gagal, coba region lain.

**3.3 App Service (Web App)**

1. Cari **App Services**, klik **Create**, lalu pilih **Web App**.
2. Isi:
   - Resource Group: `rg-pso-azure`
   - Name: nama unik. **Catat nama ini**, akan dipakai di workflow (nama menjadi bagian URL `https://<nama>.azurewebsites.net`).
   - Publish: **Code**
   - Runtime stack: **Node 22 LTS**
   - Operating System: **Linux**
   - Region: sama dengan resource lain
   - Pricing plan: **Basic B1**
3. Klik **Review + create**, lalu **Create**.

### 4. Konfigurasi koneksi

**4.1 Hubungkan Web App ke Cosmos DB**

1. Buka **Azure Cosmos DB**, pilih akun yang baru dibuat.
2. Buka **Settings → Connection strings**, lalu salin **PRIMARY CONNECTION STRING**.
3. Buka **App Service** milikmu → **Settings → Environment variables → App settings → Add**. Tambahkan:

   | Name | Value |
   | --- | --- |
   | `MONGODB_URL` | PRIMARY CONNECTION STRING dari Cosmos DB |
   | `HOSTNAME` | `0.0.0.0` |
   | `SCM_DO_BUILD_DURING_DEPLOYMENT` | `false` |

   Klik **Apply**, lalu **Confirm**.

   > Nama `MONGODB_URL` harus persis sama, karena dibaca di [app/libs/mongoDB.js](app/libs/mongoDB.js). Dengan environment variable, kredensial tidak perlu ditulis langsung di kode.

4. Masih di Web App, buka **Settings → Configuration → Stack settings**, isi **Startup Command** dengan:

   ```
   node server.js
   ```

   Klik **Apply**. Hasil build Next.js standalone dijalankan lewat `server.js`.

**4.2 Ambil Publish Profile**

1. Buka **App Service** → **Overview**.
2. Klik **Download publish profile**.
3. Simpan file `.PublishSettings`, lalu buka dengan text editor dan salin seluruh isinya (`Ctrl+A`, `Ctrl+C`).

   > Jika tombol di-nonaktifkan atau file kosong, buka **Settings → Configuration → General settings** dan aktifkan **SCM Basic Auth Publishing Credentials**, lalu coba lagi.

   > File ini berisi kredensial deployment. **Jangan di-commit** dan jangan dibagikan.

### 5. Menyimpan secrets di GitHub

1. Buka repository fork-mu di GitHub → **Settings → Secrets and variables → Actions**.
2. Klik **New repository secret**, lalu tambahkan dua secret:

   | Name | Value |
   | --- | --- |
   | `MONGODB_URL` | PRIMARY CONNECTION STRING dari Cosmos DB (sama dengan di App Service) |
   | `AZURE_WEBAPP_PUBLISH_PROFILE` | Seluruh isi file `.PublishSettings` |

3. Klik **Add secret** untuk masing-masing.

### 6. Menyesuaikan workflow

Buka [.github/workflows/deploy.yml](.github/workflows/deploy.yml), lalu ganti nilai `AZURE_WEBAPP_NAME` dengan **nama Web App** yang kamu buat di langkah 3.3:

```yaml
env:
  AZURE_WEBAPP_NAME: nama-web-app-kamu
```

### 7. Menjalankan pipeline (deploy)

**Metode A: push ke `main`**

```bash
git add .
git commit -m "First deploy azure"
git push -u origin main
```

**Metode B: jalankan manual dengan GitHub CLI**

```bash
# Install GitHub CLI
winget install --id GitHub.cli     # Windows
brew install gh                    # macOS
sudo apt install gh                # Ubuntu/Debian

# Login lalu jalankan workflow
gh auth login
gh workflow run deploy.yml
```

Pantau prosesnya di tab **Actions** pada GitHub. Semua step harus berstatus hijau.

### 8. Verifikasi

1. Buka `https://<nama-web-app>.azurewebsites.net`. Kunjungan pertama bisa lambat karena aplikasi baru dijalankan.
2. Tambahkan data produk lewat form (**Create**).
3. Pastikan data muncul di daftar produk (**Read**), lalu coba **Update** dan **Delete**.
4. Opsional: di Cosmos DB buka **Data Explorer** untuk melihat collection `products`.
5. Uji CI/CD: ubah sedikit teks di aplikasi, lalu push ke `main`. Perubahan harus muncul di website setelah workflow selesai, tanpa deploy manual.

---

## Troubleshooting

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| Step **Deploy** gagal dengan error autentikasi / publish profile | Secret `AZURE_WEBAPP_PUBLISH_PROFILE` kosong, tidak lengkap, atau Basic Auth nonaktif | Salin ulang seluruh isi `.PublishSettings`, aktifkan SCM Basic Auth di Web App |
| Step **Deploy** gagal: app tidak ditemukan | `AZURE_WEBAPP_NAME` tidak sama dengan nama Web App | Samakan persis (huruf kecil, tanpa `.azurewebsites.net`) |
| Workflow tidak jalan setelah push | Actions dinonaktifkan pada fork | Aktifkan di tab **Actions** (langkah 2.3) |
| Website menampilkan "Application Error" atau tidak mau start | Startup Command belum diisi | Isi `node server.js` (langkah 4.1) lalu restart Web App |
| Halaman terbuka tapi simpan data gagal / loading lama | `MONGODB_URL` salah atau belum di-set di App Service | Periksa App settings, lalu **Restart** Web App |
| `Operation products.find() buffering timed out` | Koneksi ke database gagal | Periksa connection string dan pengaturan **Networking** Cosmos DB (izinkan akses dari semua jaringan atau dari Azure services) |
| `bad auth` / `Authentication failed` | Connection string salah atau key sudah di-regenerate | Salin ulang PRIMARY CONNECTION STRING |

Log aplikasi dapat dilihat di Web App → **Monitoring → Log stream**.

## Keamanan dan biaya

- Jangan pernah commit `.env`, file `.PublishSettings`, atau connection string. Gunakan GitHub Secrets dan App Settings.
- Plan **B1** mengonsumsi kredit Azure for Students selama berjalan. Setelah selesai belajar, **Stop** Web App atau hapus seluruh **Resource Group** untuk menghentikan biaya.

## Teknologi

- Next.js 16
- Mongoose
- Azure App Service, Azure Cosmos DB (MongoDB API)
- GitHub Actions
