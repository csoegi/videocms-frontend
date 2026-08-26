<script setup lang="ts">
const conf = useRuntimeConfig();
const token = useToken();
const { theme } = useTheme(); // Use your existing composable
const stats = ref<any>(null);
const loading = ref(true);
const activeTimeline = ref(30); // Default to 30 days
const refreshing = ref(false);

const load = async () => {
    try {
        const data = await $fetch(`${conf.public.apiUrl}/stats/analytics`, {
            headers: { Authorization: `Bearer ${token.value}` }
        });
        stats.value = data;
    } catch (e) {
        console.error(e);
    } finally {
        loading.value = false;
    }
};

const refreshReport = async () => {
    refreshing.value = true;
    try {
        await $fetch(`${conf.public.apiUrl}/stats/analytics/refresh`, {
            method: 'POST',
            headers: { Authorization: `Bearer ${token.value}` }
        });
        await load(); // Reload the data
    } finally {
        refreshing.value = false;
    }
};

const chartOptions = computed(() => ({
    chart: {
        type: 'area',
        toolbar: { show: false },
        fontFamily: 'inherit',
        background: 'transparent',
    },
    // Syncs ApexCharts internals with your useTheme state
    theme: { mode: theme.value }, 
    colors: ['#6366f1', '#38bdf8', '#4ade80'], // Matches your Indigo/Sky/Green variables
    stroke: { curve: 'smooth', width: 2 },
    xaxis: {
        categories: stats.value?.daily_stats.map(s => {
            const date = new Date(s.date);
            // Format based on timeline: "09 May" for days, "HH:mm" for 24h
            return date.toLocaleDateString('en-GB', { day: '2-digit', month: 'short' });
        }).reverse() || [],
        // LOGIC: 
        // 24h -> 1 (Show today only)
        // 7d  -> 7 (Show last 7 days)
        // 30d -> 30 (Show all 30 days)
        range: activeTimeline.value === 24 ? 1 : (activeTimeline.value === 7 ? 7 : 30),
        axisBorder: { show: false },
        labels: {
            style: { fontSize: '10px', colors: '#64748b' },
            rotate: -45,
            hideOverlappingLabels: false // Set to false to ensure labels show up when zoomed
        },
        markers: {
            size: activeTimeline.value === 24 ? 5 : 0, // Show dots only on the 24h view
            strokeColors: '#6366f1',
            strokeWidth: 2,
            hover: { size: 7 }
        }
    },
    yaxis: { labels: { style: { fontSize: '10px' } } },
    grid: { 
        // Uses a subtle transparency so it works on both base-200 themes
        borderColor: 'rgba(148, 163, 184, 0.1)', 
        strokeDashArray: 4 
    },
    dataLabels: { enabled: false },
    tooltip: { theme: theme.value }
}));

const series = computed(() => [
    { name: 'Views', data: stats.value?.daily_stats.map(s => s.views).reverse() || [] },
    { name: 'Streams', data: stats.value?.daily_stats.map(s => s.streams).reverse() || [] },
    { name: 'Downloads', data: stats.value?.daily_stats.map(s => s.downloads).reverse() || [] }
]);

// Helper to map device labels to Lucide icons
const getDeviceIcon = (device: string) => {
    const d = device.toLowerCase();
    if (d.includes('mobile')) return 'lucide:smartphone';
    if (d.includes('tablet')) return 'lucide:tablet';
    if (d.includes('desktop')) return 'lucide:monitor';
    return 'lucide:help-circle';
};

// Helper to convert ISO code to Flag Emoji
const getFlagEmoji = (countryCode: string) => {
    if (!countryCode || countryCode === 'Unknown') return '🏳️';
    const codePoints = countryCode
        .toUpperCase()
        .split('')
        .map(char => 127397 + char.charCodeAt(0));
    return String.fromCodePoint(...codePoints);
};

onMounted(load);
</script>

<template>
    <div v-if="stats" class="flex flex-col w-full gap-6 animate-in fade-in duration-500">
        <div v-if="loading" class="flex justify-center py-20">
            <span class="loading loading-spinner loading-lg text-primary"></span>
        </div>
        
        <div v-else-if="stats" class="flex flex-col gap-6 animate-in fade-in duration-500">
            <!-- Metric Cards -->
            <div class="grid grid-cols-1 md:grid-cols-3 gap-4 w-full">
                <div v-for="item in [
                        { label: 'Total Views', val: stats.total_views, color: 'text-primary' },
                        { label: 'Total Streams', val: stats.total_streams, color: 'text-secondary' },
                        { label: 'Total Downloads', val: stats.total_downloads, color: 'text-success' }
                    ]" :key="item.label" class="card bg-base-100 shadow-xl border border-base-200 h-full w-full">
                    <div class="card-body p-4">
                        <div class="card-title text-sm flex items-center gap-2">{{ item.label }}</div>
                        <div class="stat-value text-2xl" :class="item.color">{{ item.val }}</div>
                    </div>
                </div>
            </div>

            <!-- Chart Section -->
            <div class="card bg-base-100 shadow-xl border border-base-200 w-full">
                <div class="card-body p-6">
                    <div class="flex justify-between items-start mb-6">
                        <div class="flex items-center gap-3">
                            <div class="p-2 bg-primary/10 rounded-lg text-primary">
                                <Icon name="lucide:bar-chart-3" class="w-5 h-5" />
                            </div>
                            <div>
                                <h3 class="card-title text-sm flex items-center gap-2">Activity Overview</h3>
                                <p class="text-[10px] uppercase font-bold opacity-40">30-Day performance tracking</p>
                            </div>
                        </div>
                        <!-- Timeline & Refresh Controls -->
                        <div class="flex items-center gap-2 bg-base-200/50 p-1 rounded-lg border border-base-300">
                            <button v-for="t in [24, 7, 30]" :key="t" 
                                @click="activeTimeline = t"
                                class="btn btn-xs font-bold uppercase text-[10px] border-none"
                                :class="activeTimeline === t ? 'btn-primary' : 'btn-ghost opacity-50'">
                                {{ t === 24 ? '24h' : t + 'd' }}
                            </button>
                            <div class="divider divider-horizontal mx-0 w-px h-4 self-center bg-base-300"></div>
                            <button @click="refreshReport" :disabled="refreshing" class="btn btn-xs btn-ghost btn-circle">
                                <Icon name="lucide:rotate-cw" :class="{ 'animate-spin': refreshing }" class="w-3.5 h-3.5" />
                            </button>
                        </div>
                    </div>

                    <div class="h-[300px] w-full">
                        <client-only>
                            <apexchart type="area" height="300" :options="chartOptions" :series="series" />
                        </client-only>
                    </div>
                </div>
            </div>
            
            <!-- Lists Grid -->
            <div class="grid grid-cols-1 lg:grid-cols-3 gap-6 w-full">
                <div v-for="list in [
                    { name: 'Top Viewed', data: stats.top_viewed, icon: 'lucide:eye' },
                    { name: 'Top Streamed', data: stats.top_streamed, icon: 'lucide:play' },
                    { name: 'Top Downloaded', data: stats.top_downloaded, icon: 'lucide:download' }
                ]" :key="list.name" class="card bg-base-100 shadow-xl border border-base-200 w-full h-[380px]">
                    
                     <!-- Header INSIDE the card -->
                    <div class="p-4 border-b border-base-300 bg-base-200/30 flex items-center gap-3">
                        <Icon :name="list.icon" class="w-4 h-4 text-primary" />
                        <div>
                            <h3 class="card-title text-sm flex items-center gap-2">{{ list.name }}</h3>
                        </div>
                    </div>

                    <!-- Scrollable Body -->
                    <div class="overflow-y-auto grow divide-y divide-base-300/30">
                        <div v-for="(item, i) in list.data" :key="i" class="flex items-center justify-between p-4 hover:bg-base-200/50 transition-colors group">
                            <div class="flex items-center gap-3 overflow-hidden">
                                <span class="font-mono text-[10px] opacity-30 w-4">#{{ i + 1 }}</span>
                                <span class="text-xs font-bold group-hover:text-primary transition-colors">
                                    {{ item.label }}
                                </span>
                            </div>
                            <span data-v-709138aa="" class="text-xs font-bold text-secondary whitespace-nowrap ml-4">{{ item.count }}</span>
                        </div>
                        <!-- Empty State -->
                        <div v-if="!list.data?.length" class="flex flex-col items-center justify-center h-full opacity-20 py-20">
                            <Icon name="lucide:database" class="w-8 h-8 mb-2" />
                            <span class="text-xs font-bold uppercase italic">No data yet</span>
                        </div>
                    </div>
                </div>
            </div>


            <!-- Demographics Grid -->
            <!-- 2. Countries & Devices Row -->
            <div class="grid grid-cols-1 md:grid-cols-2 gap-6 w-full">
                <div v-for="list in [
                    { name: 'Top Countries', data: stats.top_countries, type: 'country' },
                    { name: 'Top Devices', data: stats.top_devices, type: 'device' }
                ]" :key="list.name" class="card bg-base-100 shadow-xl border border-base-200 w-full overflow-hidden h-[340px]">
                    <!-- Header INSIDE the card -->
                    <div class="p-4 border-b border-base-300 bg-base-200/30 flex items-center gap-3">
                        <Icon :name="list.type === 'country' ? 'lucide:globe' : 'lucide:smartphone'" class="w-4 h-4 text-primary" />
                        <div>
                            <h4 class="card-title text-sm flex items-center gap-2">{{ list.name }}</h4>
                        </div>
                    </div>

                    <div class="p-5 flex flex-col gap-5 overflow-y-auto">
                        <div v-for="item in list.data" :key="item.label" class="flex flex-col gap-2">
                            <div class="flex justify-between items-center text-[11px] font-bold px-1 uppercase">
                                <div class="flex items-center gap-2">
                                    <span v-if="list.type === 'country'">{{ getFlagEmoji(item.label) }}</span>
                                    <Icon v-else :name="getDeviceIcon(item.label)" class="w-3.5 h-3.5 opacity-60" />
                                    {{ item.label }}
                                </div>
                                <span class="font-mono opacity-60">{{ item.count }}</span>
                            </div>
                            <progress class="progress progress-secondary h-1.5 w-full" :value="item.count" :max="stats.total_views"></progress>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>