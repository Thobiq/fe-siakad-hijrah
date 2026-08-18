<script lang="ts">
    import { onMount } from 'svelte';
    import { PUBLIC_API_URL } from '$env/static/public';
    import Sidebar from '$lib/components/Sidebar.svelte';

    let isSidebarOpen = false;
    let userName = "Admin";


    interface ProfilSekolah {
        id: number;
        nama_sekolah: string;
        alamat: string;
        nama_kepala_sekolah: string;
        logo_path: string | null;
    }

    let profil: ProfilSekolah = {
        id: 0,
        nama_sekolah: '',
        alamat: '',
        nama_kepala_sekolah: '',
        logo_path: null
    };

    let isLoading = true;
    let isSaving = false;
    let message = '';
    let isError = false;

    let selectedFile: File | null = null;
    let imagePreview: string | null = null;

    onMount(async () => {
        const userData = JSON.parse(localStorage.getItem('user_data') || '{}');
        if (userData.name) userName = userData.name;
        await loadProfil();
    });

    async function loadProfil() {
        try {
            isLoading = true;
            const token = localStorage.getItem('auth_token');
            const response = await fetch(`${PUBLIC_API_URL}/profil-sekolah`, {
                headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' }
            });
            const res = await response.json();
            
            if (response.ok) {
                profil = res;
                if (profil.logo_path) {
                    const baseUrl = PUBLIC_API_URL.replace('/api', '');
                    imagePreview = `${baseUrl}/` + profil.logo_path;
                }
            }
        } catch (e: any) {
            console.error('Gagal memuat profil', e);
        } finally {
            isLoading = false;
        }
    }

    function handleFileChange(event: Event) {
        const target = event.target as HTMLInputElement;
        if (target.files && target.files.length > 0) {
            selectedFile = target.files[0];
            imagePreview = URL.createObjectURL(selectedFile);
        }
    }

    async function saveProfil() {
        try {
            isSaving = true;
            message = '';
            
            const formData = new FormData();
            formData.append('nama_sekolah', profil.nama_sekolah);
            formData.append('alamat', profil.alamat);
            formData.append('nama_kepala_sekolah', profil.nama_kepala_sekolah);
            
            if (selectedFile) {
                formData.append('logo', selectedFile);
            }

            const token = localStorage.getItem('auth_token');
            const res = await fetch(`${PUBLIC_API_URL}/profil-sekolah`, {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${token}`,
                    'Accept': 'application/json'
                },
                body: formData
            });

            if (!res.ok) throw new Error('Gagal menyimpan profil');
            
            const result = await res.json();
            profil = result.data;
            if (profil.logo_path) {
                const baseUrl = PUBLIC_API_URL.replace('/api', '');
                imagePreview = `${baseUrl}/` + profil.logo_path;
            }
            
            // Beri tahu Sidebar bahwa profil telah diperbarui
            window.dispatchEvent(new CustomEvent('profilUpdated', { detail: profil }));
            
            message = 'Profil sekolah berhasil diperbarui!';
            isError = false;
            
            // clear success message after 3 seconds
            setTimeout(() => {
                message = '';
            }, 3000);
        } catch (e: any) {
            message = e.message || 'Terjadi kesalahan';
            isError = true;
        } finally {
            isSaving = false;
        }
    }
</script>

<svelte:head>
    <title>Profil Sekolah - SIAKAD Al-Hijrah</title>
</svelte:head>

<div class="flex h-screen bg-gray-50 overflow-hidden font-sans">
    <Sidebar bind:isOpen={isSidebarOpen} />

    <div class="flex-1 flex flex-col overflow-hidden w-full relative">
        <header class="h-20 bg-white flex items-center justify-between px-6 md:px-10 shrink-0 border-b border-gray-100">
            <div class="flex items-center gap-4">
                <button class="md:hidden p-2 -ml-2 text-gray-500 hover:bg-gray-100 rounded-lg transition" on:click={() => isSidebarOpen = true}>
                    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="4" x2="20" y1="12" y2="12"/><line x1="4" x2="20" y1="6" y2="6"/><line x1="4" x2="20" y1="18" y2="18"/></svg>
                </button>
                <h1 class="text-xl font-bold text-gray-600">Profil Sekolah</h1>
            </div>
            <div class="flex items-center gap-3">
                <span class="font-bold text-gray-600 hidden sm:block">{userName}</span>
                <div class="w-10 h-10 rounded-full bg-[#2da76b] text-white flex items-center justify-center shrink-0">
                    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="8" r="5"/><path d="M20 21a8 8 0 0 0-16 0"/></svg>
                </div>
            </div>
        </header>

        <main class="flex-1 overflow-y-auto p-6 md:p-10 flex flex-col">
            {#if isLoading}
                <div class="flex justify-center items-center py-12">
                    <div class="animate-spin rounded-full h-8 w-8 border-b-2 border-[#2da76b]"></div>
                </div>
            {:else}
                <div class="bg-white rounded-lg shadow-sm border border-gray-100 max-w-2xl">
                    <div class="p-6">
                {#if message}
                    <div class="mb-4 p-4 rounded-md {isError ? 'bg-red-50 text-red-700' : 'bg-green-50 text-green-700'}">
                        {message}
                    </div>
                {/if}

                <form on:submit|preventDefault={saveProfil} class="space-y-6">
                    <div>
                        <label class="block text-sm font-medium text-gray-700">Nama Sekolah</label>
                        <p class="text-xs text-gray-500 mb-1">Cukup isi nama yayasan/institusinya, (contoh: "Al-Hijrah"). Kata "KB" atau "TK" otomatis ditambahkan pada PDF.</p>
                        <input 
                            type="text" 
                            bind:value={profil.nama_sekolah}
                            class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500"
                            required
                        >
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700">Alamat Lengkap</label>
                        <textarea 
                            bind:value={profil.alamat}
                            rows="3"
                            class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500"
                            required
                        ></textarea>
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700">Nama Kepala Sekolah</label>
                        <input 
                            type="text" 
                            bind:value={profil.nama_kepala_sekolah}
                            class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500"
                            required
                        >
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700">Logo Sekolah</label>
                        <div class="mt-2 flex items-center space-x-6">
                            <div class="shrink-0 h-24 w-24 bg-gray-100 rounded-lg flex items-center justify-center overflow-hidden border border-gray-300">
                                {#if imagePreview}
                                    <img src={imagePreview} alt="Logo Preview" class="h-full w-full object-contain">
                                {:else}
                                    <span class="text-gray-400 text-sm">Tidak ada logo</span>
                                {/if}
                            </div>
                            <label class="block">
                                <span class="sr-only">Pilih Logo Baru</span>
                                <input type="file" 
                                    accept="image/*"
                                    on:change={handleFileChange}
                                    class="block w-full text-sm text-gray-500
                                    file:mr-4 file:py-2 file:px-4
                                    file:rounded-full file:border-0
                                    file:text-sm file:font-semibold
                                    file:bg-primary-50 file:text-primary-700
                                    hover:file:bg-primary-100
                                    cursor-pointer"
                                />
                            </label>
                        </div>
                    </div>

                    <div class="pt-4 border-t border-gray-200">
                            <button 
                                type="submit" 
                                disabled={isSaving}
                                class="inline-flex justify-center rounded-xl border border-transparent bg-[#2da76b] py-2.5 px-6 text-sm font-bold text-white shadow-sm hover:bg-[#258d59] focus:outline-none focus:ring-2 focus:ring-[#2da76b] focus:ring-offset-2 disabled:opacity-50 transition"
                            >
                                {isSaving ? 'Menyimpan...' : 'Simpan Profil'}
                            </button>
                        </div>
                    </form>
                </div>
            </div>
        {/if}
        </main>
    </div>
</div>
