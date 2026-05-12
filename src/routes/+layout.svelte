<script lang="ts">
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { page } from '$app/stores';
  
  // Jika kamu menggunakan file CSS global untuk Tailwind, pastikan di-import di sini
  import './layout.css'; 

  onMount(() => {
    // SvelteKit store 'page.subscribe' akan selalu berjalan setiap kali URL berubah
    const unsubscribe = page.subscribe(($page) => {
      const token = localStorage.getItem('auth_token');
      const currentPath = $page.url.pathname;

      // 1. Jika TIDAK ADA token dan mencoba masuk ke halaman selain '/login'
      if (!token && currentPath !== '/login') {
        goto('/login');
      }

      // 2. Jika SUDAH ADA token tapi mencoba iseng buka halaman '/login' lagi
      if (token && currentPath === '/login') {
        goto('/dashboard');
      }
    });

    // Bersihkan langganan (subscription) saat komponen dihancurkan
    return unsubscribe;
  });
</script>

<slot />