# Analisis Pembatalan Pesanan Shopee

> **Tujuan:** Mengidentifikasi penyebab utama pembatalan pesanan pada toko Shopee dan memberikan rekomendasi berbasis data untuk menurunkan cancel rate serta meningkatkan konversi penjualan.

---

## Latar Belakang

Dalam ekosistem e-commerce, pembatalan pesanan merupakan salah satu masalah operasional yang berdampak langsung pada pendapatan, reputasi toko, dan efisiensi logistik. Proyek ini menganalisis dataset transaksi penjualan dari platform Shopee untuk memahami pola pembatalan, siapa yang membatalkan, mengapa terjadi, dan faktor-faktor apa saja yang berkontribusi - agar penjual dapat mengambil langkah yang tepat dan terukur.

---

## Dataset

| Atribut | Keterangan |
|---|---|
| **Sumber** | Export data transaksi Shopee Seller Center (`sales_datasets.csv`) |
| **Isi** | Data pesanan termasuk status, metode pembayaran, opsi pengiriman, provinsi, alasan pembatalan, total bayar, berat, diskon, dan waktu |
| **Format Awal** | Nilai mata uang dalam string Rupiah (`Rp1.000.000`), kategori tidak konsisten (huruf besar/kecil campuran) |

---

## Tahapan Analisis

### 1. Data Cleaning & Preprocessing
- **Trim kolom**: Menghapus spasi berlebih pada nama kolom
- **Parsing Rupiah**: Mengkonversi kolom bertipe string berformat Rupiah ke `float` menggunakan fungsi kustom `parse_rupiah()`
- **Normalisasi Metode Pembayaran**: Menyatukan variasi penulisan (contoh: `"COD (Bayar di Tempat)"`, `"Cash On Delivery"` → `"COD"`) ke dalam 9 kategori bersih menggunakan `payment_map`
- **Normalisasi Opsi Pengiriman**: Menyederhanakan variasi nama layanan kurir (Hemat, Reguler, Same Day, Instant, Next Day, Kargo, GoSend, GrabExpress) menggunakan fungsi `simplify_shipping()`
- **Ekstraksi Aktor & Alasan Pembatalan**: Membuat kolom `Cancel Actor` (Pembeli / Penjual / Sistem) dan `Alasan Simplify` (11 kategori bersih) dari kolom teks bebas `Alasan Pembatalan`
- **Feature Engineering Waktu**: Menambah kolom `order_month`, `order_year`, `order_day`, `order_hour` dari timestamp pesanan

### 2. Identifikasi Masalah
- Menghitung total transaksi, pesanan selesai, dibatalkan, dan status lainnya
- Menghitung **cancel rate** keseluruhan sebagai baseline masalah

### 3. Analisis Eksplorasi (EDA)
- Distribusi status pesanan (bar chart + pie chart)
- Analisis aktor pembatalan (pie chart proporsi Pembeli / Sistem / Penjual)
- Analisis alasan pembatalan menggunakan **Pareto Chart (80/20)** - diagram batang + kurva kumulatif
- Analisis cancel rate per **metode pembayaran**
- Analisis **tren pembatalan bulanan** (stacked bar + line cancel rate)
- Analisis **geografis** pembatalan per provinsi (top 10)
- Distribusi variabel numerik (`total_qty`, `total_weight_gr`, `total_returned_qty`, `Total Diskon`) berdasarkan status pesanan
- **Heatmap korelasi** antar variabel numerik terhadap `is_cancel`

---

## Temuan Utama

### Cancel Rate
Dataset menunjukkan adanya cancel rate yang signifikan dari total keseluruhan transaksi, dengan mayoritas pembatalan dilakukan oleh pihak Pembeli, diikuti oleh Sistem, dan sebagian kecil oleh Penjual.

### Aktor Pembatalan
| Aktor | Proporsi |
|---|---|
| Pembeli | Mayoritas (~dominan) |
| Sistem | Signifikan |
| Penjual | Minoritas |

> Tingginya pembatalan dari sisi **pembeli** mengindikasikan masalah pada kepuasan pra-pembelian, sedangkan pembatalan oleh **sistem** umumnya berkaitan dengan kegagalan pembayaran atau pesanan tidak diproses tepat waktu.

### Alasan Pembatalan (Pareto)
Berdasarkan analisis Pareto, 80% kasus pembatalan disebabkan oleh beberapa alasan teratas, antara lain:

1. **Berubah Pikiran** - pembeli membatalkan sendiri tanpa alasan teknis
2. **Ubah Pesanan** - pembeli ingin memodifikasi pesanan setelah dibuat
3. **Ubah Alamat** - kesalahan input alamat pengiriman
4. **Belum Dibayar** - pesanan dibuat tapi tidak diselesaikan pembayarannya
5. **Penjual Terlambat** - penjual gagal memproses/mengirim dalam batas waktu
6. **Pengiriman Gagal** - paket tidak sampai ke pembeli

### Metode Pembayaran vs Cancel Rate
- **COD (Cash on Delivery)** cenderung memiliki cancel rate lebih tinggi dibanding metode pembayaran digital - karena tidak ada komitmen finansial di muka dari pembeli
- **ShopeePay dan Kartu Kredit/Debit** umumnya memiliki cancel rate lebih rendah karena pembayaran sudah dilakukan di awal

### Tren Bulanan
Tren pembatalan berfluktuasi mengikuti pola volume pesanan - cancel rate cenderung **meningkat pada periode volume tinggi** (contoh: campaign/harbolnas), kemungkinan karena:
- Antrian pesanan yang menumpuk di sisi penjual
- Sistem terlambat memproses pembayaran
- Pembeli impulsif saat flash sale

### Geografis (Top 10 Provinsi)
Provinsi dengan volume pembatalan tertinggi umumnya juga merupakan provinsi dengan volume pesanan tertinggi (Jawa, Jabodetabek). Namun perlu diwaspadai provinsi yang memiliki cancel rate per transaksi di atas rata-rata, yang dapat mengindikasikan masalah logistik lokal atau profil pembeli tertentu.

### Korelasi Variabel Numerik
Heatmap korelasi menunjukkan bahwa variabel seperti `total_returned_qty` memiliki korelasi positif terhadap `is_cancel`, mengindikasikan bahwa pesanan dengan riwayat retur cenderung lebih sering dibatalkan.

---

## Sebab-Akibat

| Sebab | Akibat |
|---|---|
| Pembeli impulsif / tidak komitmen (terutama COD) | Cancel rate tinggi, kerugian ongkos kirim, slot stok terblokir |
| Kesalahan input alamat pengiriman | Pengiriman gagal, pesanan dibatalkan, biaya logistik terbuang |
| Penjual terlambat memproses pesanan | Sistem auto-cancel, penurunan skor toko, pembeli kecewa |
| Pesanan tidak dibayar dalam batas waktu | Sistem otomatis membatalkan, pendapatan tidak terealisasi |
| Volume pesanan melonjak saat campaign | Kapasitas operasional tidak cukup → terlambat kirim → pembatalan |
| Harga produk lebih murah di tempat lain | Pembeli cancel untuk beli di toko lain → kehilangan pelanggan |

---

## Rekomendasi & Saran

### Prioritas Tinggi
1. Kurangi ketergantungan COD - Berikan insentif (voucher cashback, gratis ongkir lebih besar) untuk mendorong pembeli beralih ke ShopeePay atau metode digital lain yang memiliki cancel rate lebih rendah
2. Aktifkan konfirmasi pesanan otomatis - Kirim pesan/notifikasi segera setelah pesanan dibuat untuk mengkonfirmasi dan mengikat komitmen pembeli
3. Percepat waktu proses pengiriman - Pasang SLA internal yang lebih ketat dari ketentuan platform (misal: proses dalam 12 jam, bukan 24 jam) untuk menghindari auto-cancel sistem

### Prioritas Menengah
4. Perbaiki alur pengisian alamat - Sediakan panduan atau validasi input alamat di deskripsi produk atau template pesan otomatis agar pembeli tidak salah input
5. Monitor cancel rate per metode pembayaran - Secara periodik evaluasi metode mana yang paling sering berujung batal dan sesuaikan strategi promosi
6. Siapkan kapasitas ekstra saat campaign - Batasi jumlah stok aktif saat Harbolnas / flash sale jika kapasitas pengiriman tidak mencukupi

### Jangka Panjang
7. Bangun sistem alert tren pembatalan - Pantau cancel rate bulanan secara otomatis; jika melewati ambang batas, trigger investigasi segera
8. Analisis segmentasi pembeli - Identifikasi profil pembeli yang sering membatalkan (berdasarkan provinsi, jam pesan, metode bayar) dan terapkan kebijakan berbeda (misal: nonaktifkan COD untuk akun tertentu)
9. Lacak retur vs pembatalan - Karena `total_returned_qty` berkorelasi dengan pembatalan, pastikan produk yang sering diretur dievaluasi deskripsi dan kualitas packagingnya

---

## Faktor-Faktor yang Harus Diperhatikan

| Faktor | Penjelasan |
|---|---|
| **Metode Pembayaran** | COD secara inheren lebih rawan batal; evaluasi terus distribusinya |
| **Waktu Proses Penjual** | Keterlambatan penjual adalah penyebab yang 100% dapat dikontrol |
| **Volume Campaign** | Lonjakan pesanan tanpa persiapan kapasitas = pembatalan meningkat |
| **Kualitas Data Alamat** | Alamat salah → pengiriman gagal → batal; perlu edukasi pembeli |
| **Musiman / Temporal** | Pola pembatalan bervariasi per bulan, perlu dipantau tren-nya |
| **Geografi** | Beberapa provinsi memiliki cancel rate di atas rata-rata - perlu dicek infrastruktur logistik |
| **Komitmen Pembeli** | Pembeli yang sudah membayar di muka (non-COD) jauh lebih kecil kemungkinan membatalkan |
| **Riwayat Retur** | Produk dengan retur tinggi berpotensi memiliki masalah deskripsi/kualitas yang juga mendorong pembatalan |

---

## Tech Stack

| Tool | Fungsi |
|---|---|
| `Python 3` | Bahasa pemrograman utama |
| `Pandas` | Manipulasi dan analisis data |
| `NumPy` | Operasi numerik |
| `Matplotlib` | Visualisasi data (bar, pie, line, histogram) |
| `Seaborn` | Heatmap korelasi |
| `Jupyter Notebook` | Environment analisis interaktif |

---



## 👤 Author

**[Adji Komara Sandra]**
Data Analyst | Python · SQL · Pandas · Visualization

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue)](https://www.linkedin.com/in/adji-komara-sandra/)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black)](https://github.com/Adjisandra23)

---

*Analisis ini dibuat sebagai bagian dari portofolio data analytics untuk menunjukkan kemampuan dalam data cleaning, exploratory data analysis (EDA), visualisasi, dan penyusunan rekomendasi berbasis data.*