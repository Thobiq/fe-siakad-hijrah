<script lang="ts">
  import { PUBLIC_API_URL } from '$env/static/public';
  
  export let data = []; 
  export let dataGuru = [];
  export let isLoading = false;
  export let tingkat = "";

  // --- STATE PENCARIAN ---
  let searchQuery = "";

  // --- LOGIKA SORTING ---
  let sortColumn = null;
  let sortOrder = 'asc';

  function handleSort(column) {
    if (sortColumn === column) {
      sortOrder = sortOrder === 'asc' ? 'desc' : 'asc';
    } else {
      sortColumn = column;
      sortOrder = 'asc';
    }
  }

  // --- REACTIVE DATA ---
  $: displayedData = data
    .filter(item => {
      if (!searchQuery) return true;
      const term = searchQuery.toLowerCase();
      return (
        item.nama_kelas?.toLowerCase().includes(term) ||
        item.tahun_ajaran?.toLowerCase().includes(term) ||
        item.wali_kelas?.nama?.toLowerCase().includes(term)
      );
    })
    .sort((a, b) => {
      if (!sortColumn) return 0;
      let valA = sortColumn === 'wali_kelas' ? (a.wali_kelas?.nama || "") : (a[sortColumn] || "");
      let valB = sortColumn === 'wali_kelas' ? (b.wali_kelas?.nama || "") : (b[sortColumn] || "");
      
      if (typeof valA === 'string') valA = valA.toLowerCase();
      if (typeof valB === 'string') valB = valB.toLowerCase();

      if (valA < valB) return sortOrder === 'asc' ? -1 : 1;
      if (valA > valB) return sortOrder === 'asc' ? 1 : -1;
      return 0;
    });

  // --- STATE MODAL ---
  let showModal = false;
  let showDeleteModal = false;
  let activeItem = null;
  let isEditMode = false;

  // Form State
  let formNamaKelas = "";
  let formTahunAjaran = "";
  let formGuruId = "";

  let isProcessing = false;

  function resetForm() {
    formNamaKelas = ""; 
    formTahunAjaran = ""; 
    formGuruId = "";
  }

  function openTambahModal() {
    resetForm();
    isEditMode = false;
    showModal = true;
  }

  function openEditModal(item) {
    activeItem = item;
    formNamaKelas = item.nama_kelas; 
    formTahunAjaran = item.tahun_ajaran; 
    formGuruId = item.guru_id || "";
    isEditMode = true;
    showModal = true;
  }

  function openDeleteModal(item) {
    activeItem = item;
    showDeleteModal = true;
  }

  function closeAllModals() {
    showModal = false;
    showDeleteModal = false;
    activeItem = null;
    isProcessing = false;
  }

  // --- API HANDLERS ---
  async function handleSimpanKelas() {
    if (!formNamaKelas || !formTahunAjaran) {
      alert("Semua kolom wajib diisi kecuali Wali Kelas!"); return;
    }
    
    isProcessing = true;
    const token = localStorage.getItem('auth_token');
    const url = isEditMode ? `${PUBLIC_API_URL}/kelas/${activeItem.id}` : `${PUBLIC_API_URL}/kelas`;
    const method = isEditMode ? 'PUT' : 'POST';

    const payload = {
        tingkat: tingkat,
        nama_kelas: formNamaKelas,
        tahun_ajaran: formTahunAjaran,
        guru_id: formGuruId || null
    };

    try {
      const response = await fetch(url, {
        method: method,
        headers: {
          'Content-Type': 'application/json', 'Accept': 'application/json', 'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify(payload)
      });

      const result = await response.json();
      if (response.ok && result.status) {
        if (isEditMode) {
          data = data.map(d => d.id === activeItem.id ? result.data : d);
        } else {
          data = [...data, result.data];
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
      const response = await fetch(`${PUBLIC_API_URL}/kelas/${activeItem.id}`, {
        method: 'DELETE',
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      const result = await response.json();
      if (response.ok && result.status) {
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
</script>

<div class="bg-white rounded-3xl shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100 p-6 md:p-8 flex flex-col gap-6">
  
  <div class="flex flex-col md:flex-row md:items-center justify-between gap-4">
    <div class="flex items-center gap-6">
      <button on:click={openTambahModal} class="bg-[#2da76b] hover:bg-[#289562] text-white px-5 py-2.5 rounded-xl font-bold transition-colors flex items-center gap-2 shadow-sm disabled:opacity-50">
        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><line x1="12" x2="12" y1="5" y2="19"/><line x1="5" x2="19" y1="12" y2="12"/></svg>
        Tambah Kelas
      </button>
    </div>

    <div class="relative w-full md:w-[300px]">
      <input type="text" bind:value={searchQuery} placeholder="Cari kelas..." class="w-full pl-5 pr-10 py-2.5 rounded-xl border border-gray-300 focus:ring-2 focus:ring-[#2da76b] focus:border-[#2da76b] outline-none text-gray-700 transition-all"/>
      <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-400"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.3-4.3"/></svg>
    </div>
  </div>

  <div class="rounded-2xl overflow-x-auto border border-gray-100">
    <table class="w-full text-left border-collapse whitespace-nowrap">
      <thead>
        <tr class="bg-[#2da76b] text-white">
          <th class="px-6 py-4 font-semibold text-sm cursor-pointer hover:bg-[#289562] transition-colors select-none group" on:click={() => handleSort('nama_kelas')}>
            <div class="flex items-center gap-2">Nama Kelas <span class="text-[10px] opacity-50 group-hover:opacity-100 transition-opacity">↑↓</span></div>
          </th>
          <th class="px-6 py-4 font-semibold text-sm cursor-pointer hover:bg-[#289562] transition-colors select-none group" on:click={() => handleSort('tahun_ajaran')}>
            <div class="flex items-center gap-2">Tahun Ajaran <span class="text-[10px] opacity-50 group-hover:opacity-100 transition-opacity">↑↓</span></div>
          </th>
          <th class="px-6 py-4 font-semibold text-sm cursor-pointer hover:bg-[#289562] transition-colors select-none group" on:click={() => handleSort('wali_kelas')}>
            <div class="flex items-center gap-2">Wali Kelas <span class="text-[10px] opacity-50 group-hover:opacity-100 transition-opacity">↑↓</span></div>
          </th>
          <th class="px-6 py-4 font-semibold text-sm text-center">Aksi</th>
        </tr>
      </thead>
      <tbody>
        {#if isLoading}
          <tr>
            <td colspan="5" class="px-6 py-8 text-center text-gray-500 font-medium">
              <div class="flex items-center justify-center gap-3">
                <svg class="animate-spin h-6 w-6 text-[#2da76b]" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                Memuat data...
              </div>
            </td>
          </tr>
        {:else}
          {#each displayedData as row}
            <tr class="odd:bg-white even:bg-gray-50 hover:bg-green-50/50 transition-colors">
              <td class="px-6 py-4 font-bold text-gray-700">{row.nama_kelas}</td>
              <td class="px-6 py-4 text-gray-600">{row.tahun_ajaran}</td>
              <td class="px-6 py-4 font-medium text-[#2da76b]">{row.wali_kelas ? row.wali_kelas.nama : '-'}</td>
              <td class="px-6 py-4 flex items-center justify-center gap-2">
                <button class="p-2 bg-amber-500 hover:bg-amber-600 text-white rounded-lg shadow-sm transition-colors" on:click={() => openEditModal(row)}>
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/><path d="m15 5 4 4"/></svg>
                </button>
                <button class="p-2 bg-red-500 hover:bg-red-600 text-white rounded-lg shadow-sm transition-colors" on:click={() => openDeleteModal(row)}>
                    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/></svg>
                </button>
              </td>
            </tr>
          {/each}
          {#if displayedData.length === 0}
            <tr>
              <td colspan="4" class="px-6 py-8 text-center text-gray-400 font-medium">Tidak ada data kelas ditemukan.</td>
            </tr>
          {/if}
        {/if}
      </tbody>
    </table>
  </div>
</div>

{#if showModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-gray-900/40 backdrop-blur-sm" on:click={closeAllModals}></div>
    
    <div class="bg-white w-full max-w-[500px] rounded-[30px] relative shadow-2xl flex flex-col z-10 animate-in fade-in zoom-in-95 duration-200 overflow-hidden">
      
      <div class="px-8 pt-8 pb-4 flex-shrink-0 relative">
        <h2 class="text-2xl font-bold text-gray-600 text-center">{isEditMode ? 'Edit' : 'Tambah'} Kelas</h2>
        <button on:click={closeAllModals} class="absolute top-8 right-8 text-gray-400 hover:text-gray-600 transition-colors bg-gray-100 p-2 rounded-full">
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg>
        </button>
      </div>

      <div class="px-8 py-2 overflow-y-auto flex-1 custom-scrollbar">
        <div class="flex flex-col gap-4 pb-4">
          
          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Nama Kelas</label>
            <input type="text" bind:value={formNamaKelas} placeholder="Misal: Kelas KB 1" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>
          
          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Tahun Ajaran</label>
            <input type="text" bind:value={formTahunAjaran} placeholder="Misal: 2025-2026" class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-500 font-medium transition-all"/>
          </div>

          <div>
            <label class="text-gray-600 font-semibold mb-2 block ml-1">Wali Kelas (Opsional)</label>
            <select bind:value={formGuruId} class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 font-medium transition-all appearance-none">
              <option value="">-- Pilih Guru --</option>
              {#each dataGuru as guru}
                <option value={guru.id}>{guru.nama} ({guru.nomor_induk})</option>
              {/each}
            </select>
          </div>
          
        </div>
      </div>

      <div class="px-8 pb-8 pt-2 flex-shrink-0 bg-white">
        <button on:click={handleSimpanKelas} disabled={isProcessing} class="w-full py-4 {isEditMode ? 'bg-amber-500 hover:bg-amber-600' : 'bg-[#2da76b] hover:bg-[#289562]'} text-white rounded-2xl font-bold text-lg transition-colors shadow-sm flex items-center justify-center gap-2">
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

{#if showDeleteModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-gray-900/40 backdrop-blur-sm transition-opacity" on:click={closeAllModals}></div>
    <div class="bg-white rounded-3xl p-8 max-w-sm w-full shadow-2xl relative z-10 flex flex-col items-center">
      <div class="w-16 h-16 bg-red-100 text-red-500 rounded-full flex items-center justify-center mb-4">
        <svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="m21.73 18-8-14a2 2 0 0 0-3.48 0l-8 14A2 2 0 0 0 4 21h16a2 2 0 0 0 1.73-3Z"/><line x1="12" x2="12" y1="9" y2="13"/><line x1="12" x2="12.01" y1="17" y2="17"/></svg>
      </div>
      <h3 class="text-xl font-bold text-gray-800 mb-2">Hapus Kelas?</h3>
      <p class="text-gray-500 text-center mb-8">Apakah Anda yakin ingin menghapus <strong>{activeItem?.nama_kelas}</strong>?</p>
      <div class="flex gap-3 w-full">
        <button class="flex-1 py-3 px-4 bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold rounded-xl transition-colors" on:click={closeAllModals}>Batal</button>
        <button class="flex-1 py-3 px-4 bg-red-500 hover:bg-red-600 text-white font-bold rounded-xl transition-colors shadow-sm" on:click={confirmDelete}>
          {#if isProcessing}
            Menghapus...
          {:else}
            Ya, Hapus
          {/if}
        </button>
      </div>
    </div>
  </div>
{/if}
