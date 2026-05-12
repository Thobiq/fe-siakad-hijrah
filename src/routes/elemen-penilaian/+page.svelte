<script lang="ts">
  import Sidebar from '$lib/components/Sidebar.svelte';
  import { onMount } from 'svelte';
  import { PUBLIC_API_URL } from '$env/static/public';
  import { goto } from '$app/navigation';

  // --- STATE UTAMA ---
  let isSidebarOpen = false;
  let userName = "Bu Hijrah";
  let isLoading = true;

  // --- STATE DATA ELEMEN ---
  let daftarElemen = [];
  let searchQuery = "";

  // --- STATE MODAL ---
  let showFormModal = false;
  let showDeleteModal = false;
  let modalMode = 'add'; // 'add' atau 'edit'
  let activeItem = null;
  
  // State Input Form
  let inputNamaElemen = "";

  // --- FILTER PENCARIAN ---
  $: filteredElemen = daftarElemen.filter(el => 
      el.nama_elemen.toLowerCase().includes(searchQuery.toLowerCase())
  );

  onMount(async () => {
    const userData = JSON.parse(localStorage.getItem('user_data') || '{}');
    if (userData.name) userName = userData.name;
    
    await fetchElemen();
  });

  // --- FUNGSI FETCH DATA ---
  async function fetchElemen() {
    isLoading = true;
    const token = localStorage.getItem('auth_token');
    try {
      // Pastikan kamu punya endpoint ini di Laravel ya!
      const response = await fetch(`${PUBLIC_API_URL}/elemen-capaian`, {
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      const result = await response.json();
      if (result.status) {
        daftarElemen = result.data;
      }
    } catch (error) {
      console.error("Gagal memuat data elemen:", error);
    } finally {
      isLoading = false;
    }
  }

  // --- KONTROL MODAL FORM (TAMBAH/EDIT) ---
  function openAddModal() {
    modalMode = 'add';
    inputNamaElemen = "";
    activeItem = null;
    showFormModal = true;
  }

  function openEditModal(item) {
    modalMode = 'edit';
    activeItem = item;
    inputNamaElemen = item.nama_elemen;
    showFormModal = true;
  }

  async function handleSaveElemen() {
    if (!inputNamaElemen.trim()) {
      alert("Nama elemen tidak boleh kosong!");
      return;
    }

    const token = localStorage.getItem('auth_token');
    const url = modalMode === 'add' 
      ? `${PUBLIC_API_URL}/elemen-capaian` 
      : `${PUBLIC_API_URL}/elemen-capaian/${activeItem.id}`;
    
    const method = modalMode === 'add' ? 'POST' : 'PUT';

    try {
      const response = await fetch(url, {
        method: method,
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
          'Accept': 'application/json'
        },
        body: JSON.stringify({ nama_elemen: inputNamaElemen })
      });

      const result = await response.json();

      if (response.ok && result.status) {
        showFormModal = false;
        await fetchElemen(); // Refresh data setelah sukses
      } else {
        alert(result.message || "Gagal menyimpan data.");
      }
    } catch (error) {
      alert("Terjadi kesalahan pada server.");
    }
  }

  // --- KONTROL MODAL HAPUS ---
  function openDeleteModal(item) {
    activeItem = item;
    showDeleteModal = true;
  }

  async function confirmDelete() {
    const token = localStorage.getItem('auth_token');
    try {
      const response = await fetch(`${PUBLIC_API_URL}/elemen-capaian/${activeItem.id}`, {
        method: 'DELETE',
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      
      const result = await response.json();
      if (response.ok && result.status) {
        showDeleteModal = false;
        await fetchElemen(); // Refresh data
      } else {
        alert("Gagal menghapus: " + result.message);
      }
    } catch (error) {
      alert("Gagal terhubung ke server.");
    }
  }
</script>

<svelte:head>
  <title>Manajemen Elemen Penilaian - SIAKAD</title>
</svelte:head>

<div class="flex h-screen bg-gray-50 overflow-hidden font-sans">
  <Sidebar bind:isOpen={isSidebarOpen} />

  <div class="flex-1 flex flex-col overflow-hidden w-full">
    
    <header class="h-20 bg-white flex items-center justify-between px-6 md:px-10 shrink-0 border-b border-gray-100">
      <div class="flex items-center gap-4">
        <button class="md:hidden p-2 text-gray-500" on:click={() => isSidebarOpen = true}>
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><line x1="4" x2="20" y1="12" y2="12"/><line x1="4" x2="20" y1="6" y2="6"/><line x1="4" x2="20" y1="18" y2="18"/></svg>
        </button>
        <h1 class="text-xl font-bold text-gray-700 hidden md:block">Master Data Elemen</h1>
      </div>
      <div class="flex items-center gap-3">
        <span class="font-bold text-gray-600">{userName}</span>
        <div class="w-10 h-10 rounded-full bg-[#2da76b] text-white flex items-center justify-center shrink-0">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="8" r="5"/><path d="M20 21a8 8 0 0 0-16 0"/></svg>
        </div>
      </div>
    </header>

    <main class="flex-1 overflow-y-auto p-4 md:p-8">
      <div class="bg-white rounded-3xl border border-gray-200 shadow-sm p-6 md:p-8 flex flex-col gap-6 max-w-5xl mx-auto">
        
        <div class="flex flex-col md:flex-row md:items-center justify-between gap-4">
          <button on:click={openAddModal} class="bg-[#2da76b] hover:bg-[#289562] text-white px-5 py-2.5 rounded-xl font-bold transition-colors flex items-center justify-center gap-2 shadow-sm shrink-0">
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><line x1="12" x2="12" y1="5" y2="19"/><line x1="5" x2="19" y1="12" y2="12"/></svg>
            Tambah Elemen
          </button>

          <div class="relative w-full md:w-[300px]">
            <input type="text" bind:value={searchQuery} placeholder="Cari elemen..." class="w-full pl-5 pr-10 py-2.5 rounded-xl border border-gray-300 focus:ring-2 focus:ring-[#2da76b] focus:border-[#2da76b] outline-none text-gray-700 transition-all"/>
            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="absolute right-4 top-1/2 -translate-y-1/2 text-gray-400"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.3-4.3"/></svg>
          </div>
        </div>

        <div class="rounded-2xl overflow-x-auto border border-gray-100 mt-2">
          <table class="w-full text-left border-collapse whitespace-nowrap">
            <thead>
              <tr class="bg-[#2da76b] text-white">
                <th class="px-6 py-4 font-semibold text-sm w-16 text-center">No</th>
                <th class="px-6 py-4 font-semibold text-sm">Nama Elemen Penilaian</th>
                <th class="px-6 py-4 font-semibold text-sm text-right">Aksi</th>
              </tr>
            </thead>
            <tbody>
              {#if isLoading}
                <tr><td colspan="3" class="px-6 py-8 text-center text-gray-500 font-semibold">Memuat data...</td></tr>
              {:else if filteredElemen.length === 0}
                <tr><td colspan="3" class="px-6 py-8 text-center text-gray-500 font-semibold">Tidak ada elemen yang ditemukan.</td></tr>
              {:else}
                {#each filteredElemen as elemen, index}
                    <tr 
                        class="odd:bg-white even:bg-gray-50 hover:bg-green-50/50 transition-colors cursor-pointer group"
                        on:click={() => goto(`/elemen-penilaian/${elemen.id}`)}
                    >
                        <td class="px-6 py-4 font-bold text-gray-500 text-center">{index + 1}</td>
                        
                        <td class="px-6 py-4 font-medium text-gray-700">
                        {elemen.nama_elemen}
                        </td>

                        <td class="px-6 py-4 text-right flex items-center justify-end gap-2">
                        <button 
                            on:click|stopPropagation={() => openEditModal(elemen)} 
                            class="p-2 bg-blue-50 text-blue-600 hover:bg-blue-600 hover:text-white rounded-lg transition-colors shadow-sm"
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/><path d="m15 5 4 4"/></svg>
                        </button>
                        
                        <button 
                            on:click|stopPropagation={() => openDeleteModal(elemen)} 
                            class="p-2 bg-red-50 text-red-500 hover:bg-red-500 hover:text-white rounded-lg transition-colors shadow-sm"
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/></svg>
                        </button>
                        </td>
                    </tr>
                {/each}
              {/if}
            </tbody>
          </table>
        </div>

      </div>
    </main>
  </div>
</div>

{#if showFormModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-gray-900/40 backdrop-blur-sm" on:click={() => showFormModal = false}></div>
    <div class="bg-white w-full max-w-md rounded-[30px] p-8 relative shadow-2xl flex flex-col gap-6 z-10 animate-in fade-in zoom-in-95 duration-200">
      
      <button class="absolute top-5 right-5 text-gray-400 hover:text-gray-600 bg-gray-100 hover:bg-gray-200 p-2 rounded-full transition-colors" on:click={() => showFormModal = false}>
        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg>
      </button>

      <div class="flex items-center gap-4 mb-2">
        <div class="w-12 h-12 bg-green-100 text-[#2da76b] rounded-2xl flex items-center justify-center shrink-0">
          <!-- <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M12 2v20"/><path d="M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"/></svg> -->
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                <rect width="8" height="4" x="8" y="2" rx="1" ry="1"/>
                <path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2"/>
                <path d="m9 14 2 2 4-4"/>
            </svg>
        </div>
        <h2 class="text-2xl font-bold text-gray-700">{modalMode === 'add' ? 'Tambah Elemen' : 'Edit Elemen'}</h2>
      </div>
      
      <div>
        <label class="text-gray-600 font-semibold mb-2 block ml-1 text-sm" for="nama_elemen">Nama Elemen Penilaian</label>
        <input type="text" id="nama_elemen" bind:value={inputNamaElemen} placeholder="Contoh: Nilai Agama dan Budi Pekerti" class="w-full px-5 py-4 rounded-2xl bg-gray-50 border-2 border-gray-100 outline-none focus:border-[#2da76b] text-gray-800 placeholder:text-gray-400 font-medium transition-colors"/>
      </div>
      
      <button on:click={handleSaveElemen} class="w-full py-4 mt-2 bg-[#2da76b] text-white rounded-2xl font-bold text-lg hover:bg-[#289562] transition-colors shadow-sm">
        {modalMode === 'add' ? 'Simpan Data' : 'Perbarui Data'}
      </button>
    </div>
  </div>
{/if}

{#if showDeleteModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-gray-900/40 backdrop-blur-sm" on:click={() => showDeleteModal = false}></div>
    <div class="bg-white rounded-[30px] p-8 max-w-sm w-full shadow-2xl relative z-10 flex flex-col items-center animate-in fade-in zoom-in-95 duration-200">
      <div class="w-16 h-16 bg-red-50 text-red-500 rounded-full flex items-center justify-center mb-4">
        <svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="m21.73 18-8-14a2 2 0 0 0-3.48 0l-8 14A2 2 0 0 0 4 21h16a2 2 0 0 0 1.73-3Z"/><line x1="12" x2="12" y1="9" y2="13"/><line x1="12" x2="12.01" y1="17" y2="17"/></svg>
      </div>
      <h3 class="text-xl font-bold text-gray-800 mb-2">Hapus Elemen?</h3>
      <p class="text-gray-500 text-center mb-8 text-sm leading-relaxed">
        Yakin ingin menghapus elemen <strong class="text-gray-700">{activeItem?.nama_elemen}</strong>? Data yang dihapus tidak dapat dikembalikan.
      </p>
      <div class="flex gap-3 w-full">
        <button class="flex-1 py-3 px-4 bg-gray-100 hover:bg-gray-200 text-gray-600 font-bold rounded-xl transition-colors" on:click={() => showDeleteModal = false}>Batal</button>
        <button class="flex-1 py-3 px-4 bg-red-500 hover:bg-red-600 text-white font-bold rounded-xl transition-colors shadow-sm" on:click={confirmDelete}>Ya, Hapus</button>
      </div>
    </div>
  </div>
{/if}