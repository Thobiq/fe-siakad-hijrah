<script lang="ts">
  import { PUBLIC_API_URL } from '$env/static/public';
  import * as XLSX from 'xlsx';
  
  // Data dari parent (format: { id, nama, noInduk, nisn, ttl, alamat, namaIbu, status })
  export let data = []; 
  export let dataKelas = [];
  export let basePath = "";
  export let tingkat = "KB"; // "KB", "TK A", "TK B"
  export let isLoading = false;

  // --- STATE KELAS & TAHUN ---
  let selectedKelasFilter = "all";
  let showYearDropdown = false;
  let selectedYear = "25/26";
  const years = ["24/25", "25/26", "26/27"];
  let searchQuery = "";

  // --- LOGIKA SORTING (URUTKAN) ---
  let sortColumn = null;
  let sortOrder = 'asc';

  function handleSort(column) {
    if (sortColumn === column) {
      // Jika kolom yang sama diklik, balik urutannya (asc -> desc -> asc)
      sortOrder = sortOrder === 'asc' ? 'desc' : 'asc';
    } else {
      // Jika kolom berbeda diklik, jadikan asc
      sortColumn = column;
      sortOrder = 'asc';
    }
  }

  // --- LOGIKA CHECKBOX (BULK ACTIONS) ---
  let selectedIds = [];
  $: allSelected = displayedData.length > 0 && selectedIds.length === displayedData.length;

  function toggleSelectAll() {
    if (allSelected) {
      selectedIds = [];
    } else {
      selectedIds = displayedData.map(item => item.id);
    }
  }

  function toggleSelect(id) {
    if (selectedIds.includes(id)) {
      selectedIds = selectedIds.filter(i => i !== id);
    } else {
      selectedIds = [...selectedIds, id];
    }
  }

  // --- REACTIVE DATA (Filter Search + Sorting) ---
  $: displayedData = data
    .filter(item => {
      // Fitur Filter Kelas
      if (selectedKelasFilter !== "all") {
        if (selectedKelasFilter === "") {
          if (item.kelas_id) return false;
        } else {
          if (item.kelas_id != selectedKelasFilter) return false;
        }
      }

      // Fitur Pencarian Cerdas (Cari di semua text)
      if (!searchQuery) return true;
      const term = searchQuery.toLowerCase();
      return (
        item.nama?.toLowerCase().includes(term) ||
        item.noInduk?.toLowerCase().includes(term) ||
        item.nisn?.toLowerCase().includes(term) ||
        item.alamat?.toLowerCase().includes(term) ||
        item.kelas?.nama_kelas?.toLowerCase().includes(term)
      );
    })
    .sort((a, b) => {
      // Fitur Urutkan Kolom
      if (!sortColumn) return 0;
      let valA = a[sortColumn] || "";
      let valB = b[sortColumn] || "";
      
      // Handle angka jika memungkinkan, jika tidak jadikan string
      if (typeof valA === 'string') valA = valA.toLowerCase();
      if (typeof valB === 'string') valB = valB.toLowerCase();

      if (valA < valB) return sortOrder === 'asc' ? -1 : 1;
      if (valA > valB) return sortOrder === 'asc' ? 1 : -1;
      return 0;
    });

  // --- STATE MODAL TAMBAH, EDIT, HAPUS ---
  let showModal = false;
  let showEditModal = false;
  let showDeleteModal = false;
  let activeItem = null;

  // Form State
  let formNama = "";
  let formNoInduk = "";
  let formNisn = "";
  let formTtl = "";
  let formAlamat = "";
  let formNamaIbu = "";
  let formNamaAyah = "";
  let formPekerjaanAyah = "";
  let formPekerjaanIbu = "";
  let formAnakKe = "";
  let formStatus = "Aktif";
  let formKelasId = "";

  // Bulk Action State
  let showBulkActionDropdown = false;
  let showBulkActionModal = false;
  let currentBulkAction = ""; // "pindah_kelas", "naik_kelas", "ubah_status"
  let bulkActionTargetKelasId = "";
  let bulkActionTargetStatus = "Aktif";
  let bulkActionTargetTingkat = tingkat;

  let isProcessing = false; // Efek loading

  // --- FUNGSI MODAL ---
  function resetForm() {
    formNama = ""; formNoInduk = ""; formNisn = ""; 
    formTtl = ""; formAlamat = ""; formNamaIbu = ""; 
    formNamaAyah = ""; formPekerjaanAyah = ""; formPekerjaanIbu = ""; formAnakKe = "";
    formStatus = "Aktif"; formKelasId = "";
  }

  function openTambahModal() {
    resetForm();
    showModal = true;
  }

  function openEditModal(item) {
    activeItem = item;
    formNama = item.nama; formNoInduk = item.noInduk; formNisn = item.nisn;
    formTtl = item.ttl || ""; formAlamat = item.alamat || ""; formNamaIbu = item.nama_ibu || item.namaIbu || "";
    formNamaAyah = item.nama_ayah || ""; formPekerjaanAyah = item.pekerjaan_ayah || ""; 
    formPekerjaanIbu = item.pekerjaan_ibu || ""; formAnakKe = item.anak_ke || "";
    formStatus = item.status; formKelasId = item.kelas_id || "";
    showEditModal = true;
  }

  function openDeleteModal(item) {
    activeItem = item;
    showDeleteModal = true;
  }

  function closeAllModals() {
    showModal = false;
    showEditModal = false;
    showDeleteModal = false;
    showBulkActionModal = false;
    activeItem = null;
    isProcessing = false;
  }

  // --- API HANDLERS ---
  async function handleSimpanSiswa(isEdit = false) {
    if (!formNama || !formNoInduk) {
      alert("Nama dan No. Induk wajib diisi!"); return;
    }

    isProcessing = true;
    const token = localStorage.getItem('auth_token');
    const url = isEdit ? `${PUBLIC_API_URL}/siswa/${activeItem.id}` : `${PUBLIC_API_URL}/siswa`;
    const method = isEdit ? 'PUT' : 'POST';

    try {
      const response = await fetch(url, {
        method: method,
        headers: {
          'Content-Type': 'application/json', 'Accept': 'application/json', 'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
          nama: formNama, nomor_induk: formNoInduk, nisn: formNisn,
          ttl: formTtl, alamat: formAlamat, nama_ibu: formNamaIbu,
          nama_ayah: formNamaAyah, pekerjaan_ayah: formPekerjaanAyah,
          pekerjaan_ibu: formPekerjaanIbu, anak_ke: formAnakKe,
          status: formStatus, tingkat: tingkat, tahun_ajaran: selectedYear,
          kelas_id: formKelasId || null
        })
      });

      const result = await response.json();
      if (result.status) {
        // Simulasi update UI lokal
        // Update lokal agar cepat (bisa juga sekadar trigger refetch di parent, tapi karena data di pass via prop bind:data)
        const updatedKelas = formKelasId ? dataKelas.find(k => k.id == formKelasId) : null;
        if (isEdit) {
          data = data.map(d => d.id === activeItem.id ? { ...d, nama: formNama, noInduk: formNoInduk, nisn: formNisn, ttl: formTtl, alamat: formAlamat, nama_ibu: formNamaIbu, nama_ayah: formNamaAyah, pekerjaan_ayah: formPekerjaanAyah, pekerjaan_ibu: formPekerjaanIbu, anak_ke: formAnakKe, status: formStatus, kelas_id: formKelasId, kelas: updatedKelas } : d);
        } else {
          // Asumsi backend mengembalikan id baru
          data = [...data, { id: result.data?.id || Date.now(), nama: formNama, noInduk: formNoInduk, nisn: formNisn, ttl: formTtl, alamat: formAlamat, nama_ibu: formNamaIbu, nama_ayah: formNamaAyah, pekerjaan_ayah: formPekerjaanAyah, pekerjaan_ibu: formPekerjaanIbu, anak_ke: formAnakKe, status: formStatus, kelas_id: formKelasId, kelas: updatedKelas }];
        }
        closeAllModals();
      } else {
        alert(result.message || "Gagal menyimpan data.");
      }
    } catch (e) {
      alert("Gagal terhubung ke server.");
    } finally {
      isProcessing = false;
    }
  }

  async function confirmDelete() {
    isProcessing = true;
    const token = localStorage.getItem('auth_token');
    try {
      const response = await fetch(`${PUBLIC_API_URL}/siswa/${activeItem.id}`, {
        method: 'DELETE',
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      const result = await response.json();
      if (result.status) {
        data = data.filter(d => d.id !== activeItem.id);
        closeAllModals();
      } else {
        alert("Gagal menghapus: " + result.message);
      }
    } catch (e) {
      alert("Gagal terhubung ke server.");
    } finally {
      isProcessing = false;
    }
  }

  function openBulkActionModal(action) {
    currentBulkAction = action;
    showBulkActionDropdown = false;
    showBulkActionModal = true;
    bulkActionTargetTingkat = tingkat;
    bulkActionTargetKelasId = "";
  }

  async function handleBulkAction() {
    isProcessing = true;
    const token = localStorage.getItem('auth_token');
    try {
      const response = await fetch(`${PUBLIC_API_URL}/siswa/bulk-action`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'Accept': 'application/json', 'Authorization': `Bearer ${token}` },
        body: JSON.stringify({
          action: currentBulkAction,
          siswa_ids: selectedIds,
          kelas_id: bulkActionTargetKelasId || null,
          status: bulkActionTargetStatus,
          tingkat: bulkActionTargetTingkat
        })
      });

      const result = await response.json();
      if (result.status) {
        // Update state lokal untuk merefleksikan perubahan
        const targetKelas = bulkActionTargetKelasId ? dataKelas.find(k => k.id == bulkActionTargetKelasId) : null;
        
        data = data.map(d => {
          if (selectedIds.includes(d.id)) {
            if (currentBulkAction === 'pindah_kelas' || currentBulkAction === 'naik_kelas') {
              return { ...d, kelas_id: bulkActionTargetKelasId || null, kelas: targetKelas, tingkat: bulkActionTargetTingkat };
            } else if (currentBulkAction === 'ubah_status') {
              return { ...d, status: bulkActionTargetStatus };
            }
          }
          return d;
        });

        selectedIds = [];
        closeAllModals();
        alert(result.message);
      } else {
        alert("Gagal melakukan aksi massal: " + result.message);
      }
    } catch (e) {
      alert("Gagal terhubung ke server.");
    } finally {
      isProcessing = false;
    }
  }

  // --- EXPORT TO EXCEL ---
  function exportExcel() {
    if (displayedData.length === 0) {
      alert("Tidak ada data untuk di-export!");
      return;
    }

    const exportData = displayedData.map(item => ({
      "Nama Siswa": item.nama,
      "No. Induk": item.noInduk,
      "NISN": item.nisn || '-',
      "Tempat, Tanggal Lahir": item.ttl || '-',
      "Alamat": item.alamat || '-',
      "Anak Ke": item.anak_ke || '-',
      "Nama Ayah": item.nama_ayah || '-',
      "Pekerjaan Ayah": item.pekerjaan_ayah || '-',
      "Nama Ibu": item.nama_ibu || item.namaIbu || '-',
      "Pekerjaan Ibu": item.pekerjaan_ibu || '-',
      "Kelas": item.kelas ? item.kelas.nama_kelas : 'Belum Ada Kelas',
      "Tingkat": item.tingkat || tingkat,
      "Tahun Ajaran": item.tahun_ajaran || selectedYear,
      "Status": item.status
    }));

    const worksheet = XLSX.utils.json_to_sheet(exportData);
    const workbook = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(workbook, worksheet, "Data Siswa");

    // Generate nama file
    const date = new Date().toISOString().split('T')[0];
    const fileName = `Data_Siswa_${tingkat}_${selectedKelasFilter === 'all' ? 'Semua_Kelas' : 'Per_Kelas'}_${date}.xlsx`;

    XLSX.writeFile(workbook, fileName);
  }
</script>

<main class="flex-1 overflow-y-auto p-6 md:p-10">
  <div class="bg-white rounded-3xl shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 p-6 md:p-8 flex flex-col gap-6">
    
    <div class="flex flex-col md:flex-row md:items-center justify-between gap-4">
      <div class="flex items-center gap-6">
        <button on:click={openTambahModal} class="bg-[#2da76b] hover:bg-[#289562] text-white px-5 py-2.5 rounded-xl font-bold transition-colors flex items-center gap-2 shadow-sm disabled:opacity-50">
          <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><line x1="12" x2="12" y1="5" y2="19"/><line x1="5" x2="19" y1="12" y2="12"/></svg>
          Tambah Siswa
        </button>

        <button on:click={exportExcel} class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2.5 rounded-xl font-bold transition-colors flex items-center gap-2 shadow-sm">
          <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path><polyline points="14 2 14 8 20 8"></polyline><line x1="8" y1="13" x2="16" y2="13"></line><line x1="8" y1="17" x2="16" y2="17"></line><polyline points="10 9 9 9 8 9"></polyline></svg>
          Export Excel
        </button>

        <div class="flex items-center gap-3 relative">
          <span class="font-bold text-gray-600 hidden sm:block">Kelas</span>
          <select bind:value={selectedKelasFilter} class="border-2 border-[#2da76b] text-[#2da76b] px-4 py-2.5 rounded-xl font-bold bg-white focus:outline-none focus:ring-2 focus:ring-green-200 transition-all appearance-none cursor-pointer pr-10">
            <option value="all">Semua Kelas</option>
            <option value="">-- Belum Ada Kelas --</option>
            {#each dataKelas as k}
              <option value={k.id}>{k.nama_kelas} ({k.tahun_ajaran})</option>
            {/each}
          </select>
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round" stroke-linejoin="round" class="absolute right-4 top-1/2 -translate-y-1/2 text-[#2da76b] pointer-events-none"><path d="m6 9 6 6 6-6"/></svg>
        </div>
      </div>

      <div class="relative w-full md:w-[300px]">
        <input type="text" bind:value={searchQuery} placeholder="Cari data..." class="w-full pl-5 pr-10 py-2.5 rounded-xl border border-gray-300 focus:ring-2 focus:ring-[#2da76b] focus:border-[#2da76b] outline-none text-gray-700 transition-all"/>
        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-400"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.3-4.3"/></svg>
      </div>
    </div>

    {#if selectedIds.length > 0}
      <div class="flex items-center justify-between bg-green-50 p-4 rounded-xl border border-green-200 shadow-sm animate-in fade-in slide-in-from-top-2 duration-300">
        <span class="text-green-800 font-bold">{selectedIds.length} Data Terpilih</span>
        <div class="relative">
          <button class="bg-[#2da76b] text-white px-4 py-2 rounded-lg font-bold flex items-center gap-2 shadow-sm hover:bg-[#289562] transition-colors" on:click={() => showBulkActionDropdown = !showBulkActionDropdown}>
            Aksi Massal
            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="transition-transform duration-200 {showBulkActionDropdown ? 'rotate-180' : ''}"><path d="m6 9 6 6 6-6"/></svg>
          </button>
          {#if showBulkActionDropdown}
            <div class="absolute right-0 mt-2 w-48 bg-white rounded-xl shadow-xl border border-gray-100 py-2 z-50 animate-in fade-in zoom-in-95 duration-200">
              <button class="w-full text-left px-4 py-2 hover:bg-gray-50 font-medium text-gray-700 transition-colors" on:click={() => openBulkActionModal('pindah_kelas')}>Pindah Kelas</button>
              <button class="w-full text-left px-4 py-2 hover:bg-gray-50 font-medium text-gray-700 transition-colors" on:click={() => openBulkActionModal('naik_kelas')}>Naik Kelas</button>
              <button class="w-full text-left px-4 py-2 hover:bg-gray-50 font-medium text-gray-700 transition-colors" on:click={() => openBulkActionModal('ubah_status')}>Ubah Status</button>
            </div>
            <!-- Tutup dropdown jika klik di luar -->
            <div class="fixed inset-0 z-40" on:click={() => showBulkActionDropdown = false}></div>
          {/if}
        </div>
      </div>
    {/if}

    <div class="rounded-2xl overflow-x-auto border border-gray-100">
      <table class="w-full text-left border-collapse whitespace-nowrap">
        <thead>
          <tr class="bg-[#2da76b] text-white">
            <th class="px-6 py-4 w-10">
              <input type="checkbox" class="w-4 h-4 rounded text-[#2da76b] focus:ring-[#2da76b] cursor-pointer" checked={allSelected} on:change={toggleSelectAll} />
            </th>
            {#each [
              { key: 'nama', label: 'Nama' },
              { key: 'noInduk', label: 'No. Induk' },
              { key: 'nisn', label: 'NISN' },
              { key: 'ttl', label: 'TTL' },
              { key: 'alamat', label: 'Alamat' },
              { key: 'anak_ke', label: 'Anak Ke' },
              { key: 'nama_ayah', label: 'Nama Ayah' },
              { key: 'pekerjaan_ayah', label: 'Pekerjaan Ayah' },
              { key: 'namaIbu', label: 'Nama Ibu' },
              { key: 'pekerjaan_ibu', label: 'Pekerjaan Ibu' },
              { key: 'kelas_id', label: 'Kelas' },
              { key: 'status', label: 'Status' }
            ] as col}
              <th class="px-6 py-4 font-semibold text-sm cursor-pointer hover:bg-[#289562] transition-colors select-none group" on:click={() => handleSort(col.key)}>
                <div class="flex items-center gap-2">
                  {col.label}
                  <span class="text-[10px] opacity-50 group-hover:opacity-100 transition-opacity {sortColumn === col.key ? 'opacity-100 text-yellow-300' : ''}">
                    {#if sortColumn === col.key}
                      {sortOrder === 'asc' ? '▲' : '▼'}
                    {:else}
                      ↑↓
                    {/if}
                  </span>
                </div>
              </th>
            {/each}
            <th class="px-6 py-4 font-semibold text-sm text-center">Aksi</th>
          </tr>
        </thead>
        <tbody>
          {#if isLoading}
            <tr>
              <td colspan="14" class="px-6 py-8 text-center text-gray-500 font-medium">
                <div class="flex items-center justify-center gap-3">
                  <svg class="animate-spin h-6 w-6 text-[#2da76b]" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                  Memuat data...
                </div>
              </td>
            </tr>
          {:else}
            {#each displayedData as row}
              <tr class="odd:bg-white even:bg-gray-50 hover:bg-green-50/50 transition-colors {selectedIds.includes(row.id) ? 'bg-green-50/80' : ''}">
                <td class="px-6 py-4 w-10">
                  <input type="checkbox" class="w-4 h-4 rounded text-[#2da76b] focus:ring-[#2da76b] cursor-pointer" checked={selectedIds.includes(row.id)} on:change={() => toggleSelect(row.id)} />
                </td>
                <td class="px-6 py-4 font-medium text-gray-700">{row.nama}</td>
                <td class="px-6 py-4 text-gray-600">{row.noInduk}</td>
                <td class="px-6 py-4 text-gray-600">{row.nisn || '-'}</td>
                <td class="px-6 py-4 text-gray-600">{row.ttl || '-'}</td>
                <td class="px-6 py-4 text-gray-600 truncate max-w-[150px]">{row.alamat || '-'}</td>
                <td class="px-6 py-4 text-gray-600">{row.anak_ke || '-'}</td>
                <td class="px-6 py-4 text-gray-600">{row.nama_ayah || '-'}</td>
                <td class="px-6 py-4 text-gray-600">{row.pekerjaan_ayah || '-'}</td>
                <td class="px-6 py-4 text-gray-600">{row.nama_ibu || row.namaIbu || '-'}</td>
                <td class="px-6 py-4 text-gray-600">{row.pekerjaan_ibu || '-'}</td>
                <td class="px-6 py-4 text-[#2da76b] font-bold">{row.kelas ? row.kelas.nama_kelas : '-'}</td>
                <td class="px-6 py-4">
                  <span class="px-4 py-1.5 rounded-full text-sm font-bold {row.status === 'Aktif' ? 'bg-green-100 text-green-700' : 'bg-gray-200 text-gray-600'}">{row.status}</span>
                </td>
                <td class="px-6 py-4 flex items-center justify-center gap-2">
                  <button class="p-2 bg-amber-500 hover:bg-amber-600 text-white rounded-full shadow-sm transition-colors" on:click={() => openEditModal(row)}>
                    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/><path d="m15 5 4 4"/></svg>
                  </button>
                </td>
              </tr>
            {/each}
            {#if displayedData.length === 0}
              <tr>
                <td colspan="14" class="px-6 py-8 text-center text-gray-400 font-medium">Tidak ada data ditemukan.</td>
              </tr>
            {/if}
          {/if}
        </tbody>
      </table>
    </div>
  </div>
</main>

{#if showModal || showEditModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-gray-900/40 backdrop-blur-sm" on:click={closeAllModals}></div>
    
    <div class="bg-white w-full max-w-[700px] max-h-[85vh] rounded-[30px] relative shadow-2xl flex flex-col z-10 animate-in fade-in zoom-in-95 duration-200 overflow-hidden">
      
      <div class="px-8 pt-8 pb-4 flex-shrink-0 relative">
        <h2 class="text-3xl font-bold text-gray-600 text-center">{showEditModal ? 'Edit' : 'Tambah'} Siswa</h2>
        <button on:click={closeAllModals} class="absolute top-8 right-8 text-gray-400 hover:text-gray-600 transition-colors bg-gray-100 p-2 rounded-full">
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg>
        </button>
      </div>

      <div class="px-8 py-2 overflow-y-auto flex-1 custom-scrollbar">
        <div class="grid grid-cols-1 md:grid-cols-2 gap-5 pb-4">
          
          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Nama Siswa</label>
            <input type="text" bind:value={formNama} placeholder="Nama Lengkap" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>
          
          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">No. Induk</label>
            <input type="text" bind:value={formNoInduk} placeholder="Nomor Induk" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>
          
          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">NISN</label>
            <input type="text" bind:value={formNisn} placeholder="NISN (Opsional)" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>
          
          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Tempat, Tanggal Lahir</label>
            <input type="text" bind:value={formTtl} placeholder="Contoh: Jember, 08-06-2026" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>
          
          <div class="md:col-span-2">
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Alamat</label>
            <input type="text" bind:value={formAlamat} placeholder="Alamat Lengkap" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>
          
          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Anak Ke-</label>
            <input type="text" bind:value={formAnakKe} placeholder="Contoh: 1, 2, atau 1 dari 3 bersaudara" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>
          
          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Pilih Kelas</label>
            <select bind:value={formKelasId} class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 font-medium transition-all">
              <option value="">-- Belum Ada Kelas --</option>
              {#each dataKelas as k}
                <option value={k.id}>{k.nama_kelas} ({k.tahun_ajaran})</option>
              {/each}
            </select>
          </div>

          <!-- DATA ORANG TUA -->
          <div class="md:col-span-2 mt-2 border-t pt-4">
            <h3 class="text-lg font-bold text-gray-600 mb-2">Data Orang Tua</h3>
          </div>

          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Nama Ayah Kandung</label>
            <input type="text" bind:value={formNamaAyah} placeholder="Nama Ayah" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>

          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Pekerjaan Ayah</label>
            <input type="text" bind:value={formPekerjaanAyah} placeholder="Pekerjaan Ayah" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>

          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Nama Ibu Kandung</label>
            <input type="text" bind:value={formNamaIbu} placeholder="Nama Ibu" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>

          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Pekerjaan Ibu</label>
            <input type="text" bind:value={formPekerjaanIbu} placeholder="Pekerjaan Ibu" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>
          
          <div class="md:col-span-2 mt-2 border-t pt-4">
            <h3 class="text-lg font-bold text-gray-600 mb-2">Status Siswa</h3>
          </div>
          
          <div class="md:col-span-2">
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Status</label>
            <select bind:value={formStatus} class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 font-medium transition-all">
              <option value="Aktif">Aktif</option>
              <option value="Tidak Aktif">Tidak Aktif</option>
              <option value="Lulus">Lulus</option>
              <option value="Pindah">Pindah</option>
            </select>
          </div>
        </div>
      </div>

      <div class="px-8 pb-8 pt-2 flex-shrink-0 bg-white">
        <button on:click={() => handleSimpanSiswa(showEditModal)} disabled={isProcessing} class="w-full py-4 {showEditModal ? 'bg-amber-500 hover:bg-amber-600' : 'bg-[#2da76b] hover:bg-[#289562]'} text-white rounded-2xl font-bold text-lg transition-colors shadow-sm flex items-center justify-center gap-2">
          {#if isProcessing}
            <svg class="animate-spin h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
            Menyimpan...
          {:else}
            Simpan Data
          {/if}
        </button>
      </div>

    </div>
  </div>
{/if}

{#if showBulkActionModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-gray-900/40 backdrop-blur-sm" on:click={closeAllModals}></div>
    
    <div class="bg-white w-full max-w-[500px] rounded-[30px] relative shadow-2xl flex flex-col z-10 animate-in fade-in zoom-in-95 duration-200 overflow-hidden">
      
      <div class="px-8 pt-8 pb-4 flex-shrink-0 relative">
        <h2 class="text-2xl font-bold text-gray-600 text-center">
          {currentBulkAction === 'pindah_kelas' ? 'Pindah Kelas Massal' : 
           currentBulkAction === 'naik_kelas' ? 'Naik Kelas Massal' : 'Ubah Status Massal'}
        </h2>
        <button on:click={closeAllModals} class="absolute top-8 right-8 text-gray-400 hover:text-gray-600 transition-colors bg-gray-100 p-2 rounded-full">
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg>
        </button>
      </div>

      <div class="px-8 py-2 overflow-y-auto flex-1 custom-scrollbar">
        <div class="flex flex-col gap-5 pb-4">
          <p class="text-gray-500 text-center text-sm mb-2">Aksi ini akan diterapkan pada <strong>{selectedIds.length}</strong> siswa yang dipilih.</p>
          
          {#if currentBulkAction === 'pindah_kelas' || currentBulkAction === 'naik_kelas'}
            {#if currentBulkAction === 'naik_kelas'}
              <div>
                <label class="text-gray-600 font-semibold mb-2 block ml-1">Pilih Tingkat Tujuan</label>
                <select bind:value={bulkActionTargetTingkat} class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 font-medium transition-all">
                  <option value="KB">KB</option>
                  <option value="TK A">TK A</option>
                  <option value="TK B">TK B</option>
                </select>
              </div>
            {/if}
            <div>
              <label class="text-gray-600 font-semibold mb-2 block ml-1">Pilih Kelas Tujuan</label>
              <select bind:value={bulkActionTargetKelasId} class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 font-medium transition-all">
                <option value="">-- Kosongkan Kelas --</option>
                {#each dataKelas as k}
                  <option value={k.id}>{k.nama_kelas} ({k.tahun_ajaran})</option>
                {/each}
              </select>
              <p class="text-xs text-gray-400 mt-2 ml-1">*Catatan: Data pilihan kelas di dropdown ini berdasarkan tingkat halaman yang Anda buka sekarang.</p>
            </div>
          {:else if currentBulkAction === 'ubah_status'}
            <div>
              <label class="text-gray-600 font-semibold mb-2 block ml-1">Pilih Status Baru</label>
              <select bind:value={bulkActionTargetStatus} class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 font-medium transition-all">
                <option value="Aktif">Aktif</option>
                <option value="Tidak Aktif">Tidak Aktif</option>
                <option value="Lulus">Lulus</option>
                <option value="Pindah">Pindah</option>
              </select>
            </div>
          {/if}
          
        </div>
      </div>

      <div class="px-8 pb-8 pt-2 flex-shrink-0 bg-white">
        <button on:click={handleBulkAction} disabled={isProcessing} class="w-full py-4 bg-[#2da76b] hover:bg-[#289562] text-white rounded-2xl font-bold text-lg transition-colors shadow-sm flex items-center justify-center gap-2">
          {#if isProcessing}
            <svg class="animate-spin h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
            Memproses...
          {:else}
            Terapkan Aksi
          {/if}
        </button>
      </div>

    </div>
  </div>
{/if}

{#if showDeleteModal}
  {/if}