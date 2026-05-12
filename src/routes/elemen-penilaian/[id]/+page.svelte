<script lang="ts">
  import { page } from '$app/stores';
  import Sidebar from '$lib/components/Sidebar.svelte';
  import { onMount } from 'svelte';
  import { PUBLIC_API_URL } from '$env/static/public';

  let isSidebarOpen = false;
  let userName = "Bu Hijrah";
  let elemenId = $page.params.id;
  let isLoading = true;

  // Data utama
  let elemenData = { nama_elemen: "Loading...", capaian_pembelajarans: [] };

  // State Modal (Generic untuk CP, TP, ATP)
  let showModal = false;
  let modalMode = 'add'; // 'add' atau 'edit'
  let modalType = 'CP'; // 'CP', 'TP', 'ATP'
  let activeParentId = null; // Menyimpan ID parent saat tambah data baru
  let activeItem = null; // Menyimpan item yang sedang diedit
  let inputText = "";

  onMount(async () => {
    const userData = JSON.parse(localStorage.getItem('user_data') || '{}');
    if (userData.name) userName = userData.name;
    await fetchDetailElemen();
  });

  async function fetchDetailElemen() {
    isLoading = true;
    const token = localStorage.getItem('auth_token');
    try {
      // Pastikan API ini me-return elemen beserta relasi CP -> TP -> ATP
      const res = await fetch(`${PUBLIC_API_URL}/elemen-capaian/${elemenId}`, {
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      const result = await res.json();
      if (result.status) elemenData = result.data;
    } catch (error) {
      console.error(error);
    } finally {
      isLoading = false;
    }
  }

  // --- KONTROL MODAL ---
  function openAddModal(type, parentId = null) {
    modalMode = 'add';
    modalType = type;
    activeParentId = parentId;
    activeItem = null;
    inputText = "";
    showModal = true;
  }

  function openEditModal(type, item) {
    modalMode = 'edit';
    modalType = type;
    activeItem = item;
    inputText = item.deskripsi;
    showModal = true;
  }

  // --- LOGIKA SIMPAN (MOCKUP) ---
  async function handleSave() {
    if (!inputText.trim()) return alert("Deskripsi tidak boleh kosong!");
    
    const token = localStorage.getItem('auth_token');
    
    // Tentukan Endpoint (URL)
    let url = `${PUBLIC_API_URL}/`;
    if (modalType === 'CP') url += `cp`;
    if (modalType === 'TP') url += `tp`;
    if (modalType === 'ATP') url += `atp`;

    if (modalMode === 'edit') url += `/${activeItem.id}`; // Tambah ID jika edit

    // Tentukan Payload (Body)
    let payload = { deskripsi: inputText };
    if (modalMode === 'add') {
        if (modalType === 'CP') payload.elemen_capaian_id = elemenId;
        if (modalType === 'TP') payload.capaian_pembelajaran_id = activeParentId;
        if (modalType === 'ATP') payload.tujuan_pembelajaran_id = activeParentId;
    }

    try {
        const response = await fetch(url, {
            method: modalMode === 'add' ? 'POST' : 'PUT',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${token}`,
                'Accept': 'application/json'
            },
            body: JSON.stringify(payload)
        });

        const result = await response.json();
        
        if (result.status) {
            showModal = false;
            await fetchDetailElemen(); // Refresh struktur dari backend!
        } else {
            alert(result.message);
        }
    } catch (error) {
        alert("Gagal terhubung ke server.");
    }
  }

  // --- LOGIKA HAPUS (MOCKUP) ---
  async function handleDelete(type, item) {
    if (!confirm(`Yakin ingin menghapus ${type} ini? Semua data di bawahnya akan ikut terhapus.`)) return;
    
    const token = localStorage.getItem('auth_token');
    let url = `${PUBLIC_API_URL}/${type.toLowerCase()}/${item.id}`; // menjadi /cp/1, /tp/2, dst

    try {
        const response = await fetch(url, {
            method: 'DELETE',
            headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
        });
        
        const result = await response.json();
        if (result.status) {
            await fetchDetailElemen(); // Refresh hierarki dari backend
        } else {
            alert(result.message);
        }
    } catch(error) {
        alert("Gagal terhubung ke server.");
    }
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
        <a href="/elemen-penilaian" class="flex items-center gap-2 text-gray-500 hover:text-[#2da76b] font-semibold transition-colors">
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
      <div class="max-w-5xl mx-auto flex flex-col gap-6">
        
        <div class="bg-white rounded-3xl border border-gray-200 shadow-sm p-6 flex justify-between items-center">
          <div>
            <p class="text-sm font-bold text-gray-400 mb-1">Struktur Elemen Penilaian</p>
            <h1 class="text-2xl font-bold text-gray-700">{elemenData.nama_elemen}</h1>
          </div>
          <button on:click={() => openAddModal('CP', elemenId)} class="bg-blue-600 hover:bg-blue-700 text-white px-5 py-2.5 rounded-xl font-bold shadow-sm transition-colors text-sm">
            + Tambah Capaian (CP)
          </button>
        </div>

        {#if isLoading}
          <div class="text-center p-10 text-gray-500 font-bold">Memuat struktur kurikulum...</div>
        {:else}
          {#each elemenData.capaian_pembelajarans as cp}
            <div class="bg-white rounded-2xl border border-blue-200 shadow-sm overflow-hidden">
              <div class="bg-blue-50 p-4 border-b border-blue-200 flex justify-between items-start gap-4">
                <div class="flex-1">
                  <span class="inline-block px-2 py-1 bg-blue-200 text-blue-800 text-[10px] font-black rounded mb-2">CAPAIAN PEMBELAJARAN (CP)</span>
                  <p class="font-semibold text-gray-700 leading-relaxed text-sm">{cp.deskripsi}</p>
                </div>
                <div class="flex gap-2 shrink-0">
                  <button on:click={() => openEditModal('CP', cp)} class="p-1.5 text-blue-500 hover:bg-blue-100 rounded"><svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/></svg></button>
                  <button on:click={() => handleDelete('CP', cp)} class="p-1.5 text-red-500 hover:bg-red-100 rounded"><svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/></svg></button>
                </div>
              </div>

              <div class="p-4 flex flex-col gap-4">
                {#each cp.tujuan_pembelajarans as tp}
                  <div class="border border-green-200 rounded-xl overflow-hidden ml-4 md:ml-8">
                    <div class="bg-green-50 p-3 border-b border-green-200 flex justify-between items-start gap-4">
                      <div class="flex-1">
                        <span class="inline-block px-2 py-1 bg-green-200 text-green-800 text-[10px] font-black rounded mb-1">TUJUAN PEMBELAJARAN (TP)</span>
                        <p class="font-medium text-gray-700 text-sm">{tp.deskripsi}</p>
                      </div>
                      <div class="flex gap-2 shrink-0">
                        <button on:click={() => openEditModal('TP', tp)} class="p-1.5 text-blue-500 hover:bg-blue-100 rounded"><svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/></svg></button>
                        <button on:click={() => handleDelete('TP', tp)} class="p-1.5 text-red-500 hover:bg-red-100 rounded"><svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/><line x1="10" x2="10" y1="11" y2="17"/><line x1="14" x2="14" y1="11" y2="17"/></svg></button>
                      </div>
                    </div>

                    <div class="p-3 bg-white">
                      <ul class="flex flex-col gap-2">
                        {#each tp.atp_indikators as atp}
                          <li class="flex justify-between items-start gap-3 p-2 hover:bg-gray-50 rounded-lg border border-transparent hover:border-gray-100 transition-colors group">
                            <div class="flex items-start gap-2 flex-1">
                              <div class="w-1.5 h-1.5 rounded-full bg-orange-400 mt-2 shrink-0"></div>
                              <p class="text-sm text-gray-600">{atp.deskripsi}</p>
                            </div>
                            <div class="flex gap-1 opacity-0 group-hover:opacity-100 transition-opacity shrink-0">
                               <button on:click={() => openEditModal('ATP', atp)} class="p-1 text-blue-500 hover:bg-blue-50 rounded"><svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/></svg></button>
                               <button on:click={() => handleDelete('ATP', atp)} class="p-1 text-red-500 hover:bg-red-50 rounded"><svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg></button>
                            </div>
                          </li>
                        {/each}
                      </ul>
                      <button on:click={() => openAddModal('ATP', tp.id)} class="mt-3 text-xs font-bold text-orange-500 hover:text-orange-600 flex items-center gap-1 ml-3">
                        + Tambah ATP/Indikator Baru
                      </button>
                    </div>
                  </div>
                {/each}
                
                <button on:click={() => openAddModal('TP', cp.id)} class="ml-4 md:ml-8 py-2 border-2 border-dashed border-green-200 text-green-600 rounded-xl text-sm font-bold hover:bg-green-50 transition-colors">
                  + Tambah Tujuan Pembelajaran (TP)
                </button>
              </div>
            </div>
          {/each}
        {/if}

      </div>
    </main>
  </div>
</div>

{#if showModal}
  <div class="fixed inset-0 z-[100] flex items-center justify-center p-4">
    <div class="absolute inset-0 bg-gray-900/40 backdrop-blur-sm" on:click={() => showModal = false}></div>
    <div class="bg-white w-full max-w-lg rounded-[30px] p-8 relative shadow-2xl flex flex-col gap-4 z-10">
      <h2 class="text-xl font-bold text-gray-700">
        {modalMode === 'add' ? 'Tambah' : 'Edit'} {modalType === 'CP' ? 'Capaian Pembelajaran' : modalType === 'TP' ? 'Tujuan Pembelajaran' : 'ATP/Indikator'}
      </h2>
      <p class="text-xs text-gray-500 mb-2">Pastikan deskripsi ditulis dengan jelas dan terukur sesuai standar kurikulum.</p>
      
      <textarea 
        bind:value={inputText} 
        rows="4"
        placeholder="Tulis deskripsi di sini..." 
        class="w-full p-4 rounded-2xl bg-gray-50 border-2 border-gray-100 outline-none focus:border-[#2da76b] text-gray-700 text-sm"
      ></textarea>
      
      <div class="flex gap-3 mt-4">
        <button class="flex-1 py-3 bg-gray-100 text-gray-600 font-bold rounded-xl" on:click={() => showModal = false}>Batal</button>
        <button class="flex-1 py-3 bg-[#2da76b] text-white font-bold rounded-xl" on:click={handleSave}>Simpan</button>
      </div>
    </div>
  </div>
{/if}