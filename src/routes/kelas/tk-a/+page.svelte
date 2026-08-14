<script lang="ts">
  import Sidebar from '$lib/components/Sidebar.svelte';
  import TableKelas from '$lib/components/TableKelas.svelte';
  import { onMount } from 'svelte';
  import { PUBLIC_API_URL } from '$env/static/public';
  
  let isSidebarOpen = false;
  let userName = "Bu Hijrah";

  let dataKelasTKA = [];
  let dataGuru = [];
  let isLoading = true;

  onMount(async () => {
    const token = localStorage.getItem('auth_token');
    const userData = JSON.parse(localStorage.getItem('user_data') || '{}');
    if (userData.name) userName = userData.name;

    try {
      // Ambil data guru
      const guruRes = await fetch(`${PUBLIC_API_URL}/guru`, {
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      const guruResult = await guruRes.json();
      if (guruRes.ok && guruResult.status) {
        dataGuru = guruResult.data;
      }

      // Ambil data kelas
      const kelasRes = await fetch(`${PUBLIC_API_URL}/kelas?tingkat=TK A`, {
        headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
      });
      const kelasResult = await kelasRes.json();
      if (kelasRes.ok && kelasResult.status) {
        dataKelasTKA = kelasResult.data;
      }
    } catch (error) {
      console.error("Gagal mengambil data kelas:", error);
    } finally {
      isLoading = false;
    }
  });
</script>

<svelte:head>
  <title>Manajemen Kelas TK A - SIAKAD Al Hijrah</title>
</svelte:head>

<div class="flex h-screen bg-gray-50 overflow-hidden font-sans">
  
  <Sidebar bind:isOpen={isSidebarOpen} />

  <div class="flex-1 flex flex-col overflow-hidden w-full relative">
    
    <header class="h-20 bg-white flex items-center justify-between px-6 md:px-10 shrink-0 border-b border-gray-100">
      <div class="flex items-center gap-4">
        <button class="md:hidden p-2 -ml-2 text-gray-500 hover:bg-gray-100 rounded-lg transition" on:click={() => isSidebarOpen = true}>
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="4" x2="20" y1="12" y2="12"/><line x1="4" x2="20" y1="6" y2="6"/><line x1="4" x2="20" y1="18" y2="18"/></svg>
        </button>
        <h1 class="text-xl font-bold text-gray-600">Manajemen Kelas TK A</h1>
      </div>
      <div class="flex items-center gap-3">
        <span class="font-bold text-gray-600 hidden sm:block">{userName}</span>
        <div class="w-10 h-10 rounded-full bg-[#2da76b] text-white flex items-center justify-center shrink-0">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="8" r="5"/><path d="M20 21a8 8 0 0 0-16 0"/></svg>
        </div>
      </div>
    </header>

    <main class="flex-1 overflow-y-auto p-0 flex flex-col">
      <TableKelas bind:data={dataKelasTKA} bind:dataGuru={dataGuru} tingkat="TK A" isLoading={isLoading} />
    </main>

  </div>
</div>
