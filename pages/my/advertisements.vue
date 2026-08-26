<script setup lang="ts">
definePageMeta({
    layout: "panel",
    middleware: "auth",
});

const conf = useRuntimeConfig();
const token = useToken();
const ads = ref([]);
const success = ref(null);
const errors = ref(null);
const isModalOpen = ref(false);

// Helper to clear messages after a few seconds
function clearMessages() {
    setTimeout(() => {
        success.value = null;
        errors.value = null;
    }, 4000);
}

// Pagination State (matching your pattern)
const paginationIndex = ref(0);
const paginationMaxSize = ref(10);

// Global Settings State
const globalSettings = ref({
    AdsVisibility: false,
    AdsSkipSeconds: 10
});

// Form State
const form = ref({
    ID: 0, 
    AdsName: '', 
    AdsType: 'video', 
    AdsLink: '', 
    AdsDisplay: 'before_start', 
    AdsDevice: 'all', 
    AdsPriority: 0, 
    IsActive: true
});

// Open modal in "Create" mode
function openCreate() {
    resetForm();
    isModalOpen.value = true;
}

// Open modal in "Edit" mode with existing data
function openEdit(ad: any) {
    form.value = { ...ad }; // Clone the object so changes don't affect table until saved
    isModalOpen.value = true;
}

// Reset form to defaults
function resetForm() {
    form.value = {
        ID: 0, AdsName: '', AdsType: 'video', AdsLink: '', 
        AdsDisplay: 'before_start', AdsDevice: 'all', AdsPriority: 0, IsActive: true
    };
}

const listPaginationItems = computed(() => {
    const start = paginationIndex.value * paginationMaxSize.value;
    return ads.value.slice(start, start + paginationMaxSize.value);
});

const paginationMenusAmount = computed(() => Math.ceil(ads.value.length / paginationMaxSize.value));

// We store the full settings object here
const fullSettings = ref(null);

async function load() {
    try {
        const [adsData, configData] = await Promise.all([
            $fetch(`${conf.public.apiUrl}/advertisements`, { headers: { Authorization: `Bearer ${token.value}` } }),
            $fetch(`${conf.public.apiUrl}/settings`, { headers: { Authorization: `Bearer ${token.value}` } }) // Get EVERYTHING
        ]);
        ads.value = adsData;
        fullSettings.value = configData; // Store the full object (ID, AppName, etc.)
    } catch (e: any) {
        errors.value = "Failed to load data.";
    }
}

async function saveGlobalSettings() {
    if (!fullSettings.value) return;

    try {
        // Create a final payload to ensure string types for Go
        const payload = {
            ...fullSettings.value,
            AdsVisibility: String(fullSettings.value.AdsVisibility), // CRITICAL: Convert boolean to string to comply with Setting database schema
            AdsSkipSeconds: String(fullSettings.value.AdsSkipSeconds) // CRITICAL: Convert integer to string to comply with Setting database schema
        };

        await $fetch(`${conf.public.apiUrl}/settings`, {
            method: 'PUT',
            headers: { Authorization: `Bearer ${token.value}` },
            body: payload 
        });
        
        success.value = "Settings updated successfully!"; 
        load(); 
        clearMessages();
    } catch (e: any) { 
        errors.value = "Save failed"; 
    }
}

// Single function to handle both Create (POST) and Update (PUT)
async function saveAd() {
    const isUpdate = form.value.ID > 0;
    const url = isUpdate 
        ? `${conf.public.apiUrl}/advertisements/${form.value.ID}` 
        : `${conf.public.apiUrl}/advertisements`;

    const payload = {
        ads_name:     form.value.AdsName,
        ads_type:     form.value.AdsType,
        ads_link:     form.value.AdsLink,
        ads_display:  form.value.AdsDisplay,
        ads_device:   form.value.AdsDevice,
        ads_priority: parseInt(form.value.AdsPriority) || 0, // CRITICAL: Ensure it's a number
        is_active:    Boolean(form.value.IsActive)
    };

    try {
        await $fetch(url, {
            method: isUpdate ? 'PUT' : 'POST',
            headers: { Authorization: `Bearer ${token.value}` },
            body: payload
        });
        isModalOpen.value = false;
        success.value = "Advertisement saved!"; 
        load();
        clearMessages();
    } catch (e: any) {
        errors.value = e.data?.error || "Request failed";
    }
}

// Toggle Visibility (Active/Inactive) directly from the list
async function toggleAdStatus(ad: any) {
    try {
        await $fetch(`${conf.public.apiUrl}/advertisements/${ad.ID}`, {
            method: 'PUT',
            headers: { Authorization: `Bearer ${token.value}` },
            body: { is_active: !ad.IsActive }
        });
        load();
    } catch (e: any) {
        errors.value = "Failed to toggle advertisement status.";
    }
}

async function deleteAd(id: number) {
    if (!confirm('Are you sure want to delete?')) return;
    await $fetch(`${conf.public.apiUrl}/advertisements/${id}`, {
        method: 'DELETE',
        headers: { Authorization: `Bearer ${token.value}` },
    });
    load();
}

// Filter options based on selected Type
const zoneOptions = computed(() => {
    return [
        { id: 'before_start', label: 'Before Start' },
        { id: 'mid_roll', label: 'Mid-roll' }
    ];
});

// Toggle multi-select values in a comma-separated string
function toggleZone(zoneId) {
    let zones = form.value.AdsDisplay ? form.value.AdsDisplay.split(',') : [];
    const index = zones.indexOf(zoneId);
    
    if (index > -1) {
        zones.splice(index, 1);
    } else {
        // If switching from banner to video, clear 'center'
        if (form.value.AdsType === 'video' && zones.includes('center')) zones = [];
        zones.push(zoneId);
    }
    form.value.AdsDisplay = zones.join(',');
}

// Watch for Type changes to reset invalid zones
// watch(() => form.value.AdsType, (newType) => {
//     form.value.AdsDisplay = newType === 'video' ? 'before_start' : 'center';
// });

onMounted(load);
</script>

<template>
    <div class="flex flex-col grow gap-6">
        <!-- Page Header -->
        <div class="flex justify-between items-start">
            <div class="flex flex-col gap-1">
                <h1 class="text-2xl font-bold">Advertisement</h1>
                <p class="text-sm opacity-70">Manage banner or video ads for player.</p>
            </div>            
        </div>

        <!-- Success Alert -->
        <div v-if="success" class="alert alert-success shadow-sm border-none bg-emerald-500 text-white">
            <Icon name="lucide:check-circle" class="stroke-current shrink-0 h-6 w-6" />
            <div class="font-bold">{{ success }}</div>
            <button @click="success = null" class="btn btn-sm btn-circle btn-ghost ml-auto">✕</button>
        </div>

        <!-- Error Alert -->
        <div v-if="errors" class="alert alert-error shadow-sm">
            <Icon name="lucide:alert-circle" class="stroke-current shrink-0 h-6 w-6" />
            <div>{{ errors }}</div>
            <button @click="errors = null" class="btn btn-sm btn-circle btn-ghost ml-auto">✕</button>
        </div>

        <!-- Global Ads Control Bar. Use v-if so it only renders once the data is read -->
        <div v-if="fullSettings" class="card bg-base-100 shadow-sm border border-base-200 mb-2">
            <div class="p-3 flex items-center justify-between">
                <div class="flex items-center gap-8">
                    <div class="flex items-center gap-3">
                        <span class="text-[10px] font-black uppercase opacity-60">Global Ads</span>
                        <input 
                            type="checkbox" 
                            v-model="fullSettings.AdsVisibility" 
                            true-value="true" 
                            false-value="false" 
                            class="toggle toggle-primary toggle-xs" 
                        />
                    </div>
                    <div class="flex items-center gap-3">
                        <span class="text-[10px] font-black uppercase opacity-60">Skip Delay</span>
                        <!-- Use type="text" to keep it as a string for Go -->
                        <input 
                            type="text" 
                            v-model="fullSettings.AdsSkipSeconds" 
                            class="input input-bordered input-xs w-16 text-center" 
                        />
                        <span class="text-[10px] opacity-40">sec</span>
                    </div>
                </div>
                <button @click="saveGlobalSettings" class="btn btn-success btn-xs px-4 text-white font-bold">SAVE</button>
            </div>
        </div>

        <!-- Optional: Show a placeholder while loading -->
        <div v-else class="h-16 flex items-center justify-center opacity-30">
            <span class="loading loading-dots loading-sm"></span>
        </div>

        <!-- Quick Stats -->
        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div class="stats shadow bg-base-100 border border-base-200">
                <div class="stat">
                    <div class="stat-figure text-primary">
                        <Icon name="lucide:megaphone" class="w-8 h-8" />
                    </div>
                    <div class="stat-title">Active Ads</div>
                    <div class="stat-value text-primary">{{ ads.filter(a => a.IsActive).length }}</div>
                    <div class="stat-desc">Currently showing to guests</div>
                </div>
            </div>
            <div class="stats shadow bg-base-100 border border-base-200">
                <div class="stat">
                    <div class="stat-figure text-secondary">
                        <Icon name="lucide:layers" class="w-8 h-8" />
                    </div>
                    <div class="stat-title">Total Inventory</div>
                    <div class="stat-value text-secondary">{{ ads.length }}</div>
                    <div class="stat-desc">Combined video and banners</div>
                </div>
            </div>
        </div>

        <div class="flex justify-between items-start">
            <div class="flex flex-col gap-1">
            </div>     
             <button @click="openCreate()" class="btn btn-primary btn-sm px-6 rounded-lg">
                <Icon name="lucide:plus" class="w-4 h-4" />
                Add
            </button>       
        </div>

        <!-- Ads Table Card -->
        <div class="card bg-base-100 shadow-xl border border-base-200">
            <div class="card-body p-0">
                <div class="overflow-x-auto">
                    <table class="table table-zebra w-full">
                        <thead class="bg-base-200/50">
                            <tr>
                                <th>Name</th>
                                <th>Type</th>
                                <th>Device</th>
                                <th>Status</th>
                                <th class="text-right">Actions</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-if="ads.length === 0">
                                <td colspan="5">
                                    <div class="flex flex-col items-center justify-center py-12 opacity-50">
                                        <Icon name="lucide:megaphone-off" class="w-12 h-12 mb-2" />
                                        <p>No advertisements found.</p>
                                    </div>
                                </td>
                            </tr>
                            <tr v-for="ad in listPaginationItems" :key="ad.ID">
                                <td class="font-bold">{{ ad.AdsName }}</td>
                                <td><span class="badge badge-ghost font-mono uppercase text-[10px]">{{ ad.AdsType }}</span></td>
                                <td><span class="text-xs opacity-70">{{ ad.AdsDevice }}</span></td>
                                <td>
                                    <!-- Visibility Status Indicator -->
                                    <div :class="ad.IsActive ? 'text-primary' : 'text-error'" class="flex items-center gap-2 text-xs font-bold">
                                        <div class="w-2 h-2 rounded-full bg-current"></div>
                                        {{ ad.IsActive ? 'Visible' : 'Hidden' }}
                                    </div>
                                </td>
                                <td class="flex justify-end gap-1">
                                    <!-- Edit Button -->
                                    <button @click="openEdit(ad)" class="btn btn-square btn-ghost btn-xs text-info" title="Edit Ad">
                                        <Icon name="lucide:edit-3" class="w-4 h-4" />
                                    </button>
                                    
                                    <!-- Visibility Toggle Button -->
                                    <button @click="toggleAdStatus(ad)" class="btn btn-square btn-ghost btn-xs" :class="ad.IsActive ? 'text-primary' : 'text-slate-400'" title="Toggle Visibility">
                                        <Icon :name="ad.IsActive ? 'lucide:eye' : 'lucide:eye-off'" class="w-4 h-4" />
                                    </button>

                                    <!-- Delete Button -->
                                    <button @click="deleteAd(ad.ID)" class="btn btn-square btn-ghost btn-xs text-error" title="Delete Ad">
                                        <Icon name="lucide:trash-2" class="w-4 h-4" />
                                    </button>
                                </td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
        </div>

        <!-- Pagination (matching your pattern) -->
        <div class="flex justify-between items-center px-2">
            <div class="dropdown dropdown-top">
                <label tabindex="0" class="btn btn-ghost btn-sm text-xs font-normal">
                    Show {{ paginationMaxSize }}
                    <Icon name="lucide:chevron-up" class="w-3 h-3 ml-1" />
                </label>
                <ul tabindex="0" class="dropdown-content z-[1] menu p-2 shadow bg-base-100 rounded-box w-32 border border-base-200">
                    <li v-for="max in [10, 25, 50, 100]" :key="max">
                        <button @click="paginationMaxSize = max" :class="{ 'active': paginationMaxSize === max }">
                            {{ max }} rows
                        </button>
                    </li>
                </ul>
            </div>
            <div class="join">
                <button 
                    v-for="index in paginationMenusAmount" :key="index"
                    @click="paginationIndex = index - 1" 
                    class="join-item btn btn-sm"
                    :class="paginationIndex === index - 1 ? 'btn-active' : ''"
                >
                    {{ index }}
                </button>
            </div>
        </div>
    </div>
    <dialog :open="isModalOpen" class="modal modal-bottom sm:modal-middle">
        <div class="modal-box bg-[#1a2234] border border-slate-700 max-w-lg p-8">
            <h3 class="font-bold text-lg text-white mb-8">Add Advertisement</h3>
            
            <!-- Form Container: Vertical Stack with Fixed-Width Labels -->
            <div class="flex flex-col gap-6">
                
                <!-- Row 1: Ad Name -->
                <div class="grid grid-cols-[100px_1fr] items-center gap-4">
                    <label class="text-slate-400 text-sm">Ad Name</label>
                    <input v-model="form.AdsName" type="text" class="input input-bordered bg-[#0f172a] border-slate-600 text-sm h-10" placeholder="e.g. New Member Signup Promo" />
                </div>

                <!-- Row 2: Type -->
                <div class="grid grid-cols-[100px_1fr] items-center gap-4">
                    <label class="text-slate-400 text-sm">Type</label>
                    <select v-model="form.AdsType" class="select select-bordered bg-[#0f172a] border-slate-600 text-sm h-10 min-h-0">
                        <option value="video">Video</option>
                        <option value="banner">Banner</option>
                    </select>
                </div>

                <!-- Row 3: Device -->
                <div class="grid grid-cols-[100px_1fr] items-center gap-4">
                    <label class="text-slate-400 text-sm">Device</label>
                    <select v-model="form.AdsDevice" class="select select-bordered bg-[#0f172a] border-slate-600 text-sm h-10 min-h-0">
                        <option value="all">All Devices</option>
                        <option value="desktop">Desktop</option>
                        <option value="mobile">Mobile</option>
                        <option value="tablet">Tablet</option>
                    </select>
                </div>

                <!-- Row 4: Link / Slug -->
                <div class="grid grid-cols-[100px_1fr] items-center gap-4">
                    <label class="text-slate-400 text-sm">Banner / Video Link</label>
                    <input v-model="form.AdsLink" type="text" class="input input-bordered bg-[#0f172a] border-slate-600 text-sm h-10" placeholder="Enter banner/video Ad link" />
                </div>

                <!-- Row 5: Display Zone (Multi-select) -->
                <div class="grid grid-cols-[100px_1fr] items-start gap-4">
                    <label class="text-slate-400 text-sm mt-2">Display Zone</label>
                    <div class="flex flex-wrap gap-2 p-2 rounded-lg bg-[#0f172a] border border-slate-600 min-h-[40px]">
                        <button 
                            v-for="opt in zoneOptions" 
                            :key="opt.id"
                            @click="toggleZone(opt.id)"
                            type="button"
                            class="btn btn-xs rounded-md border-none lowercase"
                            :class="form.AdsDisplay.includes(opt.id) ? 'btn-primary' : 'bg-slate-700 text-slate-400'"
                        >
                            {{ opt.label }}
                        </button>
                    </div>
                </div>

                <!-- Row 6: Priority -->
                <div class="grid grid-cols-[100px_1fr] items-center gap-4">
                    <label class="text-slate-400 text-sm">Priority</label>
                    <div class="flex flex-col gap-1">
                        <input 
                            v-model.number="form.AdsPriority" 
                            type="number" 
                            class="input input-bordered bg-[#0f172a] border-slate-600 text-sm h-10 w-full" 
                            placeholder="0"
                        />
                        <span class="text-[10px] text-slate-500 font-medium italic ml-1">
                            * Higher number means higher priority
                        </span>
                    </div>
                </div>
            </div>

            <!-- Footer Buttons -->
            <div class="modal-action flex justify-end gap-3 mt-10">
                <button @click="isModalOpen = false" class="btn btn-ghost text-white capitalize font-normal">Cancel</button>
                <button @click="saveAd" class="btn btn-primary px-8 capitalize font-bold rounded-md">Save Changes</button>
            </div>
        </div>
    </dialog>
</template>