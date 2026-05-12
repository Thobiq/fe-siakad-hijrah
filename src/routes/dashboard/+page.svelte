<script lang="ts">
    import Sidebar from "$lib/components/Sidebar.svelte";
    import { onMount } from 'svelte';
    import { PUBLIC_API_URL } from '$env/static/public';

    let isSidebarOpen = false;
    let userName = "Memuat...";
    
    // Set nilai awal (default) menjadi 0
    let stats = [
        { title: "Siswa KB Ternilai", count: 0 },
        { title: "Siswa TK A Ternilai", count: 0 },
        { title: "Siswa TK B Ternilai", count: 0 }
    ];

    onMount(async () => {
        // 1. Ambil nama user dari Local Storage
        const userData = JSON.parse(localStorage.getItem('user_data') || '{}');
        if (userData.name) userName = userData.name;

        // 2. Ambil data statistik dari Backend Laravel
        const token = localStorage.getItem('auth_token');
        try {
            const response = await fetch(`${PUBLIC_API_URL}/dashboard/stats`, {
                headers: {
                    'Authorization': `Bearer ${token}`,
                    'Accept': 'application/json'
                }
            });
            
            const result = await response.json();
            
            if (response.ok && result.status) {
                // Update array stats agar Svelte otomatis merender ulang angkanya di layar
                stats = [
                    { title: "Siswa KB Ternilai", count: result.data.kb },
                    { title: "Siswa TK A Ternilai", count: result.data.tka },
                    { title: "Siswa TK B Ternilai", count: result.data.tkb }
                ];
            }
        } catch (error) {
            console.error("Gagal memuat statistik dashboard:", error);
        }
    });
</script>

<svelte:head>
  <title>Dashboard - SIAKAD Al Hijrah</title>
</svelte:head>

<div class="flex h-screen bg-gray-50 overflow-hidden">
  

  <Sidebar bind:isOpen={isSidebarOpen} />

  <div class="flex-1 flex flex-col overflow-hidden">
    
    <header class="h-20 bg-white flex items-center justify-between px-6 md:px-10 shrink-0 border-b border-gray-100">
      
        <div class="flex items-center gap-4">
            <button 
            class="md:hidden p-2 -ml-2 text-gray-500 hover:bg-gray-100 rounded-lg transition"
            on:click={() => isSidebarOpen = true}>
                <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="4" x2="20" y1="12" y2="12"/><line x1="4" x2="20" y1="6" y2="6"/><line x1="4" x2="20" y1="18" y2="18"/></svg>
            </button>

            <h1 class="text-xl font-bold text-gray-600">Dashboard</h1>
        </div>
        
        <div class="flex items-center gap-3">
            <span class="font-bold text-gray-600 hidden sm:block">{userName}</span>
            <div class="w-10 h-10 rounded-full bg-[#2da76b] text-white flex items-center justify-center shrink-0">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="8" r="5"/><path d="M20 21a8 8 0 0 0-16 0"/></svg>
            </div>
        </div>
    </header>

    <main class="flex-1 overflow-y-auto p-10 flex flex-col gap-8">
      
      <div class="bg-white rounded-3xl p-10 flex items-center justify-between border border-gray-100 shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)]">
        <div>
          <h2 class="text-4xl font-bold text-gray-500 mb-2">Hallo, <span class="text-[#2da76b]">{userName}</span></h2>
          <p class="text-gray-500 font-medium">Selamat Datang di Sistem Informasi Akademik Al-hijrah</p>
        </div>
        
        <div class="hidden md:block w-48 h-32 bg-white  flex items-center justify-center text-sm">
          <img src="teacher-icon.png" alt="">
        </div>
      </div>

      <div class="grid grid-cols-1 md:grid-cols-3 gap-8">
        {#each stats as stat}
          <div class="bg-white rounded-3xl overflow-hidden shadow-[0_4px_20px_-10px_rgba(0,0,0,0.05)] border border-gray-100">
            <div class="bg-[#2da76b] text-white text-center py-4">
              <h3 class="font-bold text-lg">{stat.title}</h3>
            </div>
            <div class="py-12 text-center">
              <span class="text-[80px] font-bold text-[#2da76b] leading-none">{stat.count}</span>
            </div>
          </div>
        {/each}
      </div>

    </main>

    <footer class="bg-white py-5 text-center text-sm font-medium text-gray-500 shrink-0 border-t border-gray-100">
      © 2026 Sistem Informasi Akademik Al Hijrah. All Right Reserved | Developed by prakarsakreatif.id
    </footer>

  </div>
</div>