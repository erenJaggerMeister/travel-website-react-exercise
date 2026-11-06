# Panduan Arsitektur & Langkah Pembuatan Travel Agency Website (Microfrontend + React.js + Vite)

Dokumen ini berisi panduan komprehensif, analisis fitur referensi ([Nordic Visitor - Norway](https://www.nordicvisitor.com/norway/)), arsitektur sistem, serta langkah-langkah implementasi (*step-by-step roadmap*) menggunakan **React.js**, **Vite**, **Microfrontend Architecture (Module Federation)**, dan prinsip **Separation of Concerns (SoC)**.

---

## 1. Analisis Fitur Referensi (Nordic Visitor)

Website [Nordic Visitor Norway](https://www.nordicvisitor.com/norway/) memiliki karakteristik platform travel e-commerce premium dengan fokus pada paket wisata terkurasi (*tailor-made* & *guided tours*). Fitur-fitur inti yang perlu diadopsi antara lain:

1. **Header & Global Controls:**
   - Multi-currency switcher (USD, EUR, GBP, dll.) dan penyesuaian harga dinamis.
   - Menu kategori berdasarkan *Travel Style* (Self-drive, Scenic Rail, Small Group Tours, Fjord Cruises, Northern Lights).
   - Quick inquiry / "Wishlist" & tombol kontak darurat 24/7.
2. **Hero Section & Quick Tour Finder:**
   - Visual visual lanskap fjord berkualitas tinggi dengan headline yang menjual *experience*.
   - Filter cepat terintegrasi: Destinasi, Bulan/Musim keberangkatan, Durasi hari, dan Tipe perjalanan.
3. **Value Proposition & Trust Signals:**
   - Keunggulan agensi ("Mengapa memilih kami": konsultan lokal berdedikasi, itinerary terverifikasi, review TripAdvisor/Trustpilot).
4. **Tour Catalog & Filtering Engine:**
   - Card tour informatif: label durasi, rute highlights, badge (e.g. *Best Seller*, *Summer Only*), rating, dan harga *“From $X”*.
   - Filter mendalam (rentang budget, tingkat kenyamanan, moda transportasi).
5. **Interactive Tour Detail Page (PDP):**
   - Ringkasan rute harian (*Day-by-Day Interactive Itinerary*) yang sinkron dengan peta interaktif (Leaflet / Mapbox).
   - Daftar fasilitas: *What's included* (hotel, sewa mobil, tiket kapal) vs *What's excluded*.
   - Sistem kustomisasi paket (opsi upgrade hotel, penambahan aktivitas opsional seperti glacier hike, fjord cruise).
6. **Booking & Inquiry Wizard:**
   - Kalender musim & ketersediaan tanggal.
   - Formulir kalkulasi harga otomatis berdasarkan jumlah penumpang dan add-on.

---

## 2. Arsitektur Microfrontend (MFE)

Menggunakan pendekatan **Monorepo** (didukung oleh `pnpm workspaces` atau `Turborepo`) dan **Module Federation** via `@originjs/vite-plugin-federation`.

```
                    ┌──────────────────────────────────────────┐
                    │          Host / Shell Application        │
                    │  (Port 3000: Router, Header, Footer,     │
                    │   Global Context, Error Boundaries)      │
                    └──────┬─────────────┬─────────────┬───────┘
                           │             │             │
              ┌────────────┘             │             └────────────┐
              ▼                          ▼                          ▼
┌──────────────────────────┐ ┌───────────────────────┐ ┌──────────────────────────┐
│  MFE: Catalog & Search   │ │  MFE: Tour Detail &   │ │   MFE: Booking Wizard    │
│       (Port 3001)        │ │  Itinerary (Port 3002)│ │       (Port 3003)        │
│  - Tour Finder / Filter  │ │  - Day-by-day Timeline│ │  - Price Calculator      │
│  - Tour Grid & Cards     │ │  - Interactive Map    │ │  - Add-ons Selector      │
│  - Travel Styles Browse  │ │  - Inclusions/Exclusions│ - Inquiry/Checkout Form  │
└─────────────┬────────────┘ └───────────┬───────────┘ └────────────┬─────────────┘
              │                          │                          │
              └──────────────────────────┼──────────────────────────┘
                                         ▼
                    ┌──────────────────────────────────────────┐
                    │   Shared Core & UI Package (Workspace)   │
                    │  - UI Design System (Tailwind/Tokens)    │
                    │  - Event Bus / Nano Stores (State Sync)  │
                    │  - API Client / Types / Formatting Utils │
                    └──────────────────────────────────────────┘
```

### Pembagian Modul Microfrontend:
1. **`host-app` (Shell / Container):**
   - Bertanggung jawab atas routing level teratas (React Router v6).
   - Global layout (Header, Navigasi, Footer).
   - Global contexts (Tema, Preferensi Mata Uang, Wishlist).
   - Host orchestrator untuk mengimpor remote module secara asinkron (React `Suspense` + `ErrorBoundary`).
2. **`catalog-mfe` (Remote: Discovery & Catalog):**
   - Halaman katalog tur, search bar interaktif, filter drawer/sidebar.
   - Komponen Tour Card yang bisa diekspos juga ke Host untuk ditaruh di homepage.
3. **`tour-detail-mfe` (Remote: Itinerary & Map Detail):**
   - Halaman detail tur lengkap: Hero galeri, itinerary interaktif per hari yang terhubung dengan visualisasi peta rute.
   - Komponen FAQ dan tab rincian akomodasi.
4. **`booking-mfe` (Remote: Customizer & Checkout):**
   - Modul kalkulasi harga dinamis (kamar single/double, opsi mobil transmisi manual/matic, tur tambahan).
   - Step-by-step booking/inquiry flow.
5. **`shared-ui` & `shared-core` (Shared Internal Packages):**
   - Desain komponen reusable: Button, Modal, Card, Badge, Drawer, Form inputs.
   - Helper format uang (`formatCurrency`), tanggal, validasi skema (Zod), dan types TypeScript bersama.

---

## 3. Penerapan Separation of Concerns (SoC)

Setiap microfrontend menerapkan pemisahan tanggung jawab yang ketat ke dalam 4 lapisan (*layer*):

```
mfe-module/
├── src/
│   ├── api/          # 1. DATA ACCESS LAYER: Fetcher, axios client, DTO transformation
│   ├── domain/       # 2. DOMAIN / TYPES LAYER: Pure TypeScript interfaces, entities, validation
│   ├── hooks/        # 3. LOGIC / STATE LAYER: Custom hooks, state management (Zustand/React Query)
│   ├── components/   # 4. PRESENTATION LAYER: Dumb/Presentational UI components
│   │   ├── atoms/
│   │   └── molecules/
│   └── views/        # Page-level smart components yang menggabungkan presentation + hooks
```

- **Presentation Layer:** Murni bertanggung jawab me-render UI dan menangani interaksi pengguna. Tidak melakukan `fetch()` langsung dan tidak menyimpan *business logic*.
- **Logic & State Layer:** Ditampung di React Custom Hooks (misal: `useTourFilter`, `useItineraryMap`, `usePriceCalculator`). Mengelola *state*, memoization, dan memanggil service API.
- **Data Access Layer:** Service murni JavaScript/TypeScript untuk request HTTP ke API backend, memetakan DTO (*Data Transfer Object*) ke domain model, dan penanganan error jaringan.
- **Domain & Types Layer:** Menjamin konsistensi kontrak data antar modul MFE tanpa ketergantungan pada UI framework.

---

## 4. Langkah-Langkah Implementasi & Perintah Setup (Step-by-Step Roadmap)

### Fase 1: Setup Workspace & Command Instalasi Proyek

Berikut adalah perintah CLI lengkap yang dapat disalin dan dijalankan untuk menyiapkan struktur monorepo dari awal.

#### 1. Prasyarat
Pastikan Node.js (versi >= 18) dan `pnpm` sudah terpasang di komputer Anda.
```bash
# Cek versi Node.js & pnpm
node -v
pnpm -v

# Jika pnpm belum terpasang:
npm install -g pnpm
```

#### 2. Inisialisasi Root Monorepo & Konfigurasi Workspace
Jalankan perintah ini di direktori root proyek (`travel-website-react-exercise`):

```bash
# Inisialisasi package.json root
pnpm init

# Buat struktur direktori untuk apps dan packages
mkdir apps
mkdir packages
mkdir packages/shared-ui
mkdir packages/shared-core
```

Buat file `pnpm-workspace.yaml` di root proyek:
```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

#### 3. Inisialisasi Aplikasi Vite (React + TypeScript) di Folder `apps/`
Gunakan `pnpm create vite` dengan template `react-ts` untuk masing-masing modul MFE:

```bash
# Pindah ke folder apps
cd apps

# Buat Host (Shell) App
pnpm create vite host --template react-ts

# Buat Remote Catalog App
pnpm create vite catalog --template react-ts

# Buat Remote Tour Detail App
pnpm create vite tour-detail --template react-ts

# Buat Remote Booking App
pnpm create vite booking --template react-ts

# Kembali ke root direktori
cd ..
```

#### 4. Setup Packages Bersama (`packages/shared-core` & `packages/shared-ui`)
Inisialisasi `package.json` pada masing-masing shared package:

```bash
# Inisialisasi shared-core (types, utils, event bus)
cd packages/shared-core
pnpm init
# Ubah name di packages/shared-core/package.json menjadi "@travel/shared-core"

# Inisialisasi shared-ui (desain sistem bersama, Tailwind primitives)
cd ../shared-ui
pnpm init
# Ubah name di packages/shared-ui/package.json menjadi "@travel/shared-ui"

# Kembali ke root
cd ../..
```

#### 5. Instalasi Dependensi di Tingkat Root (Dev Tooling & Orchestration)
Untuk menjalankan semua microfrontend secara bersamaan (*concurrently*):

```bash
pnpm add -D -w concurrently
```

Tambahkan scripts berikut di `package.json` root untuk mempermudah eksekusi:
```json
{
  "name": "travel-website-react-exercise",
  "private": true,
  "scripts": {
    "install:all": "pnpm install",
    "dev:host": "pnpm --filter host dev --port 3000",
    "dev:catalog": "pnpm --filter catalog dev --port 3001",
    "dev:tour-detail": "pnpm --filter tour-detail dev --port 3002",
    "dev:booking": "pnpm --filter booking dev --port 3003",
    "dev": "concurrently -k -p \"[{name}]\" -n \"HOST,CATALOG,TOUR,BOOKING\" -c \"cyan,green,yellow,magenta\" \"pnpm dev:host\" \"pnpm dev:catalog\" \"pnpm dev:tour-detail\" \"pnpm dev:booking\"",
    "build": "pnpm -r run build"
  }
}
```

---

#### 6. Instalasi Dependensi Spesifik untuk Setiap Modul MFE

Gunakan opsi `--filter` dari `pnpm` untuk menginstal dependensi ke masing-masing proyek secara terisolasi:

##### A. Plugin Module Federation (Wajib di semua modul)
```bash
# Instal plugin federation di Host dan semua Remotes
pnpm --filter host add -D @originjs/vite-plugin-federation
pnpm --filter catalog add -D @originjs/vite-plugin-federation
pnpm --filter tour-detail add -D @originjs/vite-plugin-federation
pnpm --filter booking add -D @originjs/vite-plugin-federation
```

##### B. Dependensi untuk `apps/host` (Shell App)
```bash
# Router, Icons, dan State Listener
pnpm --filter host add react-router-dom lucide-react nanostores @nanostores/react
# Hubungkan dengan local shared packages
pnpm --filter host add @travel/shared-ui@workspace:* @travel/shared-core@workspace:*
```

##### C. Dependensi untuk `apps/catalog` (Discovery & Search MFE)
```bash
# Data fetching, Icons, Filter state
pnpm --filter catalog add @tanstack/react-query lucide-react
# Hubungkan dengan local shared packages
pnpm --filter catalog add @travel/shared-ui@workspace:* @travel/shared-core@workspace:*
```

##### D. Dependensi untuk `apps/tour-detail` (Itinerary & Interactive Map MFE)
```bash
# Peta interaktif Leaflet + Types, Accordion/Tabs icons
pnpm --filter tour-detail add leaflet react-leaflet lucide-react
pnpm --filter tour-detail add -D @types/leaflet
# Hubungkan dengan local shared packages
pnpm --filter tour-detail add @travel/shared-ui@workspace:* @travel/shared-core@workspace:*
```

##### E. Dependensi untuk `apps/booking` (Customizer & Inquiry Form MFE)
```bash
# Form engine, Schema validator Zod, Icons
pnpm --filter booking add react-hook-form zod @hookform/resolvers lucide-react
# Hubungkan dengan local shared packages
pnpm --filter booking add @travel/shared-ui@workspace:* @travel/shared-core@workspace:*
```

##### F. Dependensi untuk `packages/shared-ui` & `packages/shared-core`
```bash
# Tailwind CSS, utility classes
pnpm --filter @travel/shared-ui add -D tailwindcss postcss autoprefixer
pnpm --filter @travel/shared-ui add clsx tailwind-merge

# Event bus ringan & cross-app state
pnpm --filter @travel/shared-core add nanostores @nanostores/react
```

##### G. Inisialisasi Tailwind CSS
Jalankan inisialisasi Tailwind di `packages/shared-ui`:
```bash
cd packages/shared-ui
npx tailwindcss init -p
cd ../..
```

Setelah itu, jalankan `pnpm install` dari root untuk memastikan symlink workspace terhubung sempurna:
```bash
pnpm install
```

---

### Fase 2: Pembangunan Shared Core & Design System (`packages/*`)
1. **Definisikan Domain Model (`packages/shared-core/types`):**
   - Model `Tour`, `ItineraryDay`, `PriceTier`, `Currency`, `BookingRequest`.
2. **Buat Event Bus / Cross-MFE Communication:**
   - Implementasikan mekanika komunikasi ringan antar MFE (misal menggunakan Custom DOM Events atau `nanostores`) untuk event seperti:
     - `CURRENCY_CHANGED`
     - `ADD_TO_WISHLIST`
     - `INITIATE_BOOKING`
3. **Buat UI Primitives (`packages/shared-ui`):**
   - Buat komponen atomik: `Button`, `Badge`, `RatingStars`, `Modal`, `Accordion`, `Input`, `Select`.
   - Pastikan komponen bersifat agnostik terhadap business logic.

---

### Fase 3: Konfigurasi Module Federation dengan Vite
1. **Instal `@originjs/vite-plugin-federation`:**
   - Tambahkan plugin di setiap `vite.config.ts`.
2. **Konfigurasi Remote (`apps/catalog`, `apps/tour-detail`, `apps/booking`):**
   - Ekspos komponen utama, contoh pada `apps/catalog/vite.config.ts`:
     ```ts
     federation({
       name: 'catalog',
       filename: 'remoteEntry.js',
       exposes: {
         './TourCatalogView': './src/views/TourCatalogView.tsx',
         './TourCard': './src/components/TourCard.tsx',
         './QuickSearchFilter': './src/components/QuickSearchFilter.tsx'
       },
       shared: ['react', 'react-dom', 'lucide-react']
     })
     ```
3. **Konfigurasi Host (`apps/host`):**
   - Tentukan daftar remotes di `apps/host/vite.config.ts`:
     ```ts
     federation({
       name: 'host',
       remotes: {
         catalogApp: 'http://localhost:3001/assets/remoteEntry.js',
         tourDetailApp: 'http://localhost:3002/assets/remoteEntry.js',
         bookingApp: 'http://localhost:3003/assets/remoteEntry.js'
       },
       shared: ['react', 'react-dom']
     })
     ```

---

### Fase 4: Pembangunan Host / Shell App (`apps/host`)
1. **Global Layout & Navigation:**
   - Buat Top Bar (Currency switcher, Contact number, Help center).
   - Buat Main Navigation (Mega menu: *Destinations*, *Travel Styles*, *Special Offers*, *About Norway*).
   - Buat Footer informatif (Sertifikasi travel, garansi perlindungan konsumen, newsletter).
2. **Routing & Dynamic Remote Loading:**
   - Pasang React Router v6.
   - Buat *wrapper* loader dinamis dengan React `lazy()` dan `Suspense` yang menampilkan skeleton loading saat remote MFE dimuat.
   - Bungkus setiap remote dengan `ErrorBoundary` terisolasi agar kegagalan salah satu MFE tidak merusak seluruh halaman.
3. **Landing Page Composition:**
   - Gabungkan Hero Banner milik Host dengan komponen federated `./QuickSearchFilter` dari `catalogApp`.

---

### Fase 5: Pembangunan Catalog & Search MFE (`apps/catalog`)
1. **Data Layer & Mock API:**
   - Siapkan mock data paket wisata khas Norwegia (e.g. *Norway in a Nutshell*, *Fjord Highlights Self-Drive*, *Northern Lights Adventure*).
   - Buat service API dengan React Query / TanStack Query untuk caching dan pemuatan data yang cepat.
2. **Logic Layer (Custom Hook):**
   - Buat `useTourFilter` untuk menangani filtering multi-kriteria: durasi (3-7 hari, 8-14 hari), tipe tour, rentang budget, dan musim (Summer/Winter).
3. **Presentation Layer:**
   - Grid layout responsif untuk menampilkan Tour Cards.
   - Badge status (e.g. "Self Drive", "Popular", "Customizable").
   - Filter drawer untuk layar mobile dan sidebar filter untuk desktop.

---

### Fase 6: Pembangunan Tour Detail & Itinerary MFE (`apps/tour-detail`)
1. **Hero & Highlight Summary:**
   - Menampilkan ringkasan durasi, tingkat kesulitan, akomodasi, dan rute utama.
2. **Interactive Day-by-Day Itinerary:**
   - Accordion interaktif untuk rincian hari 1, hari 2, dst.
   - Integrasi peta rute (Leaflet / MapLibre) di mana mengklik hari tertentu akan mengarahkan fokus peta ke koordinat lokasi hari tersebut (contoh: Oslo -> Flåm -> Bergen).
3. **Included vs Excluded List:**
   - Komponen visual checklist hijau (included) dan tanda minus abu-abu (excluded) persis seperti standar Nordic Visitor.
4. **Tombol CTA "Book / Inquire This Tour":**
   - Memicu event atau navigasi ke Booking MFE dengan membawa parameter `tourId`.

---

### Fase 7: Pembangunan Customizer & Booking MFE (`apps/booking`)
1. **Engine Kalkulator Harga:**
   - Logika penyesuaian harga berdasarkan musim (peak summer vs off-peak winter).
   - Opsi pemilihan kamar (Single room supplement, Twin/Double, Family).
   - Opsi penambahan aktivitas (*Extra activities*): Kayaking di Nærøyfjord, Glacier Hiking di Nigardsbreen, Fjord Safari boat.
2. **Multi-Step Inquiry Wizard:**
   - Langkah 1: Pilih tanggal & jumlah peserta.
   - Langkah 2: Kustomisasi paket & aktivitas ekstra.
   - Langkah 3: Data diri, preferensi khusus, dan submission inquiry/reservasi.

---

### Fase 8: Komunikasi Lintas Modul & Pengujian
1. **Sinkronisasi Mata Uang:**
   - Uji perubahan mata uang di Host Header (misal USD ke EUR) dan pastikan harga di `catalog-mfe`, `tour-detail-mfe`, dan `booking-mfe` ter-update seketika.
2. **Uji Isolasi Kegagalan (Resilience Testing):**
   - Matikan server `booking-mfe` dan verifikasi bahwa `catalog-mfe` dan `tour-detail-mfe` tetap dapat diakses oleh user tanpa layar putih (*white screen*).
3. **Optimasi Asset & Caching:**
   - Konfigurasi cache header untuk file `remoteEntry.js` (no-cache / short TTL) vs hashed assets (long TTL).

---

## 5. Rekomendasi Library & Stack Pendukung

| Kebutuhan | Rekomendasi Library | Alasan |
| :--- | :--- | :--- |
| **MFE Bundler** | `@originjs/vite-plugin-federation` | Kompatibilitas native dengan Vite, build super cepat |
| **Routing** | `react-router-dom` v6 | Standar industri untuk SPA dan navigasi Host |
| **Data Fetching & Cache** | `@tanstack/react-query` | Manajemen cache server-state, retry, dan loading states |
| **Interactive Map** | `leaflet` + `react-leaflet` | Ringan, open-source, mudah di-customize untuk rute tur |
| **Cross-MFE State** | `nanostores` atau Custom Events | Berukuran ultra-kecil (<1KB), agnostik terhadap framework |
| **Styling & UI Tokens** | `tailwindcss` + `clsx` + `tailwind-merge` | Utility-first, konsistensi tema antar repositori |
| **Icons** | `lucide-react` | Koleksi icon clean dan modern cocok untuk travel UI |
| **Form & Validasi** | `react-hook-form` + `zod` | Performa form tinggi untuk booking wizard kompleks |
