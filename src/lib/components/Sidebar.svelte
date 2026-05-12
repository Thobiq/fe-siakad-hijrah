<script lang="ts">
  // 1. Import store page dari SvelteKit
  import { page } from '$app/stores';
  import {goto} from '$app/navigation';
  import {PUBLIC_API_URL} from '$env/static/public';

  export let isOpen = false;

  // 2. Buat variabel reaktif untuk memantau URL saat ini
  $: currentPath = $page.url.pathname;

  // 3. Logika pintar untuk Dropdown Penilaian: 
  // Jika URL saat ini mengandung '/penilaian', otomatis buka dropdown-nya
  let isPenilaianOpen = false;
  $: {
    if (currentPath.startsWith('/penilaian')) {
      isPenilaianOpen = true;
    }
  }

  function closeSidebar() {
    isOpen = false;
  }

  async function handleLogout() {
    const token = localStorage.getItem('auth_token');
    
    if (token) {
      try {
        // Beritahu Laravel untuk menghapus token ini dari database (Biar aman dari hacker)
        await fetch(`${PUBLIC_API_URL}/logout`, {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${token}`,
            'Accept': 'application/json'
          }
        });
      } catch (error) {
        console.error('Gagal menghubungi server logout', error);
      }
    }

    // Hapus kunci dari browser
    localStorage.removeItem('auth_token');
    localStorage.removeItem('user_data');
    
    // Tendang kembali ke halaman login
    goto('/login');
  }
</script>

{#if isOpen}
  <div class="fixed inset-0 bg-black/50 z-40 md:hidden backdrop-blur-sm transition-opacity" on:click={closeSidebar}></div>
{/if}

<aside class="w-[280px] bg-[#2da76b] text-white flex flex-col shrink-0 rounded-tr-[30px] rounded-br-[30px] shadow-xl z-50 fixed inset-y-0 left-0 transition-transform duration-300 ease-in-out md:relative md:translate-x-0 {isOpen ? 'translate-x-0' : '-translate-x-full'}">
    
    <button class="md:hidden absolute top-6 right-4 text-white/80 hover:text-white bg-black/20 p-1.5 rounded-lg" on:click={closeSidebar}>
      <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
    </button>
    
    <div class="p-6 flex items-center gap-3 border-b border-white/20 mt-2">
      <div class="w-10 h-10 rounded-full bg-yellow-400 overflow-hidden flex items-center justify-center shrink-0">
        <img src="/logo-alhijrah.png" alt="Logo" class="w-full h-full object-cover" />
      </div>
      <h2 class="font-semibold text-lg leading-tight">Yayasan Al-Hijrah</h2>
    </div>

    <nav class="flex-1 px-4 py-8 flex flex-col gap-2 overflow-y-auto">
      
      <a href="/dashboard" class="flex items-center gap-3 px-4 py-3 rounded-xl transition-colors {currentPath.startsWith('/dashboard') ? 'bg-white text-[#2da76b] font-bold shadow-sm' : 'text-white/90 hover:bg-white/10 hover:text-white font-medium'}" on:click={closeSidebar}>
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><rect width="7" height="9" x="3" y="3" rx="1"/><rect width="7" height="5" x="14" y="3" rx="1"/><rect width="7" height="9" x="14" y="12" rx="1"/><rect width="7" height="5" x="3" y="16" rx="1"/></svg>
        Dashboard
      </a>

      <a href="/guru" class="flex items-center gap-3 px-4 py-3 rounded-xl transition-colors {currentPath.startsWith('/guru') ? 'bg-white text-[#2da76b] font-bold shadow-sm' : 'text-white/90 hover:bg-white/10 hover:text-white font-medium'}" on:click={closeSidebar}>
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M19 21v-2a4 4 0 0 0-4-4H9a4 4 0 0 0-4 4v2"/><circle cx="12" cy="7" r="4"/></svg>
        Guru
      </a>

      <a href="/siswa" class="flex items-center gap-3 px-4 py-3 rounded-xl transition-colors {currentPath.startsWith('/siswa') ? 'bg-white text-[#2da76b] font-bold shadow-sm' : 'text-white/90 hover:bg-white/10 hover:text-white font-medium'}" on:click={closeSidebar}>
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M22 10v6M2 10l10-5 10 5-10 5z"/><path d="M6 12v5c3 3 9 3 12 0v-5"/></svg>
        Siswa
      </a>
      <a href="/elemen-penilaian" class="flex items-center gap-3 px-4 py-3 rounded-xl transition-colors {currentPath.startsWith('/elemen-penilaian') ? 'bg-white text-[#2da76b] font-bold shadow-sm' : 'text-white/90 hover:bg-white/10 hover:text-white font-medium'}" on:click={closeSidebar}>
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M22 10v6M2 10l10-5 10 5-10 5z"/><path d="M6 12v5c3 3 9 3 12 0v-5"/></svg>
        Elemen Penilaian
      </a>

      <div class="flex flex-col">
        <button 
          class="flex items-center justify-between w-full px-4 py-3 rounded-xl transition-colors {currentPath.startsWith('/penilaian') ? 'text-white font-bold bg-white/10' : 'text-white/90 hover:bg-white/10 hover:text-white font-medium'}"
          on:click={() => isPenilaianOpen = !isPenilaianOpen}
        >
          <div class="flex items-center gap-3">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2"/><path d="M15 2H9a1 1 0 0 0-1 1v2a1 1 0 0 0 1 1h6a1 1 0 0 0 1-1V3a1 1 0 0 0-1-1Z"/><path d="m9 14 2 2 4-4"/></svg>
            <span>Penilaian</span>
          </div>
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round" stroke-linejoin="round" class="transition-transform duration-300 {isPenilaianOpen ? 'rotate-180' : ''}"><path d="m6 9 6 6 6-6"/></svg>
        </button>

        {#if isPenilaianOpen}
          <div class="flex flex-col gap-1 mt-1 mb-2 ml-[25px] pl-5 border-l-2 border-white/40">
            <a href="/penilaian/kb" class="block px-5 py-2.5 rounded-xl transition-colors {currentPath.startsWith('/penilaian/kb') ? 'bg-white text-[#2da76b] font-bold shadow-sm' : 'text-white/90 hover:bg-white/10 hover:text-white font-bold'}" on:click={closeSidebar}>
              Penilaian KB
            </a>
            <a href="/penilaian/tk-a" class="block px-5 py-2.5 rounded-xl transition-colors {currentPath.startsWith('/penilaian/tk-a') ? 'bg-white text-[#2da76b] font-bold shadow-sm' : 'text-white/90 hover:bg-white/10 hover:text-white font-bold'}" on:click={closeSidebar}>
              Penilaian TK A
            </a>
            <a href="/penilaian/tk-b" class="block px-5 py-2.5 rounded-xl transition-colors {currentPath.startsWith('/penilaian/tk-b') ? 'bg-white text-[#2da76b] font-bold shadow-sm' : 'text-white/90 hover:bg-white/10 hover:text-white font-bold'}" on:click={closeSidebar}>
              Penilaian TK B
            </a>
          </div>
        {/if}
      </div>

      <a href="/e-rapor" class="flex items-center gap-3 px-4 py-3 rounded-xl transition-colors {currentPath.startsWith('/e-rapor') ? 'bg-white text-[#2da76b] font-bold shadow-sm' : 'text-white/90 hover:bg-white/10 hover:text-white font-medium'}" on:click={closeSidebar}>
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M14.5 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7.5L14.5 2z"/><polyline points="14 2 14 8 20 8"/><path d="M10.42 12.61a2.1 2.1 0 1 1 2.97 2.97L7.95 21 4 22l.99-3.95 5.43-5.44Z"/></svg>
        E-Rapor
      </a>
    </nav>

    <div class="p-4 mb-4">
      <button 
        on:click={handleLogout} 
        class="w-full flex items-center gap-3 text-white hover:bg-white/10 px-4 py-3 rounded-xl font-bold transition-colors"
      >
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4"/><polyline points="16 17 21 12 16 7"/><line x1="21" x2="9" y1="12" y2="12"/></svg>
        Logout
      </button>
    </div>
</aside>