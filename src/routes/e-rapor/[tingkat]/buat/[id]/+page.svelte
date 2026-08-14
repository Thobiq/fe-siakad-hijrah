<script lang="ts">
  import Sidebar from '$lib/components/Sidebar.svelte';
  import PhotoUploader from '$lib/components/PhotoUploader.svelte';
  import { onMount } from 'svelte';
  import { page } from '$app/stores';
  import { PUBLIC_API_URL } from '$env/static/public';
  import { goto } from '$app/navigation';

  let isSidebarOpen = false;
  let userName = "Bu Hijrah";
  let isLoading = true;

  
  // --- STATE DATA DIRI ---
  let dataDiri = {
    namaSekolah: "TK Al-Hijrah",
    namaSiswa: "",
    nis: "",
    semester: "Genap",
    tahunAjaran: "",
    guruKelas: userName,
    kelas: "",
    fase: "Fondasi", // Default PAUD/TK Fase Fondasi
    tingkat: "",
    siswaId: null,
    tinggiBadan: "",
    beratBadan: "",
    lingkarKepala: "",
    tanggalTtd: ""
  };

  $: tingkatRapor = $page.params.tingkat;
  $: isKB = tingkatRapor === 'kb';
  $: {
    if (dataDiri) {
      dataDiri.namaSekolah = isKB ? "KB Al-Hijrah" : "TK Al-Hijrah";
    }
  }
  
  // --- STATE NARASI ---
  let narasiAgama = "";
  let narasiJatiDiri = "";
  let narasiLiterasi = "";
  let narasiKokurikuler = "";
  let refleksiGuru = "";

  // --- STATE LOADING AI ---
  let isGenerating = {
    agama: false,
    jatiDiri: false,
    literasi: false,
    kokurikuler: false
  };


  // --- STATE KHUSUS KB ---
  let pembiasaanAgama = [
    { id: 1, materi: "", capaian: "" }
  ];
  let catatanBuGuru = "";
  let ketidakhadiran = { sakit: "", ijin: "", tanpaKeterangan: "" };

  function addMateri() {
    pembiasaanAgama = [...pembiasaanAgama, { id: Date.now(), materi: "", capaian: "" }];
  }

  function removeMateri(id) {
    pembiasaanAgama = pembiasaanAgama.filter(m => m.id !== id);
  }

  // --- FUNGSI CALL API GEMINI ---
  async function handleGenerateAI(kategori) {
    let poinSingkat = "";
    
    // 1. Ambil poin yang diketik guru berdasarkan kategori
    if (kategori === 'Agama') poinSingkat = narasiAgama;
    else if (kategori === 'Jati Diri') poinSingkat = narasiJatiDiri;
    else if (kategori === 'Literasi') poinSingkat = narasiLiterasi;
    else if (kategori === 'Kokurikuler') poinSingkat = narasiKokurikuler;

    if (!poinSingkat || poinSingkat.trim() === "") {
      alert(`Silakan ketik poin-poin singkat untuk kategori ${kategori} terlebih dahulu di kotak teks.`);
      return;
    }

    isGenerating[kategori] = true;
    const token = localStorage.getItem('auth_token');

    try {
      const response = await fetch(`${PUBLIC_API_URL}/ai/generate-narasi`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
          nama_siswa: dataDiri.namaSiswa,
          kategori: kategori,
          poin_guru: poinSingkat
        })
      });

      const result = await response.json();

      if (response.ok && result.status) {
        // 2. Timpa isi textarea dengan teks utuh dari AI
        if (kategori === 'Agama') narasiAgama = result.data;
        else if (kategori === 'Jati Diri') narasiJatiDiri = result.data;
        else if (kategori === 'Literasi') narasiLiterasi = result.data;
        else if (kategori === 'Kokurikuler') narasiKokurikuler = result.data;
      } else {
        alert("Gagal membuat narasi: " + result.message);
      }
    } catch (error) {
      console.error(error);
      alert("Terjadi kesalahan jaringan saat menghubungi AI.");
    } finally {
      isGenerating[kategori] = false;
    }
  }

  // --- STATE FOTO ---
  let fotoAgama = [{id:1, file:null, preview:null}, {id:2, file:null, preview:null}, {id:3, file:null, preview:null}];
  let fotoJatiDiri = [{id:1, file:null, preview:null}, {id:2, file:null, preview:null}, {id:3, file:null, preview:null}];
  let fotoLiterasi = [{id:1, file:null, preview:null}, {id:2, file:null, preview:null}, {id:3, file:null, preview:null}];
  let fotoKokurikuler = [{id:1, file:null, preview:null}, {id:2, file:null, preview:null}, {id:3, file:null, preview:null}];

  onMount(async () => {
    const token = localStorage.getItem('auth_token');
    const userData = JSON.parse(localStorage.getItem('user_data') || '{}');
    if (userData.name) {
      userName = userData.name;
      dataDiri.guruKelas = userName;
    }

    const penilaianId = $page.params.id;

    try {
      const response = await fetch(`${PUBLIC_API_URL}/penilaian/${penilaianId}`, {
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      const result = await response.json();
      
      if (response.ok && result.status) {
        const p = result.data.penilaian;
        const s = p.siswa;
        
        dataDiri.namaSiswa = s?.nama || "-";
        dataDiri.siswaId = s?.id || null;
        dataDiri.nis = s?.nomor_induk || "-";
        dataDiri.tahunAjaran = p?.tahun_ajaran || "-";
        dataDiri.tingkat = s?.tingkat || "-";
        dataDiri.kelas = s?.kelas?.nama_kelas || "-";
        
        // Fase otomatis
        if (dataDiri.tingkat.includes("TK")) {
          dataDiri.fase = "Fondasi";
        }

        // AUTO-FILL JIKA RAPOR SUDAH ADA
        if (s?.rapor) {
          const r = s.rapor;
          
          // Data Diri
          dataDiri.semester = r.semester || "Genap";
          dataDiri.tinggiBadan = r.tinggi_badan || "";
          dataDiri.beratBadan = r.berat_badan || "";
          dataDiri.lingkarKepala = r.lingkar_kepala || "";

          // Narasi
          narasiAgama = r.narasi_agama || "";
          narasiJatiDiri = r.narasi_jati_diri || "";
          narasiLiterasi = r.narasi_literasi || "";
          narasiKokurikuler = r.narasi_kokurikuler || "";
          
          // Absensi & Refleksi
          ketidakhadiran.sakit = r.sakit || "";
          ketidakhadiran.ijin = r.ijin || "";
          ketidakhadiran.tanpaKeterangan = r.tanpa_keterangan || "";
          refleksiGuru = r.refleksi_guru || "";

          // Khusus KB
          if (r.pembiasaan_agama) {
            pembiasaanAgama = typeof r.pembiasaan_agama === 'string' ? JSON.parse(r.pembiasaan_agama) : r.pembiasaan_agama;
          }
          catatanBuGuru = r.catatan_bu_guru || "";

          // Foto Helper (Menampilkan URL Foto dari Backend)
          const fillPhotos = (stateArray, savedPaths) => {
            if (!savedPaths || !Array.isArray(savedPaths)) return;
            savedPaths.forEach((path, i) => {
              if (i < 3) {
                stateArray[i].preview = PUBLIC_API_URL.replace('/api', '') + path;
              }
            });
          };

          // Karena field foto tersimpan sebagai array (URL string)
          fillPhotos(fotoAgama, r.foto_agama);
          fillPhotos(fotoJatiDiri, r.foto_jati_diri);
          fillPhotos(fotoLiterasi, r.foto_literasi);
          fillPhotos(fotoKokurikuler, r.foto_kokurikuler);
        }
      }
    } catch (error) {
      console.error("Gagal mengambil data Penilaian:", error);
    } finally {
      isLoading = false;
    }
  });

  // Action Svelte untuk Auto-resize Textarea
  function autoResize(node, value) {
    function resize() {
      node.style.height = 'auto';
      node.style.height = node.scrollHeight + 'px';
    }
    node.addEventListener('input', resize);
    setTimeout(resize, 0); // initial resize
    return {
      update() {
        // dipanggil saat parameter 'value' berubah, misal diisi oleh AI
        setTimeout(resize, 0);
      },
      destroy() {
        node.removeEventListener('input', resize);
      }
    }
  }

  // Handle Simpan ke Backend
  async function handleSimpan() {
    console.log("Menyimpan Rapor...");
    
    if (!dataDiri.siswaId) {
      alert("ID Siswa tidak ditemukan. Pastikan data siswa sudah termuat sempurna.");
      return;
    }

    const formData = new FormData();

    // 1. Data Diri & Parameter Utama
    formData.append('siswa_id', dataDiri.siswaId);
    formData.append('tingkat', dataDiri.tingkat);
    formData.append('semester', dataDiri.semester);
    formData.append('tahun_ajaran', dataDiri.tahunAjaran);
    formData.append('tinggi_badan', dataDiri.tinggiBadan);
    formData.append('berat_badan', dataDiri.beratBadan);
    formData.append('lingkar_kepala', dataDiri.lingkarKepala);

    // 2. Narasi
    formData.append('narasi_agama', narasiAgama);
    formData.append('narasi_jati_diri', narasiJatiDiri);
    formData.append('narasi_literasi', narasiLiterasi);
    if (!isKB) {
      formData.append('narasi_kokurikuler', narasiKokurikuler);
    }
    
    // 3. Absensi & Refleksi
    formData.append('sakit', ketidakhadiran.sakit || 0);
    formData.append('ijin', ketidakhadiran.ijin || 0);
    formData.append('tanpa_keterangan', ketidakhadiran.tanpaKeterangan || 0);
    formData.append('refleksi_guru', refleksiGuru);

    // 4. Data Khusus KB
    if (isKB) {
      formData.append('pembiasaan_agama', JSON.stringify(pembiasaanAgama));
      formData.append('catatan_bu_guru', catatanBuGuru);
    }

    // 5. Lampiran Foto
    const appendPhotos = (fieldName, photoArray) => {
      photoArray.forEach(item => {
        if (item.file) {
          // File di-append sebagai array di PHP agar masuk ke $request->file('foto_agama')
          formData.append(`${fieldName}[]`, item.file);
        }
      });
    };

    appendPhotos('foto_agama', fotoAgama);
    appendPhotos('foto_jati_diri', fotoJatiDiri);
    appendPhotos('foto_literasi', fotoLiterasi);
    if (!isKB) appendPhotos('foto_kokurikuler', fotoKokurikuler);

    const token = localStorage.getItem('auth_token');

    try {
      const response = await fetch(`${PUBLIC_API_URL}/rapor`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Accept': 'application/json'
          // Catatan: JANGAN set Content-Type secara manual jika menggunakan FormData, 
          // browser akan otomatis memberikan nilai multipart/form-data beserta boundary-nya.
        },
        body: formData
      });

      const result = await response.json();

      if (response.ok && result.status) {
        alert("E-Rapor berhasil disimpan ke Database!");
        goto('/dashboard');
      } else {
        alert("Gagal menyimpan rapor: " + result.message);
      }
    } catch (err) {
      console.error("Error save rapor:", err);
      alert("Terjadi kesalahan jaringan saat menyimpan rapor.");
    }
  }
</script>

<svelte:head>
  <title>Buat Rapor - SIAKAD Al Hijrah</title>
</svelte:head>

<div class="flex h-screen bg-gray-50 overflow-hidden font-sans">
  <Sidebar bind:isOpen={isSidebarOpen} />

  <div class="flex-1 flex flex-col overflow-hidden w-full relative">
    
    <header class="h-20 bg-white flex items-center justify-between px-6 md:px-10 shrink-0 border-b border-gray-100">
      <div class="flex items-center gap-4">
        <button class="md:hidden p-2 -ml-2 text-gray-500 hover:bg-gray-100 rounded-lg transition" on:click={() => isSidebarOpen = true}>
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="4" x2="20" y1="12" y2="12"/><line x1="4" x2="20" y1="6" y2="6"/><line x1="4" x2="20" y1="18" y2="18"/></svg>
        </button>
        <button on:click={() => history.back()} class="mr-2 p-2 bg-gray-100 text-gray-600 rounded-full hover:bg-gray-200 transition-colors">
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="m15 18-6-6 6-6"/></svg>
        </button>
        <h1 class="text-xl font-bold text-gray-600">Buat Rapor Siswa</h1>
      </div>
      <div class="flex items-center gap-3">
        <span class="font-bold text-gray-600 hidden sm:block">{userName}</span>
        <div class="w-10 h-10 rounded-full bg-[#2da76b] text-white flex items-center justify-center shrink-0">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="8" r="5"/><path d="M20 21a8 8 0 0 0-16 0"/></svg>
        </div>
      </div>
    </header>

    <main class="flex-1 overflow-y-auto p-6 md:p-10 flex flex-col items-center">
      {#if isLoading}
        <div class="flex items-center justify-center h-full gap-3 text-gray-500 font-bold">
          <svg class="animate-spin h-6 w-6 text-[#2da76b]" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
          Memuat Data Rapor...
        </div>
      {:else}
        
        <div class="w-full max-w-4xl flex flex-col gap-8 pb-10">
          
          <!-- SEKSI: DATA DIRI SISWA -->
          <div class="bg-white rounded-[30px] p-8 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 relative overflow-hidden">
            <div class="absolute top-0 left-0 w-full h-2 bg-[#2da76b]"></div>
            <h2 class="text-2xl font-bold text-gray-800 mb-6">Data Diri Siswa</h2>
            
            <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
              
              <!-- Kolom Kiri -->
              <div class="flex flex-col gap-4">
                <div class="flex flex-col">
                  <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">Nama Sekolah</span>
                  <div class="font-semibold text-gray-700 bg-gray-50 px-4 py-3 rounded-xl border border-gray-100">{dataDiri.namaSekolah}</div>
                </div>
                <div class="flex flex-col">
                  <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">Nama Siswa</span>
                  <div class="font-bold text-gray-800 bg-gray-50 px-4 py-3 rounded-xl border border-gray-100">{dataDiri.namaSiswa}</div>
                </div>
                <div class="flex flex-col">
                  <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">NIS (Nomor Induk)</span>
                  <div class="font-semibold text-gray-700 bg-gray-50 px-4 py-3 rounded-xl border border-gray-100">{dataDiri.nis}</div>
                </div>
                <div class="grid grid-cols-2 gap-4">
                  <div class="flex flex-col">
                    <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">Semester</span>
                    <select bind:value={dataDiri.semester} class="font-semibold text-gray-700 bg-white px-4 py-3 rounded-xl border border-gray-200 focus:outline-none focus:ring-2 focus:ring-[#2da76b]">
                      <option value="Ganjil">Ganjil</option>
                      <option value="Genap">Genap</option>
                    </select>
                  </div>
                  <div class="flex flex-col">
                    <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">Tahun Ajaran</span>
                    <div class="font-semibold text-gray-700 bg-gray-50 px-4 py-3 rounded-xl border border-gray-100">{dataDiri.tahunAjaran}</div>
                  </div>
                </div>
              </div>

              <!-- Kolom Kanan -->
              <div class="flex flex-col gap-4">
                <div class="flex flex-col">
                  <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">Guru Kelas (Wali)</span>
                  <div class="font-semibold text-gray-700 bg-gray-50 px-4 py-3 rounded-xl border border-gray-100">{dataDiri.guruKelas}</div>
                </div>
                <div class="grid grid-cols-2 gap-4">
                  <div class="flex flex-col">
                    <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">Kelompok</span>
                    <div class="font-semibold text-gray-700 bg-gray-50 px-4 py-3 rounded-xl border border-gray-100">{dataDiri.kelas}</div>
                  </div>
                  {#if !isKB}
                  <div class="flex flex-col">
                    <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">Fase</span>
                    <div class="font-semibold text-[#2da76b] bg-green-50 px-4 py-3 rounded-xl border border-green-100">{dataDiri.fase}</div>
                  </div>
                  {/if}
                </div>
                
                <div class="grid grid-cols-3 gap-4 mt-2">
                  <div class="flex flex-col group">
                    <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">Tinggi (cm)</span>
                    <input type="number" bind:value={dataDiri.tinggiBadan} placeholder="0" class="font-bold text-gray-800 text-center bg-white px-2 py-3 rounded-xl border border-gray-200 focus:outline-none focus:ring-2 focus:ring-[#2da76b] transition-all group-hover:border-[#2da76b]" />
                  </div>
                  <div class="flex flex-col group">
                    <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">Berat (kg)</span>
                    <input type="number" bind:value={dataDiri.beratBadan} placeholder="0" class="font-bold text-gray-800 text-center bg-white px-2 py-3 rounded-xl border border-gray-200 focus:outline-none focus:ring-2 focus:ring-[#2da76b] transition-all group-hover:border-[#2da76b]" />
                  </div>
                  <div class="flex flex-col group">
                    <span class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-1">Kepala (cm)</span>
                    <input type="number" bind:value={dataDiri.lingkarKepala} placeholder="0" class="font-bold text-gray-800 text-center bg-white px-2 py-3 rounded-xl border border-gray-200 focus:outline-none focus:ring-2 focus:ring-[#2da76b] transition-all group-hover:border-[#2da76b]" />
                  </div>
                </div>

              </div>

            </div>
          </div>

          <!-- SEKSI: PEMBIASAAN AGAMA (KHUSUS KB) -->
          {#if isKB}
          <div class="bg-white rounded-[30px] p-8 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 flex flex-col gap-6">
            <h3 class="text-xl font-bold text-gray-800 flex items-center justify-between">
              <div class="flex items-center gap-3">
                <span class="w-8 h-8 rounded-full bg-teal-100 text-teal-600 flex items-center justify-center shrink-0">
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 2v20"/><path d="M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"/></svg>
                </span>
                Pembiasaan Agama
              </div>
              <button class="bg-teal-50 text-teal-600 hover:bg-teal-600 hover:text-white px-4 py-2.5 rounded-xl font-bold text-sm flex items-center gap-2 transition-colors" on:click={addMateri}>
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="12" y1="5" x2="12" y2="19"></line><line x1="5" y1="12" x2="19" y2="12"></line></svg>
                Tambah Materi
              </button>
            </h3>

            <div class="overflow-x-auto rounded-2xl border border-gray-100">
              <table class="w-full text-left border-collapse min-w-[500px]">
                <thead>
                  <tr class="bg-teal-50/50 text-teal-800 border-b border-teal-100">
                    <th class="px-5 py-4 font-bold text-sm w-[40%]">Materi</th>
                    <th class="px-3 py-4 font-bold text-sm text-center">Belum Hafal</th>
                    <th class="px-3 py-4 font-bold text-sm text-center">Kurang Lancar</th>
                    <th class="px-3 py-4 font-bold text-sm text-center">Lancar</th>
                    <th class="px-4 py-4 font-bold text-sm text-center w-14"></th>
                  </tr>
                </thead>
                <tbody>
                  {#each pembiasaanAgama as item, i (item.id)}
                    <tr class="border-b border-gray-50 last:border-none group hover:bg-gray-50 transition-colors">
                      <td class="px-4 py-3 align-top">
                        <textarea 
                          use:autoResize={item.materi}
                          bind:value={item.materi}
                          placeholder="Contoh: Doa sebelum makan"
                          class="w-full min-h-[44px] bg-white border border-gray-200 rounded-xl px-4 py-3 text-gray-700 text-sm focus:outline-none focus:ring-2 focus:ring-teal-500 resize-none transition-all font-medium"
                        ></textarea>
                      </td>
                      <td class="px-3 py-3 align-middle text-center">
                        <label class="cursor-pointer inline-flex items-center justify-center w-6 h-6 rounded-full border-2 {item.capaian === 'Belum Hafal' ? 'border-red-500 bg-red-100' : 'border-gray-300 hover:border-red-400'} transition-all">
                          <input type="radio" name="capaian-{item.id}" value="Belum Hafal" bind:group={item.capaian} class="hidden">
                          {#if item.capaian === 'Belum Hafal'}<div class="w-2.5 h-2.5 rounded-full bg-red-500"></div>{/if}
                        </label>
                      </td>
                      <td class="px-3 py-3 align-middle text-center">
                        <label class="cursor-pointer inline-flex items-center justify-center w-6 h-6 rounded-full border-2 {item.capaian === 'Kurang Lancar' ? 'border-yellow-500 bg-yellow-100' : 'border-gray-300 hover:border-yellow-400'} transition-all">
                          <input type="radio" name="capaian-{item.id}" value="Kurang Lancar" bind:group={item.capaian} class="hidden">
                          {#if item.capaian === 'Kurang Lancar'}<div class="w-2.5 h-2.5 rounded-full bg-yellow-500"></div>{/if}
                        </label>
                      </td>
                      <td class="px-3 py-3 align-middle text-center">
                        <label class="cursor-pointer inline-flex items-center justify-center w-6 h-6 rounded-full border-2 {item.capaian === 'Lancar' ? 'border-green-500 bg-green-100' : 'border-gray-300 hover:border-green-400'} transition-all">
                          <input type="radio" name="capaian-{item.id}" value="Lancar" bind:group={item.capaian} class="hidden">
                          {#if item.capaian === 'Lancar'}<div class="w-2.5 h-2.5 rounded-full bg-green-500"></div>{/if}
                        </label>
                      </td>
                      <td class="px-4 py-3 align-middle text-center">
                        <button class="p-2 text-gray-400 hover:text-red-500 hover:bg-red-50 rounded-lg transition-colors opacity-0 group-hover:opacity-100 focus:opacity-100" on:click={() => removeMateri(item.id)}>
                          <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/></svg>
                        </button>
                      </td>
                    </tr>
                  {/each}
                  {#if pembiasaanAgama.length === 0}
                    <tr><td colspan="5" class="py-6 text-center text-gray-400 text-sm font-medium">Belum ada materi ditambahkan. Klik tombol "Tambah Materi".</td></tr>
                  {/if}
                </tbody>
              </table>
            </div>
          </div>
          {/if}

          <!-- SEKSI: CATATAN BU GURU (KHUSUS KB) -->
          {#if isKB}
          <div class="bg-gradient-to-br from-pink-500/10 to-pink-500/5 rounded-[30px] p-8 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-pink-500/20 flex flex-col gap-6">
            <h3 class="text-xl font-bold text-pink-600 flex items-center gap-3">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"></path></svg>
              Catatan Bu Guru
            </h3>
            
            <textarea 
              use:autoResize={catatanBuGuru}
              bind:value={catatanBuGuru}
              placeholder="Tuliskan catatan khusus dari Bu Guru..."
              class="w-full min-h-[120px] bg-white border border-pink-500/30 rounded-2xl p-5 text-gray-700 font-medium focus:outline-none focus:ring-2 focus:ring-pink-400 transition-all resize-none leading-relaxed"
            ></textarea>
          </div>
          {/if}

          <!-- SEKSI: AGAMA DAN BUDI PEKERTI -->
          <div class="bg-white rounded-[30px] p-8 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 flex flex-col gap-6">
            <h3 class="text-xl font-bold text-gray-800 flex items-center gap-3">
              <span class="w-8 h-8 rounded-full bg-blue-100 text-blue-600 flex items-center justify-center shrink-0">1</span>
              Penilaian Agama dan Budi Pekerti
            </h3>
            
            <div class="flex flex-col gap-2 relative group">
              <textarea 
                use:autoResize={narasiAgama}
                bind:value={narasiAgama}
                placeholder="Tuliskan narasi perkembangan anak pada aspek Nilai Agama dan Budi Pekerti..."
                class="w-full min-h-[120px] bg-gray-50 border border-gray-200 rounded-2xl p-5 text-gray-700 font-medium focus:outline-none focus:ring-2 focus:ring-[#2da76b] focus:bg-white transition-all resize-none leading-relaxed"
              ></textarea>
              <button 
                on:click={() => handleGenerateAI('Agama')} 
                disabled={isGenerating.Agama}
                class="self-end mt-2 bg-blue-50 text-blue-600 hover:bg-blue-600 hover:text-white px-5 py-2.5 rounded-xl font-bold text-sm flex items-center gap-2 transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
              >
                {#if isGenerating.Agama}
                  <svg class="animate-spin h-4 w-4" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                  Berpikir...
                {:else}
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m12 3-1.912 5.813a2 2 0 0 1-1.275 1.275L3 12l5.813 1.912a2 2 0 0 1 1.275 1.275L12 21l1.912-5.813a2 2 0 0 1 1.275-1.275L21 12l-5.813-1.912a2 2 0 0 1-1.275-1.275L12 3Z"/></svg>
                  Buat Narasi (AI)
                {/if}
              </button>
            </div>

            <div class="flex flex-col gap-3">
              <span class="font-bold text-gray-600">Foto Kegiatan Anak</span>
              <PhotoUploader bind:photos={fotoAgama} />
            </div>
          </div>

          <!-- SEKSI: JATI DIRI -->
          <div class="bg-white rounded-[30px] p-8 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 flex flex-col gap-6">
            <h3 class="text-xl font-bold text-gray-800 flex items-center gap-3">
              <span class="w-8 h-8 rounded-full bg-purple-100 text-purple-600 flex items-center justify-center shrink-0">2</span>
              Penilaian Jati Diri
            </h3>
            
            <div class="flex flex-col gap-2 relative group">
              <textarea 
                use:autoResize={narasiJatiDiri}
                bind:value={narasiJatiDiri}
                placeholder="Tuliskan narasi perkembangan anak pada aspek Jati Diri..."
                class="w-full min-h-[120px] bg-gray-50 border border-gray-200 rounded-2xl p-5 text-gray-700 font-medium focus:outline-none focus:ring-2 focus:ring-[#2da76b] focus:bg-white transition-all resize-none leading-relaxed"
              ></textarea>
              <button 
                on:click={() => handleGenerateAI('Jati Diri')} 
                disabled={isGenerating['Jati Diri']}
                class="self-end mt-2 bg-blue-50 text-blue-600 hover:bg-blue-600 hover:text-white px-5 py-2.5 rounded-xl font-bold text-sm flex items-center gap-2 transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
              >
                {#if isGenerating['Jati Diri']}
                  <svg class="animate-spin h-4 w-4" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                  Berpikir...
                {:else}
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m12 3-1.912 5.813a2 2 0 0 1-1.275 1.275L3 12l5.813 1.912a2 2 0 0 1 1.275 1.275L12 21l1.912-5.813a2 2 0 0 1 1.275-1.275L21 12l-5.813-1.912a2 2 0 0 1-1.275-1.275L12 3Z"/></svg>
                  Buat Narasi (AI)
                {/if}
              </button>
            </div>

            <div class="flex flex-col gap-3">
              <span class="font-bold text-gray-600">Foto Kegiatan Anak</span>
              <PhotoUploader bind:photos={fotoJatiDiri} />
            </div>
          </div>

          <!-- SEKSI: DASAR LITERASI DAN STEM -->
          <div class="bg-white rounded-[30px] p-8 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 flex flex-col gap-6">
            <h3 class="text-xl font-bold text-gray-800 flex items-center gap-3">
              <span class="w-8 h-8 rounded-full bg-orange-100 text-orange-600 flex items-center justify-center shrink-0">3</span>
              Dasar Literasi dan STEM
            </h3>
            
            <div class="flex flex-col gap-2 relative group">
              <textarea 
                use:autoResize={narasiLiterasi}
                bind:value={narasiLiterasi}
                placeholder="Tuliskan narasi perkembangan anak pada aspek Dasar Literasi dan STEM..."
                class="w-full min-h-[120px] bg-gray-50 border border-gray-200 rounded-2xl p-5 text-gray-700 font-medium focus:outline-none focus:ring-2 focus:ring-[#2da76b] focus:bg-white transition-all resize-none leading-relaxed"
              ></textarea>
              <button 
                on:click={() => handleGenerateAI('Literasi')} 
                disabled={isGenerating['Literasi']}
                class="self-end mt-2 bg-blue-50 text-blue-600 hover:bg-blue-600 hover:text-white px-5 py-2.5 rounded-xl font-bold text-sm flex items-center gap-2 transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
              >
                {#if isGenerating['Literasi']}
                  <svg class="animate-spin h-4 w-4" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                  Berpikir...
                {:else}
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m12 3-1.912 5.813a2 2 0 0 1-1.275 1.275L3 12l5.813 1.912a2 2 0 0 1 1.275 1.275L12 21l1.912-5.813a2 2 0 0 1 1.275-1.275L21 12l-5.813-1.912a2 2 0 0 1-1.275-1.275L12 3Z"/></svg>
                  Buat Narasi (AI)
                {/if}
              </button>
            </div>

            <div class="flex flex-col gap-3">
              <span class="font-bold text-gray-600">Foto Kegiatan Anak</span>
              <PhotoUploader bind:photos={fotoLiterasi} />
            </div>
          </div>

          <!-- SEKSI: KOKURIKULER (HANYA TK) -->
          {#if !isKB}
          <div class="bg-white rounded-[30px] p-8 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 flex flex-col gap-6">
            <h3 class="text-xl font-bold text-gray-800 flex items-center gap-3">
              <span class="w-8 h-8 rounded-full bg-yellow-100 text-yellow-600 flex items-center justify-center shrink-0">4</span>
              Penilaian Kokurikuler
            </h3>
            
            <div class="flex flex-col gap-2 relative group">
              <textarea 
                use:autoResize={narasiKokurikuler}
                bind:value={narasiKokurikuler}
                placeholder="Tuliskan narasi perkembangan anak pada aspek Kokurikuler..."
                class="w-full min-h-[120px] bg-gray-50 border border-gray-200 rounded-2xl p-5 text-gray-700 font-medium focus:outline-none focus:ring-2 focus:ring-[#2da76b] focus:bg-white transition-all resize-none leading-relaxed"
              ></textarea>
              <button 
                on:click={() => handleGenerateAI('Kokurikuler')} 
                disabled={isGenerating['Kokurikuler']}
                class="self-end mt-2 bg-blue-50 text-blue-600 hover:bg-blue-600 hover:text-white px-5 py-2.5 rounded-xl font-bold text-sm flex items-center gap-2 transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
              >
                {#if isGenerating['Kokurikuler']}
                  <svg class="animate-spin h-4 w-4" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                  Berpikir...
                {:else}
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m12 3-1.912 5.813a2 2 0 0 1-1.275 1.275L3 12l5.813 1.912a2 2 0 0 1 1.275 1.275L12 21l1.912-5.813a2 2 0 0 1 1.275-1.275L21 12l-5.813-1.912a2 2 0 0 1-1.275-1.275L12 3Z"/></svg>
                  Buat Narasi (AI)
                {/if}
              </button>
            </div>

            <div class="flex flex-col gap-3">
              <span class="font-bold text-gray-600">Foto Kegiatan Anak</span>
              <PhotoUploader bind:photos={fotoKokurikuler} />
            </div>
          </div>
          {/if}

          <!-- SEKSI: KETIDAKHADIRAN -->
          <div class="bg-white rounded-[30px] p-8 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 flex flex-col gap-6">
            <h3 class="text-xl font-bold text-gray-800 flex items-center gap-3">
              <div class="w-8 h-8 rounded-full bg-red-100 text-red-600 flex items-center justify-center shrink-0">
                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="4" width="18" height="18" rx="2" ry="2"></rect><line x1="16" y1="2" x2="16" y2="6"></line><line x1="8" y1="2" x2="8" y2="6"></line><line x1="3" y1="10" x2="21" y2="10"></line></svg>
              </div>
              Ketidakhadiran
            </h3>
            
            <div class="grid grid-cols-1 md:grid-cols-3 gap-6 mt-2">
              <div class="flex flex-col group">
                <span class="text-sm font-bold text-gray-500 tracking-wider mb-2 text-center">Sakit</span>
                <div class="relative">
                  <input type="number" bind:value={ketidakhadiran.sakit} placeholder="0" class="w-full font-bold text-xl text-gray-800 text-center bg-red-50/50 py-4 rounded-2xl border border-red-100 focus:outline-none focus:ring-2 focus:ring-red-400 transition-all" />
                  <span class="absolute right-4 top-1/2 -translate-y-1/2 font-bold text-gray-400 text-sm">Hari</span>
                </div>
              </div>
              <div class="flex flex-col group">
                <span class="text-sm font-bold text-gray-500 tracking-wider mb-2 text-center">Ijin</span>
                <div class="relative">
                  <input type="number" bind:value={ketidakhadiran.ijin} placeholder="0" class="w-full font-bold text-xl text-gray-800 text-center bg-orange-50/50 py-4 rounded-2xl border border-orange-100 focus:outline-none focus:ring-2 focus:ring-orange-400 transition-all" />
                  <span class="absolute right-4 top-1/2 -translate-y-1/2 font-bold text-gray-400 text-sm">Hari</span>
                </div>
              </div>
              <div class="flex flex-col group">
                <span class="text-sm font-bold text-gray-500 tracking-wider mb-2 text-center">Tanpa Keterangan</span>
                <div class="relative">
                  <input type="number" bind:value={ketidakhadiran.tanpaKeterangan} placeholder="0" class="w-full font-bold text-xl text-gray-800 text-center bg-gray-50/50 py-4 rounded-2xl border border-gray-200 focus:outline-none focus:ring-2 focus:ring-gray-400 transition-all" />
                  <span class="absolute right-4 top-1/2 -translate-y-1/2 font-bold text-gray-400 text-sm">Hari</span>
                </div>
              </div>
            </div>
          </div>

          <!-- SEKSI: REFLEKSI GURU -->
          <div class="bg-gradient-to-br from-[#2da76b]/10 to-[#2da76b]/5 rounded-[30px] p-8 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-[#2da76b]/20 flex flex-col gap-6">
            <h3 class="text-xl font-bold text-[#2da76b] flex items-center gap-3">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 2v20"/><path d="M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"/></svg>
              Refleksi Guru
            </h3>
            
            <textarea 
              use:autoResize={refleksiGuru}
              bind:value={refleksiGuru}
              placeholder="Tuliskan catatan refleksi khusus dari guru kelas..."
              class="w-full min-h-[120px] bg-white border border-[#2da76b]/30 rounded-2xl p-5 text-gray-700 font-medium focus:outline-none focus:ring-2 focus:ring-[#2da76b] transition-all resize-none leading-relaxed"
            ></textarea>
          </div>

          <!-- TOMBOL SIMPAN -->
          <div class="flex justify-end mt-4 mb-8">
            <button on:click={handleSimpan} class="bg-[#2da76b] hover:bg-[#289562] text-white px-12 py-4 rounded-2xl font-bold text-lg shadow-[0_8px_20px_-6px_rgba(45,167,107,0.4)] hover:shadow-[0_12px_24px_-6px_rgba(45,167,107,0.5)] hover:-translate-y-1 transition-all flex items-center gap-3">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"/><polyline points="17 21 17 13 7 13 7 21"/><polyline points="7 3 7 8 15 8"/></svg>
              Simpan E-Rapor
            </button>
          </div>

        </div>

      {/if}
    </main>

  </div>
</div>
