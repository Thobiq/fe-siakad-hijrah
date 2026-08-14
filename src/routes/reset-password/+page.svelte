<script lang="ts">
  import { onMount } from 'svelte';
  import { page } from '$app/stores';
  import { PUBLIC_API_URL } from '$env/static/public';

  let email = "";
  let token = "";
  let password = "";
  let passwordConfirm = "";
  let isLoading = false;
  let successMessage = "";
  let errorMessage = "";
  let isTokenValid = false;

  onMount(() => {
    email = $page.url.searchParams.get('email') || "";
    token = $page.url.searchParams.get('token') || "";

    if (!email || !token) {
      errorMessage = "Tautan reset password tidak valid atau tidak lengkap.";
    } else {
      isTokenValid = true;
    }
  });

  async function handleReset() {
    if (!password || !passwordConfirm) {
      errorMessage = "Semua field harus diisi.";
      return;
    }

    if (password !== passwordConfirm) {
      errorMessage = "Konfirmasi kata sandi tidak cocok.";
      return;
    }

    if (password.length < 6) {
      errorMessage = "Kata sandi minimal 6 karakter.";
      return;
    }

    isLoading = true;
    errorMessage = "";
    successMessage = "";

    try {
      const res = await fetch(`${PUBLIC_API_URL}/reset-password`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json'
        },
        body: JSON.stringify({ email, token, password })
      });

      const data = await res.json();

      if (res.ok && data.status) {
        successMessage = data.message || "Password berhasil direset. Silakan login.";
        isTokenValid = false; // Sembunyikan form agar tidak bisa di-submit lagi
      } else {
        errorMessage = data.message || "Terjadi kesalahan atau token sudah hangus.";
      }
    } catch (e) {
      errorMessage = "Gagal terhubung ke server.";
    } finally {
      isLoading = false;
    }
  }
</script>

<svelte:head>
  <title>Reset Password - SIAKAD</title>
</svelte:head>

<div class="min-h-screen flex items-center justify-center bg-gray-50 px-4">
  <div class="max-w-md w-full bg-white rounded-3xl shadow-[0_4px_20px_-10px_rgba(0,0,0,0.1)] p-8">
    <div class="text-center mb-8">
      <div class="w-16 h-16 bg-[#2da76b]/10 text-[#2da76b] rounded-2xl flex items-center justify-center mx-auto mb-4">
        <svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect width="18" height="11" x="3" y="11" rx="2" ry="2"/><path d="M7 11V7a5 5 0 0 1 10 0v4"/></svg>
      </div>
      <h1 class="text-2xl font-bold text-gray-800">Atur Ulang Password</h1>
      {#if isTokenValid}
        <p class="text-gray-500 mt-2 text-sm leading-relaxed">Buat kata sandi baru untuk akun Anda ({email}).</p>
      {/if}
    </div>

    {#if successMessage}
      <div class="bg-green-50 border border-green-200 text-green-700 px-4 py-3 rounded-xl mb-6 flex flex-col gap-2">
        <div class="flex items-start gap-3">
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="shrink-0 mt-0.5"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg>
          <span class="text-sm font-medium">{successMessage}</span>
        </div>
        <a href="/" class="mt-2 text-center w-full bg-green-600 hover:bg-green-700 text-white font-bold py-3 rounded-xl transition-colors text-sm">
          Pergi ke Halaman Login
        </a>
      </div>
    {/if}

    {#if errorMessage}
      <div class="bg-red-50 border border-red-200 text-red-600 px-4 py-3 rounded-xl mb-6 flex items-start gap-3">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="shrink-0 mt-0.5"><circle cx="12" cy="12" r="10"/><line x1="12" x2="12" y1="8" y2="12"/><line x1="12" x2="12.01" y1="16" y2="16"/></svg>
        <span class="text-sm font-medium">{errorMessage}</span>
      </div>
    {/if}

    {#if isTokenValid}
      <form on:submit|preventDefault={handleReset} class="flex flex-col gap-5">
        <div>
          <label for="password" class="text-sm font-bold text-gray-600 mb-1.5 block ml-1">Password Baru</label>
          <input 
            id="password"
            type="password" 
            bind:value={password} 
            placeholder="Minimal 6 karakter" 
            required
            class="w-full px-5 py-3.5 rounded-xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 font-medium transition-all"
          />
        </div>

        <div>
          <label for="passwordConfirm" class="text-sm font-bold text-gray-600 mb-1.5 block ml-1">Konfirmasi Password</label>
          <input 
            id="passwordConfirm"
            type="password" 
            bind:value={passwordConfirm} 
            placeholder="Ulangi password baru" 
            required
            class="w-full px-5 py-3.5 rounded-xl bg-gray-100 border-none outline-none focus:ring-2 focus:ring-[#2da76b] text-gray-800 font-medium transition-all"
          />
        </div>

        <button 
          type="submit" 
          disabled={isLoading}
          class="w-full py-4 mt-2 bg-[#2da76b] hover:bg-[#289562] disabled:opacity-70 text-white rounded-xl font-bold text-lg transition-colors shadow-lg shadow-[#2da76b]/20 flex items-center justify-center gap-2"
        >
          {#if isLoading}
            <svg class="animate-spin h-5 w-5" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
            Menyimpan...
          {:else}
            Simpan Password Baru
          {/if}
        </button>
      </form>
    {/if}
  </div>
</div>
