<script lang="ts">
  import Sidebar from '$lib/components/Sidebar.svelte';
  import { page } from '$app/stores'; 
  import { onMount } from 'svelte';
  import {PUBLIC_API_URL} from '$env/static/public';

  let isSidebarOpen = false;
  let userName = "Bu Hijrah";

  let penilaianId = $page.params.id; 
  
  let studentName = "Loading...";
  let studentNoInduk = "...";
  let statusPenilaian = "Draft"; // Menyimpan status saat ini
  let daftarElemen = []; 

  onMount(async () => {
    const token = localStorage.getItem('auth_token');
    const userData = JSON.parse(localStorage.getItem('user_data') || '{}');
    if(userData.name) userName = userData.name;

    try {
      const response = await fetch(`${PUBLIC_API_URL}/penilaian/${penilaianId}`, {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Accept': 'application/json'
        }
      });

      const result = await response.json();

      if (response.ok && result.status) {
        studentName = result.data.penilaian.siswa.nama;
        studentNoInduk = result.data.penilaian.siswa.nomor_induk;
        statusPenilaian = result.data.penilaian.status;
        daftarElemen = result.data.elemen;
      }
    } catch (error) {
      console.error("Gagal memuat detail penilaian:", error);
    }
  });

  // --- FUNGSI TANDAI SELESAI ---
  async function tandaiSelesai() {
    if (!confirm("Tandai penilaian ini sebagai Selesai?")) return;
    const token = localStorage.getItem('auth_token');

    try {
      const response = await fetch(`${PUBLIC_API_URL}/penilaian/${penilaianId}/status`, {
        method: 'PUT',
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      const result = await response.json();
      if (result.status) {
        statusPenilaian = 'Selesai';
        alert("Berhasil! " + result.message);
      } else {
        alert("Gagal: " + result.message);
      }
    } catch (error) {
      alert("Gagal terhubung ke server.");
    }
  }

  // --- HELPER UNTUK MENGOLAH DATA EXCEL & PDF ---
  function getNilaiAkhir(pertemuan) {
    const grades = pertemuan.filter(v => v !== '');
    if (grades.includes('A')) return 'A';
    if (grades.includes('B')) return 'B';
    if (grades.includes('C')) return 'C';
    return '-';
  }

  function rebuildTableData(savedDetails, masterData) {
    let rebuiltData = [];
    if (!savedDetails || savedDetails.length === 0) return [];

    masterData.capaian_pembelajarans.forEach(cp => {
      let currentCp = null;
      cp.tujuan_pembelajarans.forEach(tp => {
        let currentTp = null;
        tp.atp_indikators.forEach(atp => {
          let savedMatch = savedDetails.find(d => d.atp_indikator_id === atp.id);
          if (savedMatch) {
            if (!currentCp) { currentCp = { teksCP: cp.deskripsi, tujuan: [] }; rebuiltData.push(currentCp); }
            if (!currentTp) { currentTp = { teksTP: tp.deskripsi, atp: [] }; currentCp.tujuan.push(currentTp); }
            currentTp.atp.push({
              teksATP: atp.deskripsi,
              pertemuan: savedMatch.pertemuan || Array(16).fill('')
            });
          }
        });
      });
    });
    return rebuiltData;
  }

  // --- FUNGSI DOWNLOAD EXCEL (MULTIPLE SHEETS) ---
  async function downloadExcelAll() {
    const token = localStorage.getItem('auth_token');
    const XLSX = await import('xlsx');
    const workbook = XLSX.utils.book_new();
    let hasData = false;

    // Looping ke semua elemen yang ada (Agama, Jati Diri, dll)
    for (let elemen of daftarElemen) {
      const res = await fetch(`${PUBLIC_API_URL}/penilaian/matriks/${penilaianId}/${elemen.id}`, { 
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' } 
      });
      const result = await res.json();
      
      if (!result.status) continue;
      
      const dataPenilaian = rebuildTableData(result.data.saved_details, result.data.elemen);
      if (dataPenilaian.length === 0) continue; // Abaikan elemen yang belum diisi nilainya
      
      hasData = true;

      // 1. Buat Kerangka Array of Arrays
      let aoa = [
        ["RANGKUMAN PENILAIAN"],
        ["NAMA", studentName],
        ["ELEMEN CAPAIAN", elemen.nama_elemen],
        ["CAPAIAN PEMBELAJARAN", "TUJUAN PEMBELAJARAN", "ATP/Indikator (Utama)", "PERTEMUAN", "", "", "", "", "", "", "", "", "", "", "", "", "", "", "", "NILAI AKHIR"], 
        ["", "", "", "1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13", "14", "15", "16", ""]
      ];

      let merges = [
        { s: { r: 0, c: 0 }, e: { r: 0, c: 19 } },
        { s: { r: 3, c: 0 }, e: { r: 4, c: 0 } },
        { s: { r: 3, c: 1 }, e: { r: 4, c: 1 } },
        { s: { r: 3, c: 2 }, e: { r: 4, c: 2 } },
        { s: { r: 3, c: 3 }, e: { r: 3, c: 18 } },
        { s: { r: 3, c: 19 }, e: { r: 4, c: 19 } }
      ];

      let currentRow = 5;
      dataPenilaian.forEach(cp => {
        let cpStartRow = currentRow;
        let cpRowCount = 0;
        cp.tujuan.forEach(tp => {
          let tpStartRow = currentRow;
          let tpRowCount = 0;
          tp.atp.forEach(atp => {
            let row = [
              cp.teksCP, tp.teksTP, atp.teksATP,
              ...(atp.pertemuan.map(val => val || '')),
              getNilaiAkhir(atp.pertemuan)
            ];
            aoa.push(row);
            currentRow++; cpRowCount++; tpRowCount++;
          });
          if (tpRowCount > 1) merges.push({ s: { r: tpStartRow, c: 1 }, e: { r: tpStartRow + tpRowCount - 1, c: 1 } });
        });
        if (cpRowCount > 1) merges.push({ s: { r: cpStartRow, c: 0 }, e: { r: cpStartRow + cpRowCount - 1, c: 0 } });
      });

      // 2. Jadikan Sheet Baru
      const worksheet = XLSX.utils.aoa_to_sheet(aoa);
      worksheet['!merges'] = merges;
      worksheet['!cols'] = [{ wch: 35 }, { wch: 35 }, { wch: 45 }, ...Array(16).fill({ wch: 4 }), { wch: 12 }];

      // Potong nama elemen jika kepanjangan (aturan Excel max 31 karakter)
      let sheetName = elemen.nama_elemen.substring(0, 30).replace(/[:\\\/?*\[\]]/g, '');
      XLSX.utils.book_append_sheet(workbook, worksheet, sheetName);
    }

    if (!hasData) { alert("Siswa ini belum memiliki nilai yang disimpan!"); return; }
    XLSX.writeFile(workbook, `Rapor_${studentName}.xlsx`);
  }

  // --- FUNGSI DOWNLOAD PDF (MULTIPLE PAGES) ---
  async function downloadPDFAll() {
    const { default: jsPDF } = await import('jspdf');
    const { default: autoTable } = await import('jspdf-autotable');
    const token = localStorage.getItem('auth_token');

    const doc = new jsPDF('l', 'pt', 'a4');
    let hasData = false;
    let pageIndex = 0;

    for (let elemen of daftarElemen) {
      const res = await fetch(`${PUBLIC_API_URL}/penilaian/matriks/${penilaianId}/${elemen.id}`, { 
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' } 
      });
      const result = await res.json();
      
      if (!result.status) continue;
      
      const dataPenilaian = rebuildTableData(result.data.saved_details, result.data.elemen);
      if (dataPenilaian.length === 0) continue;

      hasData = true;

      // Tambah halaman baru jika ini bukan elemen pertama
      if (pageIndex > 0) doc.addPage();
      pageIndex++;

      doc.setFontSize(14);
      doc.setFont("helvetica", "bold");
      doc.text(`Rangkuman Penilaian: ${elemen.nama_elemen}`, 40, 40);
      doc.setFontSize(10);
      doc.setFont("helvetica", "normal");
      doc.text(`Nama Siswa: ${studentName}`, 40, 60);
      doc.text(`Nomor Induk: ${studentNoInduk}`, 40, 75);

      let rows = [];
      dataPenilaian.forEach(cp => {
        let cpRowCount = 0;
        cp.tujuan.forEach(tp => tp.atp.forEach(atp => { if(atp.teksATP) cpRowCount++; }));
        if (cpRowCount === 0) return;

        let isFirstCpRow = true;
        cp.tujuan.forEach(tp => {
          let tpRowCount = 0;
          tp.atp.forEach(atp => { if(atp.teksATP) tpRowCount++; });
          if (tpRowCount === 0) return;

          let isFirstTpRow = true;
          tp.atp.forEach(atp => {
            if (atp.teksATP) {
              let row = [];
              if (isFirstCpRow) { row.push({ content: cp.teksCP, rowSpan: cpRowCount, styles: { valign: 'middle' } }); isFirstCpRow = false; }
              if (isFirstTpRow) { row.push({ content: tp.teksTP, rowSpan: tpRowCount, styles: { valign: 'middle' } }); isFirstTpRow = false; }
              row.push({ content: atp.teksATP, styles: { valign: 'middle' } });
              for (let i = 0; i < 16; i++) { row.push({ content: atp.pertemuan[i] || '', styles: { halign: 'center', valign: 'middle' } }); }
              row.push({ content: getNilaiAkhir(atp.pertemuan), styles: { halign: 'center', valign: 'middle', fontStyle: 'bold', textColor: [45, 167, 107] } });
              rows.push(row);
            }
          });
        });
      });

      const columns = ["Capaian Pembelajaran", "Tujuan Pembelajaran", "ATP/Indikator", "1","2","3","4","5","6","7","8","9","10","11","12","13","14","15","16", "NA"];

      autoTable(doc, {
        head: [columns],
        body: rows,
        startY: 90,
        theme: 'grid',
        styles: { fontSize: 7, cellPadding: 4, lineColor: [220, 220, 220], lineWidth: 0.5 },
        headStyles: { fillColor: [45, 167, 107], textColor: 255, halign: 'center', valign: 'middle' },
        columnStyles: { 0: { cellWidth: 120 }, 1: { cellWidth: 120 }, 2: { cellWidth: 120 } }
      });
    }

    if (!hasData) { alert("Siswa ini belum memiliki nilai yang disimpan!"); return; }
    doc.save(`Rapor_${studentName}.pdf`);
  }

  // --- FUNGSI KEMBALIKAN KE DRAFT ---
  async function kembalikanKeDraft() {
    if (!confirm("Kembalikan status ke Draft? Anda akan bisa mengedit kembali nilai di setiap elemen.")) return;
    
    const token = localStorage.getItem('auth_token');

    try {
      const response = await fetch(`${PUBLIC_API_URL}/penilaian/${penilaianId}/status-draft`, {
        method: 'PUT',
        headers: { 
          'Authorization': `Bearer ${token}`, 
          'Accept': 'application/json' 
        }
      });
      const result = await response.json();
      
      if (result.status) {
        statusPenilaian = 'Draft'; // Update UI secara reaktif
        alert(result.message);
      } else {
        alert("Gagal: " + result.message);
      }
    } catch (error) {
      console.error("Error:", error);
      alert("Gagal terhubung ke server.");
    }
  }
</script>

<svelte:head>
  <title>Detail Penilaian - SIAKAD Al Hijrah</title>
</svelte:head>

<div class="flex h-screen bg-gray-50 overflow-hidden font-sans">
  
  <Sidebar bind:isOpen={isSidebarOpen} />

  <div class="flex-1 flex flex-col overflow-hidden w-full">
    
    <header class="h-20 bg-white flex items-center justify-between px-6 md:px-10 shrink-0 border-b border-gray-100">
      <div class="flex items-center gap-4">
        <button class="md:hidden p-2 -ml-2 text-gray-500 hover:bg-gray-100 rounded-lg transition" on:click={() => isSidebarOpen = true}>
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="4" x2="20" y1="12" y2="12"/><line x1="4" x2="20" y1="6" y2="6"/><line x1="4" x2="20" y1="18" y2="18"/></svg>
        </button>
        <a href="/penilaian/kb" class="flex items-center gap-2 text-gray-500 hover:text-[#2da76b] font-semibold transition-colors">
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="m15 18-6-6 6-6"/></svg>
          Kembali
        </a>
      </div>
      <div class="flex items-center gap-3">
        <span class="font-bold text-gray-600 hidden sm:block">{userName}</span>
        <div class="w-10 h-10 rounded-full bg-[#2da76b] text-white flex items-center justify-center shrink-0">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="8" r="5"/><path d="M20 21a8 8 0 0 0-16 0"/></svg>
        </div>
      </div>
    </header>

    <main class="flex-1 overflow-y-auto p-6 md:p-10 item">
      
      <div class="bg-white rounded-[30px] border border-gray-100 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] p-6 md:p-10 flex flex-col gap-8 max-w-5xl">
        
        <div class="flex flex-col md:flex-row md:items-center gap-8 relative">
          <div class="w-40 h-40 bg-gray-300 rounded-3xl flex items-end justify-center overflow-hidden shrink-0 relative">
            <div class="absolute top-6 w-16 h-16 bg-gray-400 rounded-full"></div>
            <div class="w-28 h-16 bg-gray-400 rounded-t-[40px] mt-auto"></div>
          </div>
          
          <div class="flex flex-col gap-2">
            <h2 class="text-2xl font-bold text-gray-700">Nama : {studentName}</h2>
            <h3 class="text-2xl font-bold text-gray-500">Nomor Induk : {studentNoInduk}</h3>
            
            <div class="mt-2 inline-flex">
              <span class="px-4 py-1.5 rounded-full text-sm font-bold {statusPenilaian === 'Selesai' ? 'bg-green-100 text-green-700' : 'bg-yellow-100 text-yellow-700'}">
                Status: {statusPenilaian}
              </span>
            </div>
          </div>
        </div>

        <div class="flex flex-col xl:flex-row xl:items-center justify-between gap-6 mt-4">
          <h3 class="text-2xl font-bold text-gray-700">Elemen Penilaian</h3>
          
          <div class="flex flex-wrap items-center gap-3">
            {#if statusPenilaian !== 'Selesai'}
              <button on:click={tandaiSelesai} class="border-2 border-[#33b679] text-[#33b679] px-5 py-2.5 rounded-[14px] font-bold flex items-center gap-2 hover:bg-green-50 transition-colors">
                <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><path d="m9 11 3 3L22 4"/></svg>
                Tandai Selesai
              </button>
            {:else}
              <button on:click={kembalikanKeDraft} class="border-2 border-yellow-500 text-yellow-500 px-5 py-2.5 rounded-[14px] font-bold flex items-center gap-2 hover:bg-orange-50 transition-colors">
                <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M11 17l-5-5 5-5M18 17l-5-5 5-5"/></svg>
                Kembalikan ke Draft
              </button>
            {/if}
            
            <button on:click={downloadExcelAll} class="bg-[#33b679] text-white px-5 py-2.5 rounded-[14px] font-bold flex items-center gap-2 hover:bg-[#289562] transition-colors shadow-sm">
              <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M14.5 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7.5L14.5 2z"/><polyline points="14 2 14 8 20 8"/><path d="M8 13h2"/><path d="M8 17h2"/><path d="M14 13h2"/><path d="M14 17h2"/></svg>
              Download Excel
            </button>
            
            <button on:click={downloadPDFAll} class="bg-[#ea4335] text-white px-5 py-2.5 rounded-[14px] font-bold flex items-center gap-2 hover:bg-[#c9362a] transition-colors shadow-sm">
              <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M14.5 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7.5L14.5 2z"/><polyline points="14 2 14 8 20 8"/><path d="M10 12v6"/><path d="M10 12h2a2 2 0 0 1 2 2v0a2 2 0 0 1-2 2h-2"/><path d="M10 18h4"/></svg>
              Download PDF
            </button>
          </div>
        </div>

        <div class="flex flex-col gap-4">
          {#each daftarElemen as elemen}
            <a href="/penilaian/kb/{penilaianId}/{elemen.id}" class="w-full bg-[#33b679] text-white px-6 py-4 rounded-2xl font-bold text-lg text-left flex items-center gap-4 hover:bg-[#289562] transition-colors shadow-sm">
              <svg xmlns="http://www.w3.org/2000/svg" width="28" height="28" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                <rect width="8" height="4" x="8" y="2" rx="1" ry="1"/>
                <path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2"/>
                <path d="m9 14 2 2 4-4"/>
              </svg>
              {elemen.nama_elemen}
            </a>
          {/each}

          {#if daftarElemen.length === 0}
             <div class="text-center p-6 text-gray-500 font-semibold bg-gray-50 rounded-2xl border border-gray-100">
                Memuat daftar elemen...
             </div>
          {/if}
        </div>
      </div>
    </main>
  </div>
</div>