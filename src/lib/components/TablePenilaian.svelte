<script lang="ts">
  import { goto } from '$app/navigation';
  import {PUBLIC_API_URL} from '$env/static/public';
  import { onMount } from 'svelte';
  export let data = [];
  export let basePath = "";
  export let tingkat = "KB"; 
  export let isLoading = false;

  let allSiswa = [];
  
  onMount(async () => {
    const token = localStorage.getItem('auth_token');
    try {
      const response = await fetch(`${PUBLIC_API_URL}/siswa?tingkat=${tingkat}`, {
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      const result = await response.json();
      if (response.ok && result.status) {
        allSiswa = result.data;
      }
    } catch (e) {
      console.error(e);
    }
  });

  // --- STATE UNTUK FILTER & PENCARIAN ---
  let searchQuery = "";
  let selectedKelasFilter = "all";
  let selectedSemesterFilter = "all";

  // Karena dataKelas tidak diteruskan secara langsung, kita ambil dari data penilaian yang ada
  $: uniqueKelas = [...new Map(data.filter(item => item.kelas).map(item => [item.kelas, { nama_kelas: item.kelas, tahun_ajaran: item.tahun_ajaran }])).values()].sort((a, b) => a.nama_kelas.localeCompare(b.nama_kelas));

  $: availableSiswa = selectedKelasFilter === "all" ? allSiswa : allSiswa.filter(s => s.kelas && s.kelas.nama_kelas === selectedKelasFilter);

  $: displayedData = data.filter(item => {
    const matchKelas = selectedKelasFilter === "all" || item.kelas === selectedKelasFilter;
    if (!matchKelas) return false;

    const matchSemester = selectedSemesterFilter === "all" || item.semester === selectedSemesterFilter;
    if (!matchSemester) return false;

    if (!searchQuery) return true;
    const term = searchQuery.toLowerCase();
    return (
      item.nama?.toLowerCase().includes(term) ||
      item.noInduk?.toLowerCase().includes(term) ||
      item.kelas?.toLowerCase().includes(term) ||
      item.tahun_ajaran?.toLowerCase().includes(term) ||
      item.semester?.toLowerCase().includes(term)
    );
  });

  // --- STATE UNTUK MODAL DOWNLOAD & HAPUS ---
  let showDownloadModal = false;
  let showDeleteModal = false;
  let activeItem = null;

  function openDownloadModal(item) {
    if (item.status === 'Draft') return;
    activeItem = item;
    showDownloadModal = true;
  }

  function openDeleteModal(item) {
    activeItem = item;
    showDeleteModal = true;
  }

  async function confirmDelete() {
    const token = localStorage.getItem('auth_token');
    const id = activeItem.id;

    try {
      const response = await fetch(`${PUBLIC_API_URL}/penilaian/${id}`, {
        method: 'DELETE',
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      const result = await response.json();
      if (result.status) {
        data = data.filter(d => d.id !== id);
        showDeleteModal = false;
        activeItem = null;
      } else {
        alert("Gagal menghapus: " + result.message);
      }
    } catch (error) {
      alert("Gagal terhubung ke server.");
    }
  }

  // --- FUNGSI PEMBANTU UNTUK MERAKIT DATA SEBELUM DOWNLOAD ---
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

  // --- LOGIKA DOWNLOAD EXCEL & PDF DARI MODAL ---
  let isDownloading = false; // Mencegah user klik 2 kali saat loading

  async function handleDownloadExcel() {
    if (!activeItem) return;
    isDownloading = true;
    const id = activeItem.id;
    const token = localStorage.getItem('auth_token');
    
    try {
      // 1. Ambil data siswa dan daftar elemennya
      const resDetail = await fetch(`${PUBLIC_API_URL}/penilaian/${id}`, { headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' } });
      const detailJson = await resDetail.json();
      if (!detailJson.status) throw new Error("Gagal memuat detail");
      
      const studentName = detailJson.data.penilaian.siswa.nama;
      const daftarElemen = detailJson.data.elemen;

      const XLSX = await import('xlsx');
      const workbook = XLSX.utils.book_new();
      let hasData = false;

      // 2. Ambil nilai matriks untuk masing-masing elemen
      for (let elemen of daftarElemen) {
        const resMatriks = await fetch(`${PUBLIC_API_URL}/penilaian/matriks/${id}/${elemen.id}`, { headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' } });
        const result = await resMatriks.json();
        
        if (!result.status) continue;
        const dataPenilaian = rebuildTableData(result.data.saved_details, result.data.elemen);
        if (dataPenilaian.length === 0) continue;
        
        hasData = true;
        let aoa = [
          ["RANGKUMAN PENILAIAN"], ["NAMA", studentName], ["ELEMEN CAPAIAN", elemen.nama_elemen],
          ["CAPAIAN PEMBELAJARAN", "TUJUAN PEMBELAJARAN", "ATP/Indikator (Utama)", "PERTEMUAN", "", "", "", "", "", "", "", "", "", "", "", "", "", "", "", "NILAI AKHIR"], 
          ["", "", "", "1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13", "14", "15", "16", ""]
        ];

        let merges = [
          { s: { r: 0, c: 0 }, e: { r: 0, c: 19 } }, { s: { r: 3, c: 0 }, e: { r: 4, c: 0 } },
          { s: { r: 3, c: 1 }, e: { r: 4, c: 1 } }, { s: { r: 3, c: 2 }, e: { r: 4, c: 2 } },
          { s: { r: 3, c: 3 }, e: { r: 3, c: 18 } }, { s: { r: 3, c: 19 }, e: { r: 4, c: 19 } }
        ];

        let currentRow = 5;
        dataPenilaian.forEach(cp => {
          let cpStartRow = currentRow; let cpRowCount = 0;
          cp.tujuan.forEach(tp => {
            let tpStartRow = currentRow; let tpRowCount = 0;
            tp.atp.forEach(atp => {
              aoa.push([cp.teksCP, tp.teksTP, atp.teksATP, ...(atp.pertemuan.map(v => v || '')), getNilaiAkhir(atp.pertemuan)]);
              currentRow++; cpRowCount++; tpRowCount++;
            });
            if (tpRowCount > 1) merges.push({ s: { r: tpStartRow, c: 1 }, e: { r: tpStartRow + tpRowCount - 1, c: 1 } });
          });
          if (cpRowCount > 1) merges.push({ s: { r: cpStartRow, c: 0 }, e: { r: cpStartRow + cpRowCount - 1, c: 0 } });
        });

        const worksheet = XLSX.utils.aoa_to_sheet(aoa);
        worksheet['!merges'] = merges;
        worksheet['!cols'] = [{ wch: 35 }, { wch: 35 }, { wch: 45 }, ...Array(16).fill({ wch: 4 }), { wch: 12 }];

        let sheetName = elemen.nama_elemen.substring(0, 30).replace(/[:\\\/?*\[\]]/g, '');
        XLSX.utils.book_append_sheet(workbook, worksheet, sheetName);
      }

      if (!hasData) { alert("Siswa ini belum memiliki nilai yang disimpan!"); return; }
      XLSX.writeFile(workbook, `Rapor_${studentName}.xlsx`);
      showDownloadModal = false; // Tutup otomatis setelah selesai
    } catch (e) {
      console.error(e); alert("Terjadi kesalahan saat memproses data.");
    } finally {
      isDownloading = false;
    }
  }

  async function handleDownloadPDF() {
    if (!activeItem) return;
    isDownloading = true;
    const id = activeItem.id;
    const token = localStorage.getItem('auth_token');
    
    try {
      const resDetail = await fetch(`${PUBLIC_API_URL}/penilaian/${id}`, { headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' } });
      const detailJson = await resDetail.json();
      if (!detailJson.status) throw new Error("Gagal memuat detail");
      
      const studentName = detailJson.data.penilaian.siswa.nama;
      const studentNoInduk = detailJson.data.penilaian.siswa.nomor_induk;
      const daftarElemen = detailJson.data.elemen;

      const { default: jsPDF } = await import('jspdf');
      const { default: autoTable } = await import('jspdf-autotable');

      const doc = new jsPDF('l', 'pt', 'a4');
      let hasData = false;
      let pageIndex = 0;

      for (let elemen of daftarElemen) {
        const resMatriks = await fetch(`${PUBLIC_API_URL}/penilaian/matriks/${id}/${elemen.id}`, { headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' } });
        const result = await resMatriks.json();
        
        if (!result.status) continue;
        const dataPenilaian = rebuildTableData(result.data.saved_details, result.data.elemen);
        if (dataPenilaian.length === 0) continue;

        hasData = true;
        if (pageIndex > 0) doc.addPage();
        pageIndex++;

        doc.setFontSize(14); doc.setFont("helvetica", "bold"); doc.text(`Rangkuman Penilaian: ${elemen.nama_elemen}`, 40, 40);
        doc.setFontSize(10); doc.setFont("helvetica", "normal"); doc.text(`Nama Siswa: ${studentName}`, 40, 60); doc.text(`Nomor Induk: ${studentNoInduk}`, 40, 75);

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
          head: [columns], body: rows, startY: 90, theme: 'grid',
          styles: { fontSize: 7, cellPadding: 4, lineColor: [220, 220, 220], lineWidth: 0.5 },
          headStyles: { fillColor: [45, 167, 107], textColor: 255, halign: 'center', valign: 'middle' },
          columnStyles: { 0: { cellWidth: 120 }, 1: { cellWidth: 120 }, 2: { cellWidth: 120 } }
        });
      }

      if (!hasData) { alert("Siswa ini belum memiliki nilai yang disimpan!"); return; }
      doc.save(`Rapor_${studentName}.pdf`);
      showDownloadModal = false; // Tutup otomatis setelah selesai
    } catch (e) {
      console.error(e); alert("Terjadi kesalahan saat memproses data.");
    } finally {
      isDownloading = false;
    }
  }

  // --- LOGIKA MODAL TAMBAH PENILAIAN ---
  let showModal = false;
  let inputNama = "";
  let inputNoInduk = "";
  let inputTahun = "";
  let inputSemester = "Ganjil";

  function openModal() {
    inputNama = "";
    inputNoInduk = "";
    inputSemester = selectedSemesterFilter !== "all" ? selectedSemesterFilter : "Ganjil";
    // Coba ambil tahun ajaran dari kelas yang dipilih jika ada
    if (selectedKelasFilter !== "all") {
      const selectedClassData = uniqueKelas.find(k => k.nama_kelas === selectedKelasFilter);
      inputTahun = selectedClassData ? selectedClassData.tahun_ajaran : "";
    } else {
      inputTahun = ""; 
    }
    showModal = true;
  }

  function closeModal() {
    showModal = false;
    showDropdown = false;
  }

  let showDropdown = false;
  $: filteredSiswa = availableSiswa.filter(s => s.nama.toLowerCase().includes(inputNama.toLowerCase()));

  function selectSiswa(siswa) {
    inputNama = siswa.nama;
    inputNoInduk = siswa.nomor_induk;
    showDropdown = false;
  }

  async function handleSubmit() {
    if(!inputNama || !inputNoInduk || !inputTahun || !inputSemester) {
      alert("Mohon lengkapi semua data!");
      return;
    }
    const token = localStorage.getItem('auth_token');
    try {
      const response = await fetch(`${PUBLIC_API_URL}/penilaian`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
          nama: inputNama,
          no_induk: inputNoInduk,
          tahun_ajaran: inputTahun,
          semester: inputSemester,
          tingkat: tingkat 
        })
      });

      const result = await response.json();

      if (response.ok && result.status) {
        data = [...data, {
          id: result.data.id,
          nama: inputNama,
          noInduk: inputNoInduk,
          status: result.data.status
        }];
        closeModal();
      } else {
        alert(result.message || "Gagal menyimpan ke database.");
      }
    } catch (error) {
      alert("Gagal menghubungi server.");
    }
  }

  // --- LOGIKA MODAL EDIT PENILAIAN ---
  let showEditModal = false;
  let editId = null;
  let editNama = "";
  let editNoInduk = "";
  let editTahun = "";
  let editSemester = "Ganjil";

  let isEditing = false;

  function openEditModal(item) {
    editId = item.id;
    editNama = item.nama;
    editNoInduk = item.noInduk;
    editTahun = item.tahun_ajaran;
    editSemester = item.semester || "Ganjil";
    showEditModal = true;
  }

  function closeEditModal() {
    showEditModal = false;
    editId = null;
    editNama = "";
    editNoInduk = "";
    editTahun = "";
    editSemester = "Ganjil";
    isEditing = false;
  }

  async function handleEdit() {
    if(!editNama || !editNoInduk || !editTahun || !editSemester) {
      alert("Mohon lengkapi semua data!");
      return;
    }

    isEditing = true;

    const token = localStorage.getItem('auth_token');

    try {
      // Asumsi endpoint update Laravel menggunakan method PUT
      const response = await fetch(`${PUBLIC_API_URL}/penilaian/${editId}`, {
        method: 'PUT', 
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
          nama: editNama,
          no_induk: editNoInduk,
          tahun_ajaran: editTahun,
          semester: editSemester,
          tingkat: tingkat 
        })
      });

      const result = await response.json();

      if (response.ok && result.status) {
        // Update data di tabel secara lokal tanpa perlu refresh halaman
        data = data.map(row => {
          if (row.id === editId) {
            return { ...row, nama: editNama, noInduk: editNoInduk, tahun_ajaran: editTahun, semester: editSemester };
          }
          return row;
        });
        closeEditModal();
      } else {
        alert(result.message || "Gagal memperbarui database.");
      }
    } catch (error) {
      alert("Gagal menghubungi server.");
    } finally{
      isEditing = false;
    }
  }
</script>

<main class="flex-1 overflow-y-auto p-6 md:p-10">
  <div class="bg-white rounded-3xl shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 p-6 md:p-8 flex flex-col gap-6">
    
    <div class="flex flex-col md:flex-row md:items-center justify-between gap-4">
      <div class="flex items-center gap-6">
        <button on:click={openModal} class="bg-[#2da76b] hover:bg-[#289562] text-white px-5 py-2.5 rounded-xl font-bold transition-colors flex items-center gap-2 shadow-sm disabled:opacity-50">
          <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><line x1="12" x2="12" y1="5" y2="19"/><line x1="5" x2="19" y1="12" y2="12"/></svg>
          Tambah Penilaian
        </button>

        <div class="flex items-center gap-3 w-full sm:w-auto">
          <span class="font-bold text-gray-600 hidden sm:block">Semester</span>
          <select bind:value={selectedSemesterFilter} class="border-2 border-[#2da76b] text-[#2da76b] px-4 py-2.5 rounded-xl font-bold bg-white focus:outline-none focus:ring-2 focus:ring-green-200 transition-all appearance-none cursor-pointer pr-10">
            <option value="all">Semua Semester</option>
            <option value="Ganjil">Ganjil</option>
            <option value="Genap">Genap</option>
          </select>
        </div>

        <div class="flex items-center gap-3 w-full sm:w-auto">
          <span class="font-bold text-gray-600 hidden sm:block">Kelas</span>
          <select bind:value={selectedKelasFilter} class="border-2 border-[#2da76b] text-[#2da76b] px-4 py-2.5 rounded-xl font-bold bg-white focus:outline-none focus:ring-2 focus:ring-green-200 transition-all appearance-none cursor-pointer pr-10">
            <option value="all">Semua Kelas</option>
            {#each uniqueKelas as k}
              <option value={k.nama_kelas}>{k.nama_kelas} ({k.tahun_ajaran})</option>
            {/each}
          </select>
        </div>
      </div>

      <div class="relative w-full md:w-[300px]">
        <input type="text" bind:value={searchQuery} placeholder="Cari data..." class="w-full pl-5 pr-10 py-2.5 rounded-xl border border-gray-300 focus:ring-2 focus:ring-[#2da76b] focus:border-[#2da76b] outline-none text-gray-700 transition-all"/>
        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-400"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.3-4.3"/></svg>
      </div>
    </div>

    <div class="rounded-2xl overflow-x-auto border border-gray-100">
      <table class="w-full text-left border-collapse whitespace-nowrap">
        <thead>
          <tr class="bg-[#2da76b] text-white">
            <th class="px-6 py-4 font-semibold text-sm">Nama</th>
            <th class="px-6 py-4 font-semibold text-sm">No. Induk</th>
            <th class="px-6 py-4 font-semibold text-sm">Kelas</th>
            <th class="px-6 py-4 font-semibold text-sm">Semester</th>
            <th class="px-6 py-4 font-semibold text-sm">Status</th>
            <th class="px-6 py-4 font-semibold text-sm text-right">Aksi</th>
          </tr>
        </thead>
        <tbody>
          {#if isLoading}
            <tr>
              <td colspan="6" class="px-6 py-8 text-center text-gray-500 font-medium">
                <div class="flex items-center justify-center gap-3">
                  <svg class="animate-spin h-6 w-6 text-[#2da76b]" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                  Memuat data...
                </div>
              </td>
            </tr>
          {:else}
            {#each displayedData as row}
              <tr class="odd:bg-white even:bg-gray-50 hover:bg-green-50/50 transition-colors cursor-pointer" on:click={() => goto(`${basePath}/${row.id}`)}>
                <td class="px-6 py-4 font-medium text-gray-700">{row.nama}</td>
                <td class="px-6 py-4 text-gray-600">{row.noInduk}</td>
                <td class="px-6 py-4 text-gray-600">{row.kelas} ({row.tahun_ajaran})</td>
                <td class="px-6 py-4 text-gray-600 font-semibold">{row.semester || '-'}</td>
                <td class="px-6 py-4">
                  <span class="px-4 py-1.5 rounded-full text-sm font-bold {row.status === 'Selesai' ? 'bg-green-100 text-green-700' : 'bg-yellow-100 text-yellow-700'}">{row.status}</span>
                </td>
                <td class="px-6 py-4 text-right flex items-center justify-end gap-2">
                  <button class="px-4 py-2 rounded-lg font-semibold text-sm transition-colors {row.status === 'Selesai' ? 'bg-blue-500 hover:bg-blue-600 text-white shadow-sm' : 'bg-gray-300 text-white cursor-not-allowed'}" on:click|stopPropagation={() => openDownloadModal(row)} disabled={row.status === 'Draft'}>Download</button>
                  <button class="p-2 bg-amber-500 hover:bg-amber-600 text-white rounded-lg shadow-sm transition-colors" on:click|stopPropagation={() => openEditModal(row)}>
                    <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/><path d="m15 5 4 4"/></svg>
                  </button>
                  <button class="p-2 bg-red-500 hover:bg-red-600 text-white rounded-lg shadow-sm transition-colors" on:click|stopPropagation={() => openDeleteModal(row)}>
                    <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/></svg>
                  </button>
                </td>
              </tr>
            {/each}
            {#if displayedData.length === 0}
              <tr>
                <td colspan="6" class="px-6 py-8 text-center text-gray-400 font-medium">Tidak ada data ditemukan.</td>
              </tr>
            {/if}
          {/if}
        </tbody>
      </table>
    </div>
  </div>
</main>

{#if showModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-black/40 backdrop-blur-sm" on:click={closeModal}></div>
    <div class="bg-white w-full max-w-lg rounded-[30px] p-8 md:p-10 relative shadow-2xl flex flex-col gap-6 z-10 animate-in fade-in zoom-in-95 duration-200">
      <h2 class="text-3xl font-bold text-gray-600 text-center mb-2">Tambah Penilaian</h2>
      <div class="flex flex-col gap-5">
          <div class="relative">
            <label class="text-gray-600 font-semibold mb-2 block ml-1" for="nama">Nama Siswa</label>
            <input type="text" id="nama" autocomplete="off" bind:value={inputNama} on:focus={() => showDropdown = true} on:input={() => showDropdown = true} placeholder="Cari & Pilih Nama Siswa..." class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium relative z-50"/>
            
            {#if showDropdown}
              <div class="fixed inset-0 z-40" on:click={() => showDropdown = false}></div>
              <ul class="absolute z-50 w-full mt-2 bg-white border border-gray-100 rounded-2xl shadow-xl max-h-60 overflow-y-auto custom-scrollbar overflow-hidden">
                {#if filteredSiswa.length > 0}
                  {#each filteredSiswa as siswa}
                    <!-- svelte-ignore a11y-click-events-have-key-events -->
                    <li class="px-5 py-3 hover:bg-green-50 cursor-pointer transition-colors text-gray-700 font-medium border-b border-gray-50 last:border-0" on:click={() => selectSiswa(siswa)}>
                      {siswa.nama} <span class="text-gray-400 text-sm ml-1">({siswa.nomor_induk} - {siswa.kelas ? siswa.kelas.nama_kelas : 'Belum Ada Kelas'})</span>
                    </li>
                  {/each}
                {:else}
                  <li class="px-5 py-3 text-gray-400 font-medium text-center">Siswa tidak ditemukan</li>
                {/if}
              </ul>
            {/if}
          </div>
        <div>
          <label class="text-gray-600 font-semibold mb-2 block ml-1" for="noInduk">Nomor Induk Siswa</label>
          <input type="text" id="noInduk" bind:value={inputNoInduk} placeholder="Masukkan Nomor Induk Siswa" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium"/>
        </div>
        <div>
          <label class="text-gray-600 font-semibold mb-2 block ml-1" for="inputTahun">Tahun Pelajaran (Berdasarkan Kelas)</label>
          <input type="text" id="inputTahun" bind:value={inputTahun} class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none text-gray-800 font-medium opacity-80 cursor-not-allowed" readonly disabled/>
        </div>
        <div>
          <label class="text-gray-600 font-semibold mb-2 block ml-1" for="inputSemester">Semester</label>
          <select id="inputSemester" bind:value={inputSemester} class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none text-gray-800 font-medium focus:ring-2 focus:ring-[#2da76b] transition-all cursor-pointer">
            <option value="Ganjil">Ganjil</option>
            <option value="Genap">Genap</option>
          </select>
        </div>
      </div>
      <button on:click={handleSubmit} class="w-full py-4 mt-4 bg-[#2da76b] text-white rounded-2xl font-bold text-lg hover:bg-[#289562] transition-colors shadow-sm">Tambah Penilaian</button>
      <button on:click={closeModal} class="absolute top-6 right-6 text-gray-400 hover:text-gray-600 transition-colors bg-gray-100 p-2 rounded-full">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg>
      </button>
    </div>
  </div>
{/if}

{#if showDownloadModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-gray-900/40 backdrop-blur-sm transition-opacity" on:click={() => showDownloadModal = false}></div>
    <div class="bg-white rounded-3xl p-8 max-w-sm w-full shadow-2xl relative z-10 flex flex-col items-center">
      <button class="absolute top-4 right-4 text-gray-400 hover:text-gray-600" on:click={() => showDownloadModal = false}>
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
      </button>
      <div class="w-16 h-16 bg-blue-50 text-blue-500 rounded-full flex items-center justify-center mb-4">
        <svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" x2="12" y1="15" y2="3"/></svg>
      </div>
      <h3 class="text-xl font-bold text-gray-800 mb-1">Download Nilai</h3>
      <p class="text-gray-500 text-sm text-center mb-6">Pilih format file laporan untuk <br/> <strong class="text-gray-700">{activeItem?.nama}</strong></p>
      <div class="flex gap-4 w-full">
        <button
        on:click={handleDownloadPDF} 
        disabled={isDownloading}
        class="flex-1 flex flex-col items-center justify-center gap-2 p-4 rounded-2xl border-2 border-red-100 hover:border-red-500 hover:bg-red-50 transition-all group">
          <span class="font-bold text-gray-700">PDF</span>
        </button>
        <button
        on:click={handleDownloadExcel}
        disabled={isDownloading} 
        class="flex-1 flex flex-col items-center justify-center gap-2 p-4 rounded-2xl border-2 border-emerald-100 hover:border-emerald-500 hover:bg-emerald-50 transition-all group">
          <span class="font-bold text-gray-700">Excel</span>
        </button>
      </div>
    </div>
  </div>
{/if}

{#if showDeleteModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-gray-900/40 backdrop-blur-sm transition-opacity" on:click={() => showDeleteModal = false}></div>
    <div class="bg-white rounded-3xl p-8 max-w-sm w-full shadow-2xl relative z-10 flex flex-col items-center">
      <div class="w-16 h-16 bg-red-100 text-red-500 rounded-full flex items-center justify-center mb-4">
        <svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="m21.73 18-8-14a2 2 0 0 0-3.48 0l-8 14A2 2 0 0 0 4 21h16a2 2 0 0 0 1.73-3Z"/><line x1="12" x2="12" y1="9" y2="13"/><line x1="12" x2="12.01" y1="17" y2="17"/></svg>
      </div>
      <h3 class="text-xl font-bold text-gray-800 mb-2">Hapus Data?</h3>
      <p class="text-gray-500 text-center mb-8">Apakah Anda yakin ingin menghapus data penilaian milik <strong class="text-gray-700">{activeItem?.nama}</strong>?</p>
      <div class="flex gap-3 w-full">
        <button class="flex-1 py-3 px-4 bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold rounded-xl transition-colors" on:click={() => showDeleteModal = false}>Batal</button>
        <button class="flex-1 py-3 px-4 bg-red-500 hover:bg-red-600 text-white font-bold rounded-xl transition-colors shadow-sm" on:click={confirmDelete}>Ya, Hapus</button>
      </div>
    </div>
  </div>
{/if}

{#if showEditModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-black/40 backdrop-blur-sm" on:click={closeEditModal}></div>
    <div class="bg-white w-full max-w-lg rounded-[30px] p-8 md:p-10 relative shadow-2xl flex flex-col gap-6 z-10 animate-in fade-in zoom-in-95 duration-200">
      <h2 class="text-3xl font-bold text-gray-600 text-center mb-2">Edit Penilaian</h2>
      <div class="flex flex-col gap-5">
        <div>
          <label class="text-gray-600 font-semibold mb-2 block ml-1" for="editNama">Nama Siswa</label>
          <input type="text" id="editNama" bind:value={editNama} placeholder="Masukkan Nama Siswa" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-amber-500 text-gray-800 placeholder:text-gray-500 font-medium"/>
        </div>
        <div>
          <label class="text-gray-600 font-semibold mb-2 block ml-1" for="editNoInduk">Nomor Induk Siswa</label>
          <input type="text" id="editNoInduk" bind:value={editNoInduk} placeholder="Masukkan Nomor Induk Siswa" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-amber-500 text-gray-800 placeholder:text-gray-500 font-medium"/>
        </div>
        <div>
          <label class="text-gray-600 font-semibold mb-2 block ml-1" for="editTahun">Tahun Pelajaran</label>
          <input type="text" id="editTahun" bind:value={editTahun} class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none text-gray-800 font-medium" />
        </div>
        <div>
          <label class="text-gray-600 font-semibold mb-2 block ml-1" for="editSemester">Semester</label>
          <select id="editSemester" bind:value={editSemester} class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none text-gray-800 font-medium focus:ring-2 focus:ring-[#2da76b] transition-all cursor-pointer">
            <option value="Ganjil">Ganjil</option>
            <option value="Genap">Genap</option>
          </select>
        </div>
      </div>
      <button 
        on:click={handleEdit} 
        disabled={isEditing}
        class="w-full py-4 mt-4 bg-amber-500 text-white rounded-2xl font-bold text-lg transition-colors shadow-sm flex items-center justify-center gap-2 {isEditing ? 'opacity-70 cursor-not-allowed' : 'hover:bg-amber-600'}"
      >
        {#if isEditing}
          <svg class="animate-spin h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
          Menyimpan
        {:else}
          Simpan Perubahan
        {/if}
      </button>
      <button on:click={closeEditModal} class="absolute top-6 right-6 text-gray-400 hover:text-gray-600 transition-colors bg-gray-100 p-2 rounded-full">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg>
      </button>
    </div>
  </div>
{/if}