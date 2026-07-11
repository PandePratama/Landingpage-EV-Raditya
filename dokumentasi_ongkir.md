# Dokumentasi Implementasi Skema Ongkir (Adaptasi Maxim Cargo)

Catatan ini berisi penjelasan teknis dan logika matematika dari skema ongkos kirim (ongkir) yang telah disesuaikan agar sejalan dengan estimasi dari aplikasi Maxim Cargo. Dokumentasi ini dapat digunakan sebagai referensi untuk mengimplementasikan fitur kalkulator ongkir yang sama di proyek-proyek (website/aplikasi) lainnya.

---

## 1. Konsep Dasar & Arsitektur

Karena sistem website / *client-side* biasanya tidak memiliki batasan poligon wilayah (geofencing) internal layaknya aplikasi logistik besar, kita menggunakan **Reverse Geocoding** untuk menentukan zona pengiriman secara dinamis.

* **API Routing Jarak:** Menggunakan **OSRM API** (gratis) untuk menghitung jarak tempuh aktual (berkendara) antara titik awal dan titik tujuan.
* **API Geofencing (Deteksi Kota):** Menggunakan **Nominatim OpenStreetMap API** (gratis) untuk mengubah kordinat (Lat/Lng) tujuan menjadi alamat teks, lalu mendeteksi apakah lokasi tersebut berada di wilayah "Dalam Kota" atau "Antarkota".

---

## 2. Logika Geofencing (Dalam Kota vs Antarkota)

Penentuan skema tarif didasarkan pada nama kota/kabupaten dari titik tujuan.

```javascript
// Contoh deteksi Dalam Kota menggunakan Nominatim API
let isDalamKota = true;
try {
    const geoResponse = await fetch(`https://nominatim.openstreetmap.org/reverse?format=json&lat=${latlng.lat}&lon=${latlng.lng}`);
    const geoData = await geoResponse.json();
    
    const addressStr = geoData.display_name ? geoData.display_name.toLowerCase() : "";
    
    // Sesuaikan kata kunci dengan kota asal pengiriman Anda. 
    // Di kasus ini, titik awal ada di Denpasar.
    isDalamKota = addressStr.includes('denpasar');
} catch (error) {
    // Fallback: Jika API error, gunakan batasan jarak (misal <= 20km = Dalam Kota)
    isDalamKota = currentDistance <= 20;
}
```

---

## 3. Rumus Kalkulasi Biaya (Matematika)

Rumus di bawah ini sudah melalui proses kalibrasi (penambahan dan pengurangan *baseline*) agar toleransi selisihnya terhadap aplikasi aslinya berada di batas wajar (± Rp 3.000 akibat perbedaan tarikan garis rute di peta).

> **Penting:** Jarak (`currentDistance`) tidak dibulatkan ke atas (`Math.ceil`), melainkan menggunakan angka desimal murni (proporsional per meter) agar seakurat aplikasi.

### A. Komponen Biaya Wajib
Terdapat biaya yang selalu ditambahkan ke total akhir di setiap transaksi. Berdasarkan spesifikasi *project* ini, kita menggabungkan **Platform Fee** (Rp 6.000) dan **Biaya Bongkar/Muat** (Rp 50.000) sebagai biaya tambahan wajib.
* `BIAYA_TAMBAHAN = 56000`

### B. Tarif Dalam Kota
* **Base Tarif (0 - 5 km):** Rp 92.000 *(sudah dikalibrasi)*
* **Tarif Ekstra (> 5 km):** Rp 9.500 per kilometer selanjutnya.

```javascript
const BASE_DALAM_KOTA = 92000;
let biayaPerjalanan = 0;

if (currentDistance <= 5) {
    biayaPerjalanan = BASE_DALAM_KOTA;
} else {
    const extraKm = currentDistance - 5; 
    biayaPerjalanan = BASE_DALAM_KOTA + (extraKm * 9500);
}
```

### C. Tarif Antarkota (Luar Kota)
* **Base Tarif (0 - 6 km):** Rp 117.000 *(sudah dikalibrasi turun 15rb agar fit dengan aplikasi)*
* **Tarif Ekstra 1 (6 km - 20 km):** Rp 8.000 per kilometer.
* **Tarif Ekstra 2 (> 20 km):** Rp 5.500 per kilometer.

```javascript
const BASE_ANTARKOTA = 117000;
let biayaPerjalanan = 0;

if (currentDistance <= 6) {
    biayaPerjalanan = BASE_ANTARKOTA;
} else if (currentDistance <= 20) {
    const extraKm = currentDistance - 6;
    biayaPerjalanan = BASE_ANTARKOTA + (extraKm * 8000);
} else {
    const kmTier1 = 14; // Selisih jarak dari 6 km ke 20 km
    const extraKmTier2 = currentDistance - 20;
    biayaPerjalanan = BASE_ANTARKOTA + (kmTier1 * 8000) + (extraKmTier2 * 5500);
}
```

### D. Harga Akhir (Grand Total)
Menjumlahkan hasil `biayaPerjalanan` dengan `BIAYA_TAMBAHAN` dan dibulatkan.

```javascript
let currentOngkir = Math.round(biayaPerjalanan + BIAYA_TAMBAHAN);
```

---

## 4. Interaktivitas Peta (Draggable Marker)

Untuk memberikan pengalaman pengguna (UX) yang lebih baik, titik tujuan (pin lokasi) pada peta dikonfigurasi agar bisa digeser (*draggable*) secara manual. Hal ini sangat berguna jika deteksi lokasi otomatis dari browser pengguna kurang akurat.

**Implementasi Kode (Leaflet.js):**
```javascript
// 1. Buat marker dengan opsi draggable: true
let destMarker = L.marker([LAT, LNG], { draggable: true }).addTo(mapInstance);

// 2. Berikan tooltip agar pengguna sadar pin bisa digeser
destMarker.bindTooltip('Geser pin ini ke lokasi Anda', { 
    permanent: true, 
    direction: 'top' 
});

// 3. Tambahkan event listener saat pin selesai digeser ('dragend')
destMarker.on('dragend', function() {
    // Ambil kordinat terbaru setelah pin dilepas
    const newPos = destMarker.getLatLng();
    
    // Posisikan ulang kamera peta ke kordinat baru
    mapInstance.setView(newPos, mapInstance.getZoom());
    
    // Panggil kembali fungsi kalkulasi ongkir untuk mengupdate harga dan rute
    calculateOngkir();
});
```

---

## 5. Kesimpulan & *Best Practice*
- Selisih estimasi (sekitar Rp 1.000 - Rp 3.000) sangat wajar terjadi karena sistem *routing* API gratis (OSRM) terkadang memberikan rute alternatif yang lebih pendek atau lebih panjang beberapa ratus meter dibandingkan algoritma eksklusif milik Google Maps / Maxim.
- Selalu pisahkan variabel `BASE_DALAM_KOTA` dan `BASE_ANTARKOTA` saat membangun di *project* lain agar memudahkan penyesuaian (*tuning*) jika sewaktu-waktu harga dasar dari layanan logistik berubah.
