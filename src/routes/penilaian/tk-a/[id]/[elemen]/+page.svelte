<script lang="ts">
  import { page } from '$app/stores';
  import Sidebar from '$lib/components/Sidebar.svelte';
  import { onMount } from 'svelte'; 
  import * as XLSX from 'xlsx';
  import {PUBLIC_API_URL} from '$env/static/public';
  // import jsPDF from 'jspdf';
  // import 'jspdf-autotable';
	// import { url } from 'inspector';

  let isManualMode = false;
  let manualText = "";

  // --- STATE UTAMA ---
  let isEditMode = false;
  let isSidebarOpen = false;
  let userName = "Bu Hijrah";

  let penilaianId = $page.params.id;
  let elemenId = $page.params.elemen;

  // --- STATE UNTUK DATA API ---
  let studentName = "Loading...";
  let studentNoInduk = "...";
  let namaElemen = "Loading Elemen...";
  let masterDataElemen = null; 
  let isDataLoading = true;

  

  // Mengambil data dari Backend Laravel saat halaman pertama kali dibuka
  onMount(async () => {
    const token = localStorage.getItem('auth_token');
    
    const userData = JSON.parse(localStorage.getItem('user_data') || '{}');
    if(userData.name) userName = userData.name;

    try {
      const response = await fetch(`${PUBLIC_API_URL}/penilaian/matriks/${penilaianId}/${elemenId}`, {
        headers: {
          'Authorization': `Bearer ${token}`, 
          'Accept': 'application/json'
        }
      });
      const result = await response.json();
      
      if (result.status) {
        studentName = result.data.siswa.nama;
        studentNoInduk = result.data.siswa.nomor_induk;
        
        masterDataElemen = result.data.elemen;
        namaElemen = masterDataElemen.nama_elemen;

        // 👇 PANGGIL FUNGSI PERAKIT DI SINI
        dataPenilaian = rebuildTableData(result.data.saved_details, masterDataElemen);
      }
    } catch (error) {
      console.error("Gagal memuat matriks data:", error);
    } finally {
      isDataLoading = false;
    }
  });

  // --- FUNGSI PERAKIT DATA (Rebuild Data) ---
  function rebuildTableData(savedDetails, masterData) {
    let rebuiltData = [];
    if (!savedDetails || savedDetails.length === 0) {
      return [{
        id: crypto.randomUUID(), teksCP: "", 
        tujuan: [{ id: crypto.randomUUID(), teksTP: "", 
          atp: [{ id: crypto.randomUUID(), teksATP: "", pertemuan: Array(16).fill('') }] 
        }]
      }];
    }

    masterData.capaian_pembelajarans.forEach(cp => {
      let currentCp = null;

      cp.tujuan_pembelajarans.forEach(tp => {
        let currentTp = null;

        // 1. Ambil ATP dari Master Data yang punya nilai di savedDetails
        tp.atp_indikators.forEach(atp => {
          let savedMatch = savedDetails.find(d => d.atp_indikator_id === atp.id);
          if (savedMatch) {
            if (!currentCp) {
              currentCp = { id: crypto.randomUUID(), db_id: cp.id, teksCP: cp.deskripsi, tujuan: [] };
              rebuiltData.push(currentCp);
            }
            if (!currentTp) {
              currentTp = { id: crypto.randomUUID(), db_id: tp.id, teksTP: tp.deskripsi, atp: [] };
              currentCp.tujuan.push(currentTp);
            }
            currentTp.atp.push({
              id: crypto.randomUUID(),
              db_id: atp.id,
              teksATP: atp.deskripsi,
              pertemuan: savedMatch.pertemuan || Array(16).fill('')
            });
          }
        });

        // 2. Ambil ATP CUSTOM (Manual) yang terikat ke TP ini
        let customMatches = savedDetails.filter(d => d.tujuan_pembelajaran_id === tp.id && d.atp_indikator_id === null);
        customMatches.forEach(custom => {
            if (!currentCp) {
              currentCp = { id: crypto.randomUUID(), db_id: cp.id, teksCP: cp.deskripsi, tujuan: [] };
              rebuiltData.push(currentCp);
            }
            if (!currentTp) {
              currentTp = { id: crypto.randomUUID(), db_id: tp.id, teksTP: tp.deskripsi, atp: [] };
              currentCp.tujuan.push(currentTp);
            }
            currentTp.atp.push({
              id: crypto.randomUUID(),
              db_id: null, // Karena custom
              teksATP: custom.deskripsi_custom,
              pertemuan: custom.pertemuan || Array(16).fill('')
            });
        });
      });
    });
    return rebuiltData;
  }

  // --- STRUKTUR DATA PENILAIAN ---
  let dataPenilaian = [];

  function getNilaiAkhir(pertemuan) {
    const grades = pertemuan.filter(v => v !== '');
    if (grades.includes('A')) return 'A';
    if (grades.includes('B')) return 'B';
    if (grades.includes('C')) return 'C';
    return '-';
  }

  // --- MODAL SYSTEM (SEKARANG DINAMIS DARI DATABASE) ---
  let showModal = false;
  let modalTitle = "";
  let currentOpsi = [];
  let targetAction = { type: '', cpIdx: 0, tpIdx: 0, atpIdx: 0 };

  function openModal(type, cpIdx, tpIdx = 0, atpIdx = 0) {
    if (!masterDataElemen) {
        alert("Master data belum termuat.");
        return;
    }

    modalTitle = type === 'CP' ? "Capaian Pembelajaran" : type === 'TP' ? "Tujuan Pembelajaran" : "ATP/Indikator";
    targetAction = { type, cpIdx, tpIdx, atpIdx };
    isManualMode = false; // Reset mode setiap kali buka
    manualText = "";

    if (type === 'CP') {
        currentOpsi = masterDataElemen.capaian_pembelajarans.map(cp => ({ id: cp.id, teks: cp.deskripsi }));
    } else if (type === 'TP') {
        const selectedCpId = dataPenilaian[cpIdx].db_id;
        const matchedCp = masterDataElemen.capaian_pembelajarans.find(cp => cp.id === selectedCpId);
        currentOpsi = matchedCp ? matchedCp.tujuan_pembelajarans.map(tp => ({ id: tp.id, teks: tp.deskripsi })) : [];
    } else if (type === 'ATP') {
        const selectedTpId = dataPenilaian[cpIdx].tujuan[tpIdx].db_id;
        let matchedTp = null;
        masterDataElemen.capaian_pembelajarans.forEach(cp => {
            const found = cp.tujuan_pembelajarans.find(tp => tp.id === selectedTpId);
            if (found) matchedTp = found;
        });
        currentOpsi = matchedTp ? matchedTp.atp_indikators.map(atp => ({ id: atp.id, teks: atp.deskripsi })) : [];
    }

    showModal = true;
  }

  function selectOption(opsiObj) {
    const { type, cpIdx, tpIdx, atpIdx } = targetAction;
    if (type === 'CP') {
        dataPenilaian[cpIdx].teksCP = opsiObj.teks;
        dataPenilaian[cpIdx].db_id = opsiObj.id; 
    } else if (type === 'TP') {
        dataPenilaian[cpIdx].tujuan[tpIdx].teksTP = opsiObj.teks;
        dataPenilaian[cpIdx].tujuan[tpIdx].db_id = opsiObj.id; 
    } else if (type === 'ATP') {
        dataPenilaian[cpIdx].tujuan[tpIdx].atp[atpIdx].teksATP = opsiObj.teks;
        dataPenilaian[cpIdx].tujuan[tpIdx].atp[atpIdx].db_id = opsiObj.id; 
    }
    showModal = false;
  }

  function selectManual() {
    const { cpIdx, tpIdx, atpIdx } = targetAction;
    dataPenilaian[cpIdx].tujuan[tpIdx].atp[atpIdx].teksATP = manualText;
    dataPenilaian[cpIdx].tujuan[tpIdx].atp[atpIdx].db_id = null; // Tandai sebagai custom
    showModal = false;
  }

  // --- CONTEXT MENU ---
  let showContextMenu = null;

  function toggleMenu(id) {
    showContextMenu = showContextMenu === id ? null : id;
  }

  function hapusItem(type, cpIdx, tpIdx, atpIdx) {
    if (type === 'CP') dataPenilaian = dataPenilaian.filter((_, i) => i !== cpIdx);
    if (type === 'TP') dataPenilaian[cpIdx].tujuan = dataPenilaian[cpIdx].tujuan.filter((_, i) => i !== tpIdx);
    if (type === 'ATP') dataPenilaian[cpIdx].tujuan[tpIdx].atp = dataPenilaian[cpIdx].tujuan[tpIdx].atp.filter((_, i) => i !== atpIdx);
    showContextMenu = null;
  }

  // --- ROWSPAN & ADD LOGIC ---
  const getTpRowspan = (tp) => Math.max(1, tp.atp.length + (isEditMode ? 1 : 0));
  const getCpRowspan = (cp) => {
    let total = cp.tujuan.reduce((acc, tp) => acc + getTpRowspan(tp), 0);
    return Math.max(1, total + (isEditMode ? 1 : 0));
  };

  const addCp = () => dataPenilaian = [...dataPenilaian, { id: crypto.randomUUID(), teksCP: "", tujuan: [] }];
  const addTp = (i) => {
    dataPenilaian[i].tujuan = [...dataPenilaian[i].tujuan, { id: crypto.randomUUID(), teksTP: "", atp: [] }];
    showContextMenu = null;
  };
  const addAtp = (ci, ti) => {
    dataPenilaian[ci].tujuan[ti].atp = [...dataPenilaian[ci].tujuan[ti].atp, { id: crypto.randomUUID(), teksATP: "", pertemuan: Array(16).fill('') }];
    showContextMenu = null;
  };

  // --- STATE UNTUK FILTER & SORTING ---
  let filterNilai = 'Semua'; // 'Semua', 'A', 'B', 'C'
  let sortNilai = 'Default'; // 'Default', 'Tertinggi', 'Terkecil'

  // Variabel Reaktif: Akan otomatis menghitung ulang jika dataPenilaian, filter, atau sort berubah
  $: displayedData = processData(dataPenilaian, filterNilai, sortNilai);

  function processData(source, filter, sort) {
    // Jika tidak ada filter/sort, kembalikan data utuh
    if (filter === 'Semua' && sort === 'Default') return source;

    // 1. Deep Clone: Gandakan data agar data asli tidak rusak
    let result = JSON.parse(JSON.stringify(source));

    // 2. Proses Filter (Menyaring Nilai)
    if (filter !== 'Semua') {
      result = result.map(cp => {
        let newTujuan = cp.tujuan.map(tp => {
          // Ambil hanya ATP yang nilainya sesuai filter
          let newAtp = tp.atp.filter(atp => getNilaiAkhir(atp.pertemuan) === filter);
          return { ...tp, atp: newAtp };
        }).filter(tp => tp.atp.length > 0); // Buang TP jika tidak punya ATP yang lolos filter
        
        return { ...cp, tujuan: newTujuan };
      }).filter(cp => cp.tujuan.length > 0); // Buang CP jika tidak punya TP yang lolos filter
    }

    // 3. Proses Sort (Mengurutkan di dalam masing-masing Tujuan Pembelajaran)
    if (sort !== 'Default') {
      const bobot = { 'A': 3, 'B': 2, 'C': 1, '-': 0 }; // Pemberian skor untuk diurutkan
      
      result.forEach(cp => {
        cp.tujuan.forEach(tp => {
          tp.atp.sort((a, b) => {
            let nilaiA = bobot[getNilaiAkhir(a.pertemuan)];
            let nilaiB = bobot[getNilaiAkhir(b.pertemuan)];
            // Tertinggi (A -> C), Terkecil (C -> A)
            return sort === 'Tertinggi' ? nilaiB - nilaiA : nilaiA - nilaiB;
          });
        });
      });
    }

    return result;
  }

  // --- MODIFIKASI TOGGLE EDIT MODE ---
  // Kita pastikan saat masuk mode Edit, filter kembali normal
  function toggleEditMode() {
    if (!isEditMode) {
      filterNilai = 'Semua';
      sortNilai = 'Default';
    }
    isEditMode = !isEditMode;
  }

  // function toggleEditMode() {
  //   isEditMode = !isEditMode;
  // }

  async function simpanPenilaian() {
    const token = localStorage.getItem('auth_token');
    let detailsToSave = [];

    dataPenilaian.forEach(cp => {
      cp.tujuan.forEach(tp => {
        tp.atp.forEach(atp => {
          if (atp.teksATP) {
            // Kita butuh TP ID agar data manual tahu dia "anak" siapa
            if (tp.db_id) {
                detailsToSave.push({
                    atp_id: atp.db_id, // Bisa null
                    tp_id: tp.db_id,    // Wajib ada
                    deskripsi_custom: atp.db_id ? null : atp.teksATP,
                    pertemuan: atp.pertemuan,
                    nilai_akhir: getNilaiAkhir(atp.pertemuan)
                });
            }
          }
        });
      });
    });

    if (detailsToSave.length === 0) {
      alert("Belum ada data valid untuk disimpan.");
      return;
    }

    try {
      const response = await fetch(`${PUBLIC_API_URL}/penilaian/matriks/${penilaianId}/${elemenId}`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
          'Accept': 'application/json'
        },
        body: JSON.stringify({ details: detailsToSave })
      });
      const result = await response.json();
      if (result.status) {
        alert("Yeay! " + result.message);
        isEditMode = false;
      }
    } catch (error) {
      alert("Terjadi kesalahan pada server.");
    }
  }

  // --- FUNGSI BATAL & HAPUS DATA ELEMEN ---
  async function hapusDataElemen() {
    // 1. Munculkan peringatan agar tidak terhapus tak sengaja
    if (!confirm("Apakah Anda yakin ingin membatalkan dan menghapus seluruh nilai pada elemen ini?")) {
      return;
    }

    const token = localStorage.getItem('auth_token');

    try {
      // 2. Tembak API Simpan, TAPI kirim 'details' berupa array KOSONG []
      // Laravel akan melihat keranjang kosong ini dan otomatis menghapus datanya dari database!
      const response = await fetch(`${PUBLIC_API_URL}/penilaian/matriks/${penilaianId}/${elemenId}`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
          'Accept': 'application/json'
        },
        body: JSON.stringify({ details: [] }) // 👈 Keranjang Kosong
      });

      const result = await response.json();

      if (result.status) {
        alert("Data elemen berhasil dihapus!");
        
        // 3. Reset tampilan tabel kembali ke 1 baris kosong
        dataPenilaian = [{
          id: crypto.randomUUID(), teksCP: "", 
          tujuan: [{ id: crypto.randomUUID(), teksTP: "", 
            atp: [{ id: crypto.randomUUID(), teksATP: "", pertemuan: Array(16).fill('') }] 
          }]
        }];
        
        // 4. Matikan mode edit
        isEditMode = false;
      } else {
        alert("Gagal menghapus: " + result.message);
      }
    } catch (error) {
      console.error("Gagal menghapus data:", error);
      alert("Terjadi kesalahan pada server.");
    }
  }


  // 3. Fungsi Download Excel
  // 3. Fungsi Download Excel (Versi Rapi dengan Merge & Kolom)
  // 3. Fungsi Download Excel (Struktur Custom Rangkuman Penilaian)
  function downloadExcel() {
    // 1. Siapkan kerangka awal (Array of Arrays) untuk Header atas
    let aoa = [
      ["RANGKUMAN PENILAIAN"], // Baris 1 (Index 0)
      ["NAMA", studentName],   // Baris 2 (Index 1)
      ["ELEMEN CAPAIAN", namaElemen], // Baris 3 (Index 2)
      // Baris 4 (Index 3) - Header Tabel Atas
      ["CAPAIAN PEMBELAJARAN", "TUJUAN PEMBELAJARAN", "ATP/Indikator (Utama)", "PERTEMUAN", "", "", "", "", "", "", "", "", "", "", "", "", "", "", "", "NILAI AKHIR"], 
      // Baris 5 (Index 4) - Header Angka Pertemuan
      ["", "", "", "1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13", "14", "15", "16", ""]
    ];

    // 2. Daftarkan Merge Cell untuk Header Atas
    let merges = [
      { s: { r: 0, c: 0 }, e: { r: 0, c: 19 } }, // Merge Judul "RANGKUMAN PENILAIAN" (A1:T1)
      { s: { r: 3, c: 0 }, e: { r: 4, c: 0 } }, // Merge Header "CAPAIAN PEMBELAJARAN" atas-bawah
      { s: { r: 3, c: 1 }, e: { r: 4, c: 1 } }, // Merge Header "TUJUAN PEMBELAJARAN" atas-bawah
      { s: { r: 3, c: 2 }, e: { r: 4, c: 2 } }, // Merge Header "ATP/Indikator" atas-bawah
      { s: { r: 3, c: 3 }, e: { r: 3, c: 18 } }, // Merge Header "PERTEMUAN" menyamping (D4:S4)
      { s: { r: 3, c: 19 }, e: { r: 4, c: 19 } } // Merge Header "NILAI AKHIR" atas-bawah
    ];

    let currentRow = 5; // Data siswa mulai ditulis di baris ke-6 (Index 5)
    let isDataExist = false;

    // 3. Looping Data Penilaian
    dataPenilaian.forEach(cp => {
      let cpStartRow = currentRow;
      let cpRowCount = 0;

      cp.tujuan.forEach(tp => {
        let tpStartRow = currentRow;
        let tpRowCount = 0;

        tp.atp.forEach(atp => {
          if (atp.teksATP) {
            isDataExist = true;
            
            // Susun baris data (Dari Kiri ke Kanan)
            let row = [
              cp.teksCP,
              tp.teksTP,
              atp.teksATP,
              ...(atp.pertemuan.map(val => val || '')), // Sebar 16 nilai pertemuan
              getNilaiAkhir(atp.pertemuan)
            ];
            
            aoa.push(row);
            currentRow++;
            cpRowCount++;
            tpRowCount++;
          }
        });

        // Merge untuk Tujuan Pembelajaran (Kolom B)
        if (tpRowCount > 1) {
          merges.push({ s: { r: tpStartRow, c: 1 }, e: { r: tpStartRow + tpRowCount - 1, c: 1 } });
        }
      });

      // Merge untuk Capaian Pembelajaran (Kolom A)
      if (cpRowCount > 1) {
        merges.push({ s: { r: cpStartRow, c: 0 }, e: { r: cpStartRow + cpRowCount - 1, c: 0 } });
      }
    });

    if (!isDataExist) {
      alert("Tidak ada data untuk diexport!");
      return;
    }

    // 4. Ubah Array of Arrays (AoA) menjadi Worksheet
    const worksheet = XLSX.utils.aoa_to_sheet(aoa);
    
    // Terapkan semua aturan Merge
    worksheet['!merges'] = merges;

    // 5. Mengatur Lebar Kolom
    worksheet['!cols'] = [
      { wch: 35 }, // A: CP
      { wch: 35 }, // B: TP
      { wch: 45 }, // C: ATP
      ...Array(16).fill({ wch: 4 }), // D-S: Pertemuan
      { wch: 12 }  // T: Nilai Akhir
    ];

    // 6. Buat file dan Download
    const workbook = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(workbook, worksheet, "Penilaian");
    XLSX.writeFile(workbook, `Penilaian_${studentName}_${namaElemen}.xlsx`);
  }

  // 4. Fungsi Download PDF
  // 4. Fungsi Download PDF
  // 4. Fungsi Download PDF
  // 4. Fungsi Download PDF (Versi Auto-Merge / Rowspan)
  async function downloadPDF() {
    const { default: jsPDF } = await import('jspdf');
    const { default: autoTable } = await import('jspdf-autotable');

    let rows = [];

    // 1. Looping Data untuk menghitung Rowspan
    dataPenilaian.forEach(cp => {
      // Hitung total baris (ATP) yang ada di dalam Capaian Pembelajaran ini
      let cpRowCount = 0;
      cp.tujuan.forEach(tp => {
        tp.atp.forEach(atp => { if (atp.teksATP) cpRowCount++; });
      });

      if (cpRowCount === 0) return; // Lewati jika kosong

      let isFirstCpRow = true;

      cp.tujuan.forEach(tp => {
        // Hitung total baris (ATP) yang ada di dalam Tujuan Pembelajaran ini
        let tpRowCount = 0;
        tp.atp.forEach(atp => { if (atp.teksATP) tpRowCount++; });

        if (tpRowCount === 0) return; // Lewati jika kosong

        let isFirstTpRow = true;

        tp.atp.forEach(atp => {
          if (atp.teksATP) {
            let row = [];

            // Kolom 1: CP (Hanya dimasukkan di baris pertama kelompok ini)
            if (isFirstCpRow) {
              row.push({ 
                content: cp.teksCP, 
                rowSpan: cpRowCount, 
                styles: { valign: 'middle' } 
              });
              isFirstCpRow = false;
            }

            // Kolom 2: TP (Hanya dimasukkan di baris pertama kelompok ini)
            if (isFirstTpRow) {
              row.push({ 
                content: tp.teksTP, 
                rowSpan: tpRowCount, 
                styles: { valign: 'middle' } 
              });
              isFirstTpRow = false;
            }

            // Kolom 3: ATP
            row.push({ content: atp.teksATP, styles: { valign: 'middle' } });

            // Kolom 4-19: Pertemuan 1-16 (Dibuat rata tengah)
            for (let i = 0; i < 16; i++) {
              row.push({ 
                content: atp.pertemuan[i] || '', 
                styles: { halign: 'center', valign: 'middle' } 
              });
            }

            // Kolom 20: Nilai Akhir
            row.push({ 
              content: getNilaiAkhir(atp.pertemuan), 
              styles: { halign: 'center', valign: 'middle', fontStyle: 'bold', textColor: [45, 167, 107] } 
            });

            rows.push(row);
          }
        });
      });
    });

    if (rows.length === 0) {
      alert("Tidak ada data untuk diexport!");
      return;
    }

    // 2. Setup Dokumen PDF
    const doc = new jsPDF('l', 'pt', 'a4');
    
    doc.setFontSize(14);
    doc.setFont("helvetica", "bold");
    doc.text(`Penilaian: ${namaElemen}`, 40, 40);
    
    doc.setFontSize(10);
    doc.setFont("helvetica", "normal");
    doc.text(`Nama Siswa: ${studentName}`, 40, 60);
    doc.text(`Nomor Induk: ${studentNoInduk}`, 40, 75);

    const columns = [
      "Capaian Pembelajaran", "Tujuan Pembelajaran", "ATP/Indikator", 
      "1","2","3","4","5","6","7","8","9","10","11","12","13","14","15","16", "Nilai"
    ];
    
    // 3. Render Tabel
    autoTable(doc, {
      head: [columns],
      body: rows,
      startY: 90,
      theme: 'grid', // Memunculkan garis kotak-kotak tabel
      styles: { fontSize: 7, cellPadding: 4, lineColor: [220, 220, 220], lineWidth: 0.5 },
      headStyles: { fillColor: [45, 167, 107], textColor: 255, halign: 'center', valign: 'middle' },
      columnStyles: {
        0: { cellWidth: 120 }, 
        1: { cellWidth: 120 }, 
        2: { cellWidth: 120 }, 
      }
    });

    doc.save(`Penilaian_${studentName}_${namaElemen}.pdf`);
  }
</script>

<div class="flex h-screen bg-gray-50 overflow-hidden font-sans">
  <Sidebar bind:isOpen={isSidebarOpen} />

  <div class="flex-1 flex flex-col overflow-hidden w-full">
    <header class="h-20 bg-white flex items-center justify-between px-6 md:px-10 shrink-0 border-b border-gray-100">
      <div class="flex items-center gap-4">
        <button class="md:hidden p-2 text-gray-500" on:click={() => isSidebarOpen = true}>
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><line x1="4" x2="20" y1="12" y2="12"/><line x1="4" x2="20" y1="6" y2="6"/><line x1="4" x2="20" y1="18" y2="18"/></svg>
        </button>
        <a href="/penilaian/tk-a/{penilaianId}" class="flex items-center gap-2 text-gray-500 font-semibold">
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="m15 18-6-6 6-6"/></svg>
          Kembali
        </a>
      </div>
      <div class="flex items-center gap-3">
        <span class="font-bold text-gray-600">{userName}</span>
        <div class="w-10 h-10 rounded-full bg-[#2da76b] text-white flex items-center justify-center shrink-0">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="8" r="5"/><path d="M20 21a8 8 0 0 0-16 0"/></svg>
        </div>
      </div>
    </header>

    <main class="flex-1 overflow-y-auto p-4 md:p-8">
      <div class="bg-white rounded-3xl border border-gray-200 shadow-sm p-6 max-w-[1440px] mx-auto relative">
        
        <div class="flex justify-center mb-6">
          <div class="border border-[#2da76b] rounded-full px-6 py-2 flex items-center gap-4 bg-white shadow-sm">
            <div class="w-10 h-10 bg-gray-200 rounded-full flex items-center justify-center">
              <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="currentColor" class="text-gray-400"><path d="M12 12c2.21 0 4-1.79 4-4s-1.79-4-4-4-4 1.79-4 4 1.79 4 4 4zm0 2c-2.67 0-8 1.34-8 4v2h16v-2c0-2.66-5.33-4-8-4z"/></svg>
            </div>
            <div class="text-gray-700">
              <p class="font-bold">{studentName}</p>
              <p class="text-xs font-medium text-gray-500">{studentNoInduk}</p>
            </div>
          </div>
        </div>

        <h1 class="text-center text-2xl font-bold text-gray-600 mb-8">{namaElemen}</h1>

        <div class="flex justify-between items-center mb-6">
  
        <div class="flex items-center gap-3">
          <button class="flex items-center gap-2 px-5 py-2.5 rounded-full font-bold text-white transition-all shadow-sm {isEditMode ? 'bg-[#2da76b]' : 'bg-blue-600'}" on:click={toggleEditMode}>
            {#if isEditMode}
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M2 12s3-7 10-7 10 7 10 7-3 7-10 7-10-7-10-7Z"/><circle cx="12" cy="12" r="3"/></svg> Tampilan Lihat
            {:else}
              <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/><path d="m15 5 4 4"/></svg> Tampilan Edit
            {/if}
          </button>

          <button on:click={downloadExcel} class="flex items-center gap-2 px-4 py-2.5 rounded-full font-bold text-[#107c41] bg-green-50 hover:bg-green-100 border border-green-200 transition-all shadow-sm">
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" x2="12" y1="15" y2="3"/></svg>
            Excel
          </button>

          <button on:click={downloadPDF} class="flex items-center gap-2 px-4 py-2.5 rounded-full font-bold text-[#ea4335] bg-red-50 hover:bg-red-100 border border-red-200 transition-all shadow-sm">
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M14.5 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7.5L14.5 2z"/><polyline points="14 2 14 8 20 8"/><path d="M10 12v6"/><path d="M10 12h2a2 2 0 0 1 2 2v0a2 2 0 0 1-2 2h-2"/><path d="M10 18h4"/></svg>
            PDF
          </button>
        </div>

        <div class="flex items-center gap-3">
          <div class="relative">
            <select bind:value={filterNilai} disabled={isEditMode} class="appearance-none pl-4 pr-10 py-2.5 rounded-full border-2 border-gray-200 text-sm font-bold text-gray-600 outline-none focus:border-[#2da76b] disabled:opacity-40 disabled:cursor-not-allowed bg-white cursor-pointer shadow-sm">
              <option value="Semua">Semua Nilai</option>
              <option value="A">Hanya Nilai A</option>
              <option value="B">Hanya Nilai B</option>
              <option value="C">Hanya Nilai C</option>
            </select>
            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-400 pointer-events-none"><path d="m6 9 6 6 6-6"/></svg>
          </div>

          <div class="relative">
            <select bind:value={sortNilai} disabled={isEditMode} class="appearance-none pl-4 pr-10 py-2.5 rounded-full border-2 border-gray-200 text-sm font-bold text-gray-600 outline-none focus:border-[#2da76b] disabled:opacity-40 disabled:cursor-not-allowed bg-white cursor-pointer shadow-sm">
              <option value="Default">Urutan Default</option>
              <option value="Tertinggi">Tertinggi ke Terkecil</option>
              <option value="Terkecil">Terkecil ke Tertinggi</option>
            </select>
            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-400 pointer-events-none"><path d="m6 9 6 6 6-6"/></svg>
          </div>
        </div>

        <div class="flex gap-3">
          <button on:click={simpanPenilaian} class="bg-[#2da76b] text-white px-6 py-2.5 rounded-full font-bold shadow-sm">+ Simpan Penilaian</button>
          <button class="border-2 border-red-500 text-red-500 px-6 py-2.5 rounded-full font-bold"
          on:click={hapusDataElemen}>Batal</button>
        </div>
      </div>

        <div class="overflow-x-auto border border-[#2da76b] rounded-2xl shadow-sm">
          <table class="w-full border-collapse text-sm text-gray-700 min-w-[1300px]">
            <thead>
              <tr class="bg-[#2da76b] text-white">
                <th rowspan="2" class="border border-white/20 p-3 w-[22%]">Capaian Pembelajaran</th>
                <th rowspan="2" class="border border-white/20 p-3 w-[22%]">Tujuan Pembelajaran</th>
                <th rowspan="2" class="border border-white/20 p-3 w-[22%]">ATP/Indikator</th>
                <th colspan="16" class="border border-white/20 p-2 text-center">Pertemuan</th>
                <th rowspan="2" class="border border-white/20 p-3 text-center w-[60px]">Nilai Akhir</th>
              </tr>
              <tr class="bg-[#2da76b] text-white text-[10px]">
                {#each Array(16) as _, i}
                  <th class="border border-white/20 p-1 text-center w-[25px]">{i + 1}</th>
                {/each}
              </tr>
            </thead>
            <tbody>
              {#each (isEditMode ? dataPenilaian : displayedData) as cp, cpIndex}
                {#each cp.tujuan.length > 0 ? cp.tujuan : [{id: 'empty-tp', teksTP: '', atp: []}] as tp, tpIndex}
                  {#each tp.atp.length > 0 ? tp.atp : [{id: 'empty-atp', teksATP: '', pertemuan: Array(16).fill('')}] as atp, atpIndex}
                    <tr>
                      {#if tpIndex === 0 && atpIndex === 0}
                        <td rowspan={getCpRowspan(cp)} class="border border-gray-200 p-4 align-top relative bg-white group">
                          {#if cp.teksCP}
                            <div class="flex justify-between gap-2">
                              <p class="leading-relaxed">{cp.teksCP}</p>
                              {#if isEditMode}
                                <div class="relative">
                                  <button on:click={() => toggleMenu(cp.id)} class="p-1.5 bg-yellow-400 rounded-lg text-white shadow-sm shrink-0">
                                    <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/></svg>
                                  </button>
                                  {#if showContextMenu === cp.id}
                                    <div class="absolute right-0 top-8 bg-white shadow-xl rounded-xl border p-1 z-30 w-32">
                                      <button on:click={() => openModal('CP', cpIndex)} class="w-full text-left p-2 hover:bg-gray-50 rounded-lg text-xs font-bold text-gray-600">Pilih Ulang</button>
                                      <button on:click={() => hapusItem('CP', cpIndex)} class="w-full text-left p-2 hover:bg-red-50 rounded-lg text-xs font-bold text-red-500">Hapus</button>
                                    </div>
                                  {/if}
                                </div>
                              {/if}
                            </div>
                          {:else if isEditMode}
                            <button on:click={() => openModal('CP', cpIndex)} class="w-full py-2 bg-blue-600 text-white rounded-lg font-bold text-xs">+ Tambah</button>
                          {/if}
                        </td>
                      {/if}

                      {#if atpIndex === 0}
                        <td rowspan={getTpRowspan(tp)} class="border border-gray-200 p-4 align-top relative bg-white">
                          {#if tp.teksTP}
                            <div class="flex justify-between gap-2">
                              <p class="leading-relaxed">{tp.teksTP}</p>
                              {#if isEditMode}
                                <div class="relative">
                                  <button on:click={() => toggleMenu(tp.id)} class="p-1.5 bg-yellow-400 rounded-lg text-white shadow-sm">
                                    <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/></svg>
                                  </button>
                                  {#if showContextMenu === tp.id}
                                    <div class="absolute right-0 top-8 bg-white shadow-xl rounded-xl border p-1 z-30 w-32">
                                      <button on:click={() => openModal('TP', cpIndex, tpIndex)} class="w-full text-left p-2 hover:bg-gray-50 rounded-lg text-xs font-bold text-gray-600">Pilih Ulang</button>
                                      <button on:click={() => hapusItem('TP', cpIndex, tpIndex)} class="w-full text-left p-2 hover:bg-red-50 rounded-lg text-xs font-bold text-red-500">Hapus</button>
                                    </div>
                                  {/if}
                                </div>
                              {/if}
                            </div>
                          {:else if isEditMode && tp.id !== 'empty-tp'}
                            <button on:click={() => openModal('TP', cpIndex, tpIndex)} class="w-full py-2 bg-blue-600 text-white rounded-lg font-bold text-xs">+ Tambah</button>
                          {/if}
                        </td>
                      {/if}

                      <td class="border border-gray-200 p-4 align-top relative bg-white">
                        {#if atp.teksATP}
                          <div class="flex justify-between gap-2">
                            <p class="leading-relaxed">{atp.teksATP}</p>
                            {#if isEditMode}
                              <div class="relative">
                                <button on:click={() => toggleMenu(atp.id)} class="p-1.5 bg-yellow-400 rounded-lg text-white shadow-sm">
                                  <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/></svg>
                                </button>
                                {#if showContextMenu === atp.id}
                                  <div class="absolute right-0 top-8 bg-white shadow-xl rounded-xl border p-1 z-30 w-32">
                                    <button on:click={() => openModal('ATP', cpIndex, tpIndex, atpIndex)} class="w-full text-left p-2 hover:bg-gray-50 rounded-lg text-xs font-bold text-gray-600">Pilih Ulang</button>
                                    <button on:click={() => hapusItem('ATP', cpIndex, tpIndex, atpIndex)} class="w-full text-left p-2 hover:bg-red-50 rounded-lg text-xs font-bold text-red-500">Hapus</button>
                                  </div>
                                {/if}
                              </div>
                            {/if}
                          </div>
                        {:else if isEditMode && atp.id !== 'empty-atp'}
                          <button on:click={() => openModal('ATP', cpIndex, tpIndex, atpIndex)} class="w-full py-2 bg-blue-600 text-white rounded-lg font-bold text-xs">+ Tambah</button>
                        {/if}
                      </td>

                      {#each atp.pertemuan as _, pIndex}
                        <td class="border border-gray-200 p-0 text-center align-middle">
                          {#if isEditMode && atp.teksATP}
                            <select bind:value={atp.pertemuan[pIndex]} class="w-full h-full p-1 bg-transparent text-center font-bold text-xs outline-none cursor-pointer">
                              <option value=""></option>
                              <option value="A">A</option>
                              <option value="B">B</option>
                              <option value="C">C</option>
                            </select>
                          {:else}
                            <span class="font-bold text-xs">{atp.pertemuan[pIndex]}</span>
                          {/if}
                        </td>
                      {/each}

                      <td class="border border-gray-200 p-2 text-center align-middle bg-gray-50/50">
                        <span class="font-black text-sm text-[#2da76b]">{getNilaiAkhir(atp.pertemuan)}</span>
                      </td>
                    </tr>
                  {/each}

                  {#if isEditMode && tp.teksTP}
                    <tr class="bg-gray-50/30">
                      <td class="border border-gray-200 p-2 text-center">
                        <button on:click={() => addAtp(cpIndex, tpIndex)} class="px-4 py-1.5 bg-blue-600 text-white rounded-lg font-bold text-[10px]">+ Tambah</button>
                      </td>
                      {#each Array(17) as _}<td class="border border-gray-200"></td>{/each}
                    </tr>
                  {/if}
                {/each}

                {#if isEditMode && cp.teksCP}
                  <tr class="bg-gray-50/30">
                    <td class="border border-gray-200 p-2 text-center">
                      <button on:click={() => addTp(cpIndex)} class="px-4 py-1.5 bg-blue-600 text-white rounded-lg font-bold text-[10px]">+ Tambah</button>
                    </td>
                    <td class="border border-gray-200"></td> {#each Array(17) as _}<td class="border border-gray-200"></td>{/each}
                  </tr>
                {/if}
              {/each}

              {#if isEditMode}
                <tr class="bg-gray-50/30">
                  <td class="border border-gray-200 p-3 text-center">
                    <button on:click={addCp} class="px-6 py-2 bg-blue-600 text-white rounded-lg font-bold text-xs">+ Tambah</button>
                  </td>
                  <td class="border border-gray-200"></td> <td class="border border-gray-200"></td> {#each Array(17) as _}<td class="border border-gray-200"></td>{/each}
                </tr>
              {/if}
            </tbody>
          </table>
        </div>
      </div>
    </main>
  </div>
</div>

{#if showModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-black/40 backdrop-blur-sm" on:click={() => showModal = false}></div>
    <div class="bg-white w-full max-w-lg rounded-[40px] p-8 relative shadow-2xl flex flex-col gap-6 max-h-[80vh] overflow-hidden">
      <h2 class="text-xl font-bold text-gray-700 text-center">{modalTitle}</h2>
      
      <div class="overflow-y-auto pr-2 flex flex-col gap-3">
        {#if isManualMode}
          <div class="p-2">
            <label class="text-xs font-bold text-gray-400 mb-2 block">Tulis Indikator Secara Manual:</label>
            <textarea 
              bind:value={manualText}
              class="w-full h-32 p-4 rounded-3xl bg-gray-50 border-2 border-gray-100 focus:border-[#2da76b] outline-none text-sm font-medium text-gray-600"
              placeholder="Contoh: Murid dapat mencuci tangan secara mandiri tanpa bantuan guru..."
            ></textarea>
          </div>
        {:else}
          {#each currentOpsi as opsi}
            <button 
              on:click={() => selectOption(opsi)}
              class="text-left p-6 border-2 border-gray-100 rounded-3xl hover:border-[#2da76b] hover:bg-green-50 transition-all text-sm leading-relaxed text-gray-600 font-medium"
            >
              {opsi.teks}
            </button>
          {/each}

          {#if targetAction.type === 'ATP'}
            <button 
                on:click={() => isManualMode = true}
                class="text-center p-4 border-2 border-dashed border-gray-200 rounded-3xl text-gray-400 font-bold text-xs hover:bg-gray-50 transition-all"
            >
                + Tulis Indikator Manual (Tidak ada di daftar)
            </button>
          {/if}
        {/if}
      </div>

      <div class="flex gap-3">
          {#if isManualMode}
            <button on:click={() => isManualMode = false} class="flex-1 py-4 bg-gray-100 text-gray-500 rounded-full font-bold">Kembali</button>
            <button on:click={selectManual} class="flex-1 py-4 bg-[#2da76b] text-white rounded-full font-bold">Gunakan Teks Ini</button>
          {:else}
            <button on:click={() => showModal = false} class="w-full py-4 bg-gray-100 text-gray-500 rounded-full font-bold">Batal</button>
          {/if}
      </div>
    </div>
  </div>
{/if}

<style>
  select { appearance: none; -webkit-appearance: none; }
  ::-webkit-scrollbar { width: 6px; }
  ::-webkit-scrollbar-thumb { background: #e2e8f0; border-radius: 10px; }
</style>