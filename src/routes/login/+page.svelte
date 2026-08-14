<script lang="ts">
  import { goto } from '$app/navigation'; 
  import {PUBLIC_API_URL} from '$env/static/public';

  let username = '';
  let password = '';
  let showPassword = false;
  
  // State baru untuk UI
  let errorMessage = ''; 
  let isLoading = false;

  async function handleLogin() {
    errorMessage = ''; // Reset pesan error
    isLoading = true;  // Ubah status tombol menjadi loading

    try {
      // Menembak API Laravel (Pastikan URL sesuai dengan port Laravel-mu)
      const response = await fetch(`${PUBLIC_API_URL}/login`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json' // Beri tahu Laravel kita mau response JSON
        },
        body: JSON.stringify({ 
          username: username, 
          password: password 
        })
      });

      const result = await response.json();

      // Jika response dari Laravel sukses (status 200 OK)
      if (response.ok && result.access_token) {
        
        // 1. Simpan Token ke Local Storage agar sesi login tersimpan
        localStorage.setItem('auth_token', result.access_token);
        
        // 2. Simpan Data User (opsional, untuk nama di Sidebar nanti)
        localStorage.setItem('user_data', JSON.stringify(result.user));

        // 3. Arahkan ke halaman Dashboard
        goto('/dashboard');
        
      } else {
        // Jika gagal (misal: password salah dari backend)
        errorMessage = result.message || 'Username atau password salah.';
      }
    } catch (error) {
      // Jika server Laravel mati / masalah koneksi internet
      errorMessage = 'Tidak dapat terhubung ke server backend.';
    } finally {
      isLoading = false; // Matikan status loading
    }
  }
</script>

<svelte:head>
  <title>Login - SIAKAD Al Hijrah</title>
</svelte:head>

<main class="min-h-screen bg-gray-100 flex flex-col items-center">
  <div class="m-auto max-w-6xl w-full flex rounded-[30px] overflow-hidden shadow-2xl bg-white mx-4">
    
    <div class="hidden md:flex w-1/2 bg-gradient-to-br from-[#2da76b] to-[#1e6e46] p-12 items-center gap-6">
      <div class="w-20 h-20 rounded-full bg-yellow-400 overflow-hidden shadow-lg shrink-0">
        <img src="/logo-alhijrah.png" alt="Logo Al-Hijrah" class="w-full h-full object-cover" />
      </div>
      <div class="text-white">
        <h2 class="text-2xl font-bold leading-tight">Sistem Informasi Akademik</h2>
        <h3 class="text-lg font-medium opacity-90">Yayasan Al Hijrah</h3>
      </div>
    </div>

    <div class="w-full md:w-1/2 p-8 md:p-16 flex flex-col justify-center gap-8 relative">
      <h1 class="text-4xl font-bold text-gray-800 text-center mb-2">Login</h1>

      {#if errorMessage}
        <div class="bg-red-50 text-red-500 border border-red-200 p-4 rounded-2xl text-sm font-semibold flex items-center gap-3 animate-in fade-in slide-in-from-top-2">
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" class="shrink-0"><circle cx="12" cy="12" r="10"/><line x1="12" x2="12" y1="8" y2="12"/><line x1="12" x2="12.01" y1="16" y2="16"/></svg>
          {errorMessage}
        </div>
      {/if}

      <form on:submit|preventDefault={handleLogin} class="flex flex-col gap-6">
        
        <div class="flex flex-col gap-2">
          <label for="username" class="text-gray-600 font-semibold ml-1">Username</label>
          <input 
            type="text" 
            id="username"
            bind:value={username}
            required
            placeholder="Masukkan Username"
            class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-400"
          />
        </div>

        <div class="flex flex-col gap-2 relative">
          <label for="password" class="text-gray-600 font-semibold ml-1">Password</label>
          <input 
            type={showPassword ? 'text' : 'password'} 
            id="password"
            bind:value={password}
            required
            placeholder="Masukkan Password"
            class="w-full px-5 py-4 rounded-2xl bg-gray-100 border-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 placeholder:text-gray-400"
          />
          <button 
            type="button" 
            on:click={() => showPassword = !showPassword}
            class="absolute right-5 top-[52px] text-gray-400 hover:text-[#2da76b] transition"
          >
            {#if showPassword}
              <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M9.88 9.88a3 3 0 1 0 4.24 4.24"/><path d="M10.73 5.08A10.43 10.43 0 0 1 12 5c7 0 10 7 10 7a13.16 13.16 0 0 1-1.67 2.68"/><path d="M6.61 6.61A13.526 13.526 0 0 0 2 12s3 7 10 7a9.74 9.74 0 0 0 5.39-1.61"/><line x1="2" x2="22" y1="2" y2="22"/></svg>
            {:else}
              <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/><circle cx="12" cy="12" r="3"/></svg>
            {/if}
          </button>
        </div>

        <div class="flex justify-end mt-[-10px] mb-2">
          <a href="/forgot-password" class="text-sm font-semibold text-gray-500 hover:text-[#2da76b] transition-colors">
            Lupa Password?
          </a>
        </div>

        <button 
          type="submit"
          disabled={isLoading}
          class="w-full py-4 mt-2 rounded-2xl bg-gradient-to-r from-[#2da76b] to-[#1e6e46] text-white font-bold text-lg hover:shadow-lg hover:scale-[1.02] active:scale-95 transition-all flex justify-center items-center gap-2 disabled:opacity-70 disabled:pointer-events-none"
        >
          {#if isLoading}
            <svg class="animate-spin h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
            Memproses...
          {:else}
            Login
          {/if}
        </button>
      </form>
    </div>
  </div>

  <footer class="w-full mt-auto bg-gradient-to-r from-[#2da76b] to-[#1e6e46] p-5 text-center text-white text-xs font-medium">
    © 2026 Sistem Informasi Akademik Al Hijrah. All Right Reserved | Developed by prakarsakreatif.id
  </footer>
</main>