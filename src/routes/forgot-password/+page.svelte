<script lang="ts">
  import { PUBLIC_API_URL } from '$env/static/public';

  let email = "";
  let isLoading = false;
  let successMessage = "";
  let errorMessage = "";

  async function handleForgot() {
    if (!email) {
      errorMessage = "Harap masukkan email Anda.";
      return;
    }

    isLoading = true;
    errorMessage = "";
    successMessage = "";

    try {
      const res = await fetch(`${PUBLIC_API_URL}/forgot-password`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json'
        },
        body: JSON.stringify({ email })
      });

      const data = await res.json();

      if (res.ok && data.status) {
        successMessage = data.message || "Tautan reset password telah dikirim ke email Anda.";
        email = ""; // Clear input
      } else {
        errorMessage = data.message || "Terjadi kesalahan. Silakan coba lagi.";
      }
    } catch (e) {
      errorMessage = "Gagal terhubung ke server.";
    } finally {
      isLoading = false;
    }
  }
</script>

<svelte:head>
  <title>Lupa Password - SIAKAD</title>
</svelte:head>

<div class="min-h-screen flex items-center justify-center bg-gray-50 px-4">
  <div class="max-w-md w-full bg-white rounded-3xl shadow-[0_4px_20px_-10px_rgba(0,0,0,0.1)] p-8">
    <div class="text-center mb-8">
      <div class="w-16 h-16 bg-[#2da76b]/10 text-[#2da76b] rounded-2xl flex items-center justify-center mx-auto mb-4">
        <svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21.2 8.4c.5.38.8.97.8 1.6v10a2 2 0 0 1-2 2H4a2 2 0 0 1-2-2V10a2 2 0 0 1 .8-1.6l8-6a2 2 0 0 1 2.4 0l8 6Z"/><path d="m22 10-8.97 5.7a1.94 1.94 0 0 1-2.06 0L2 10"/></svg>
      </div>
      <h1 class="text-2xl font-bold text-gray-800">Lupa Password?</h1>
      <p class="text-gray-500 mt-2 text-sm leading-relaxed">Masukkan email yang terdaftar pada akun Anda. Kami akan mengirimkan tautan untuk mengatur ulang kata sandi.</p>
    </div>

    {#if successMessage}
      <div class="bg-green-50 border border-green-200 text-green-700 px-4 py-3 rounded-xl mb-6 flex items-start gap-3">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="shrink-0 mt-0.5"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg>
        <span class="text-sm font-medium">{successMessage}</span>
      </div>
    {/if}

    {#if errorMessage}
      <div class="bg-red-50 border border-red-200 text-red-600 px-4 py-3 rounded-xl mb-6 flex items-start gap-3">
        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="shrink-0 mt-0.5"><circle cx="12" cy="12" r="10"/><line x1="12" x2="12" y1="8" y2="12"/><line x1="12" x2="12.01" y1="16" y2="16"/></svg>
        <span class="text-sm font-medium">{errorMessage}</span>
      </div>
    {/if}

    <form on:submit|preventDefault={handleForgot} class="flex flex-col gap-5">
      <div>
        <label for="email" class="text-sm font-bold text-gray-600 mb-1.5 block ml-1">Alamat Email</label>
        <input 
          id="email"
          type="email" 
          bind:value={email} 
          placeholder="contoh@email.com" 
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
          Mengirim...
        {:else}
          Kirim Tautan Reset
        {/if}
      </button>
    </form>

    <div class="mt-8 text-center">
      <a href="/" class="text-sm font-bold text-gray-500 hover:text-[#2da76b] transition-colors flex items-center justify-center gap-2">
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="m15 18-6-6 6-6"/></svg>
        Kembali ke Login
      </a>
    </div>
  </div>
</div>
