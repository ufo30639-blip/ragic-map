<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>房源地圖系統 V10 | 待租單純圖釘版</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700;900&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Noto Sans TC', sans-serif; }
        .scroller::-webkit-scrollbar { width: 6px; }
        .scroller::-webkit-scrollbar-thumb { background-color: #cbd5e1; border-radius: 3px; }
        .scroller::-webkit-scrollbar-track { background-color: #f1f5f9; }
    </style>
</head>
<body class="bg-gray-50 h-screen flex flex-col overflow-hidden">

    <script>
        // ================= 設定區 =================
        const GOOGLE_MAPS_KEY = "AIzaSyC5L1qxNCMRuPZxTWF5GctwTX6ZfnErtGk"; 
        
        const RAGIC_API_KEY = "ZGlvWFNFb1FKazM4OU0xd1BZR2poSEcwbExsc2JsaHB1WVZRaVBrWEp1WDBubGhJUTZvWFhoeXBza2hnbGtTTFF1NTN0UlQzR2U2TGgvbXVvR0RIdkE9PQ=="; 
        const RAGIC_ENDPOINT = "https://ap12.ragic.com/xiangyu/forms2/48";

        // 欄位 ID 對照
        const F_ID_NAME      = "1012261"; // 社區名稱
        const F_ID_STATUS    = "1012259"; // 房屋狀態
        const F_ID_ADDR_1    = "1012351"; // 地址 (公式)
        const F_ID_ADDR_2    = "1012266"; // 地址 (手動)
        const F_ID_PRICE     = "1012287"; // 租金
        const F_ID_LAT       = "1012352"; // 緯度
        const F_ID_LNG       = "1012353"; // 經度
        const F_ID_ROOM      = "1012274"; // 格局
        const F_ID_SIZE      = "1012269"; // 坪數
        const F_ID_DEV_NAME  = "1012316"; // A32 開發人員
        const F_ID_DEV_PHONE = "1012317"; // C32 開發電話
    </script>

    <!-- 頂部導航 -->
    <header class="bg-white border-b border-gray-200 z-20 flex-none h-16 flex items-center justify-between px-6 shadow-sm">
        <div class="flex items-center gap-3">
            <div class="bg-blue-600 p-2 rounded-lg text-white">
                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z"></path><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 11a3 3 0 11-6 0 3 3 0 016 0z"></path></svg>
            </div>
            <div>
                <h1 class="text-xl font-bold text-gray-800 tracking-tight">房源地圖系統</h1>
                <p class="text-xs text-gray-500">僅顯示「未出租」物件</p>
            </div>
        </div>
        
        <div class="flex items-center gap-3">
            <button id="btn-fix-coords" onclick="startClientSideGeocoding()" class="hidden bg-yellow-500 hover:bg-yellow-600 text-white px-4 py-2 rounded-lg text-sm font-bold shadow-md transition flex items-center gap-2 animate-bounce">
                <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 10V3L4 14h7v7l9-11h-7z"></path></svg>
                <span id="btn-fix-text">修復缺座標資料</span>
            </button>

            <button onclick="fetchData()" class="bg-gray-100 hover:bg-gray-200 text-gray-700 px-4 py-2 rounded-lg text-sm font-medium transition flex items-center gap-2">
                <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"></path></svg>
                重新整理
            </button>
        </div>
    </header>

    <!-- 主畫面 -->
    <div class="flex-1 flex overflow-hidden relative">
        <!-- 左側列表 -->
        <aside class="w-full md:w-[400px] bg-white border-r border-gray-200 flex flex-col z-10 transition-transform duration-300 absolute md:relative h-full transform -translate-x-full md:translate-x-0 shadow-xl md:shadow-none" id="sidebar">
            <div class="p-4 border-b border-gray-100 bg-white z-10">
                <div class="relative group">
                    <input type="text" id="searchInput" placeholder="搜尋社區、地址、業務..." 
                        class="w-full pl-10 pr-4 py-2.5 bg-gray-50 border border-gray-200 rounded-xl text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:bg-white transition">
                    <svg class="w-5 h-5 text-gray-400 absolute left-3 top-3 group-focus-within:text-blue-500 transition" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"></path></svg>
                </div>
            </div>
            <div id="property-list" class="flex-1 overflow-y-auto scroller bg-white">
                <div class="flex flex-col items-center justify-center h-64 text-gray-400 gap-3">
                    <div class="w-8 h-8 border-4 border-blue-200 border-t-blue-500 rounded-full animate-spin"></div>
                    <span class="text-sm">資料讀取中...</span>
                </div>
            </div>
            <div class="px-4 py-3 bg-gray-50 border-t border-gray-200 text-xs text-gray-500 flex justify-between items-center">
                <span id="count-display">準備中...</span>
                <div class="flex gap-3">
                    <span class="flex items-center gap-1"><span class="w-2 h-2 rounded-full bg-blue-600"></span> 地圖顯示</span>
                    <span class="flex items-center gap-1"><span class="w-2 h-2 rounded-full bg-gray-300"></span> 缺座標</span>
                </div>
            </div>
        </aside>

        <!-- 地圖 -->
        <main class="flex-1 relative bg-gray-100">
            <div id="map" class="w-full h-full"></div>
            <button id="toggleSidebar" class="md:hidden absolute top-4 left-4 bg-white p-3 rounded-full shadow-lg z-20 text-gray-700 hover:bg-gray-50">
                <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"></path></svg>
            </button>
        </main>
    </div>

    <script>
        let map;
        let geocoder;
        let allProperties = [];
        let infoWindow;

        function loadGoogleMaps() {
            const script = document.createElement('script');
            script.src = `https://maps.googleapis.com/maps/api/js?key=${GOOGLE_MAPS_KEY}&callback=initApp`;
            script.defer = true;
            document.head.appendChild(script);
        }

        function initApp() {
            initMap();
            fetchData();
        }

        function initMap() {
            map = new google.maps.Map(document.getElementById("map"), {
                center: { lat: 25.0478, lng: 121.5170 },
                zoom: 12,
                mapTypeControl: false,
                streetViewControl: false,
                fullscreenControl: false,
                clickableIcons: false
            });
            geocoder = new google.maps.Geocoder();
            infoWindow = new google.maps.InfoWindow({ maxWidth: 280 });
        }

        async function fetchData() {
            const listContainer = document.getElementById('property-list');
            const btnFix = document.getElementById('btn-fix-coords');
            
            try {
                const encodedKey = encodeURIComponent(RAGIC_API_KEY);
                const url = `${RAGIC_ENDPOINT}?api&naming=EID&APIKey=${encodedKey}`;

                const response = await fetch(url);
                if (!response.ok) throw new Error(`API Error: ${response.status}`);

                const data = await response.json();
                const rawList = Object.values(data);
                
                // 1. 資料清洗與轉換
                // 2. ★ 過濾：只保留狀態包含「未出租」的資料
                allProperties = rawList.map(item => {
                    const lat = parseFloat(item[F_ID_LAT]);
                    const lng = parseFloat(item[F_ID_LNG]);
                    const hasCoords = !isNaN(lat) && !isNaN(lng) && lat !== 0;
                    const address = item[F_ID_ADDR_1] || item[F_ID_ADDR_2] || '';

                    return {
                        id: item['_ragicId'],
                        name: item[F_ID_NAME] || '未命名物件',
                        address: address,
                        lat: hasCoords ? lat : null,
                        lng: hasCoords ? lng : null,
                        hasCoords: hasCoords,
                        status: item[F_ID_STATUS] || '未知',
                        price: item[F_ID_PRICE] || 0,
                        room: item[F_ID_ROOM] || '格局未填',
                        size: item[F_ID_SIZE] || '',
                        devName: item[F_ID_DEV_NAME] || '未指定',
                        devPhone: item[F_ID_DEV_PHONE] || '',
                        marker: null // 預留存儲地圖標記的變數
                    };
                }).filter(p => p.status.includes('未出租'));

                updateUI();

                const missingCount = allProperties.filter(p => !p.hasCoords && p.address).length;
                if (missingCount > 0) {
                    btnFix.classList.remove('hidden');
                    document.getElementById('btn-fix-text').innerText = `地圖上有 ${missingCount} 筆資料缺座標，點我修復`;
                } else {
                    btnFix.classList.add('hidden');
                }

            } catch (error) {
                console.error(error);
                listContainer.innerHTML = `<div class="p-6 text-center text-red-500">連線失敗: ${error.message}</div>`;
            }
        }

        function updateUI() {
            // 清除現有標記
            allProperties.forEach(p => {
                if (p.marker) p.marker.setMap(null);
            });

            const listContainer = document.getElementById('property-list');
            listContainer.innerHTML = '';
            
            const bounds = new google.maps.LatLngBounds();
            let validCount = 0;

            if (allProperties.length === 0) {
                listContainer.innerHTML = '<div class="p-6 text-center text-gray-500">目前沒有未出租的物件</div>';
                document.getElementById('count-display').innerHTML = `總計 <b>0</b> 筆`;
                return;
            }

            allProperties.forEach((prop) => {
                const el = document.createElement('div');
                const priceFormatted = prop.price ? `$${parseInt(prop.price).toLocaleString()}` : "面議";
                
                el.className = `group p-4 border-b border-gray-100 bg-white hover:bg-blue-50/50 transition cursor-pointer relative ${prop.hasCoords ? '' : 'opacity-60 grayscale'}`;
                
                // 狀態標籤 (因為全都是未出租，統一為醒目的紅色)
                const statusBadge = prop.hasCoords 
                    ? `<span class="absolute top-4 right-4 text-[10px] bg-red-100 text-red-700 px-2 py-1 rounded-full border border-red-200 font-bold">${prop.status}</span>`
                    : `<span class="absolute top-4 right-4 text-[10px] bg-gray-200 text-gray-600 px-2 py-1 rounded-full font-bold border border-gray-300">缺座標</span>`;

                el.innerHTML = `
                    <div class="flex flex-col gap-1 pr-14">
                        <h4 class="font-bold text-gray-800 text-base line-clamp-1 group-hover:text-blue-600 transition">${prop.name}</h4>
                        <p class="text-xs text-gray-500 line-clamp-1 flex items-center gap-1" title="${prop.address}">
                            <svg class="w-3 h-3 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z"></path><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 11a3 3 0 11-6 0 3 3 0 016 0z"></path></svg>
                            ${prop.address}
                        </p>
                        
                        <div class="flex justify-between items-end mt-2">
                            <div class="text-xs text-gray-600 font-medium bg-gray-100 px-2 py-1 rounded">
                                ${prop.room} ${prop.size ? '· '+prop.size+'坪' : ''}
                            </div>
                            <span class="font-bold text-red-500 text-lg">${priceFormatted}</span>
                        </div>

                        <!-- 開發人員資訊區塊 -->
                        <div class="mt-3 pt-2 border-t border-gray-100 flex items-center gap-4 text-xs text-gray-500">
                            <div class="flex items-center gap-1" title="開發人員">
                                <svg class="w-3 h-3 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M16 7a4 4 0 11-8 0 4 4 0 018 0zM12 14a7 7 0 00-7 7h14a7 7 0 00-7-7z"></path></svg>
                                <span>${prop.devName}</span>
                            </div>
                            ${prop.devPhone ? `
                            <div class="flex items-center gap-1" title="聯絡電話">
                                <svg class="w-3 h-3 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 5a2 2 0 012-2h3.28a1 1 0 01.948.684l1.498 4.493a1 1 0 01-.502 1.21l-2.257 1.13a11.042 11.042 0 005.516 5.516l1.13-2.257a1 1 0 011.21-.502l4.493 1.498a1 1 0 01.684.949V19a2 2 0 01-2 2h-1C9.716 21 3 14.284 3 6V5z"></path></svg>
                                <a href="tel:${prop.devPhone}" class="hover:text-blue-600 hover:underline transition font-medium" onclick="event.stopPropagation()">${prop.devPhone}</a>
                            </div>` : ''}
                        </div>
                    </div>
                    ${statusBadge}
                `;

                if (prop.hasCoords) {
                    validCount++;
                    // ★ 使用最基礎預設的 Google Maps 小圖釘
                    prop.marker = new google.maps.Marker({
                        position: { lat: prop.lat, lng: prop.lng },
                        map: map,
                        title: prop.name
                    });

                    prop.marker.addListener("click", () => {
                        const priceFormatted = prop.price ? `$${parseInt(prop.price).toLocaleString()}` : "面議";
                        
                        const content = `
                            <div class="font-sans p-1 min-w-[200px]">
                                <h3 class="font-bold text-gray-900 text-base truncate mb-1 pr-4">${prop.name}</h3>
                                <div class="text-red-500 font-bold text-lg mb-2">${priceFormatted}</div>
                                <div class="space-y-1 text-xs text-gray-600 mb-3 border-t border-b border-gray-100 py-2">
                                    <div class="flex items-start gap-1">
                                        <span>📍</span>
                                        <span>${prop.address}</span>
                                    </div>
                                    <div class="flex items-center gap-1">
                                        <span>🏠</span>
                                        <span>${prop.room} ${prop.size ? '· '+prop.size+'坪' : ''}</span>
                                    </div>
                                </div>
                                <div class="flex justify-between items-center bg-gray-50 p-2 rounded mb-2 text-xs text-gray-700">
                                    <span class="font-bold">${prop.devName}</span>
                                    <a href="tel:${prop.devPhone}" class="text-blue-600 hover:underline font-bold">${prop.devPhone}</a>
                                </div>
                                <a href="https://ap12.ragic.com/xiangyu/forms2/48/${prop.id}" target="_blank" class="block mt-2 text-center w-full bg-blue-600 text-white text-xs py-2 rounded hover:bg-blue-700 transition">開啟 Ragic 資料</a>
                            </div>
                        `;
                        infoWindow.setContent(content);
                        infoWindow.open(map, prop.marker);
                    });

                    el.addEventListener('click', () => {
                        map.panTo({ lat: prop.lat, lng: prop.lng });
                        map.setZoom(16);
                        infoWindow.close();
                        google.maps.event.trigger(prop.marker, 'click');
                        if(window.innerWidth < 768) document.getElementById('sidebar').classList.add('-translate-x-full');
                    });
                    
                    bounds.extend(prop.marker.getPosition());
                }

                listContainer.appendChild(el);
            });

            document.getElementById('count-display').innerHTML = `總計 <b>${allProperties.length}</b> 筆 (顯示 ${validCount} 筆)`;

            if (validCount > 0) map.fitBounds(bounds);
        }

        async function startClientSideGeocoding() {
            const btn = document.getElementById('btn-fix-coords');
            const btnText = document.getElementById('btn-fix-text');
            btn.classList.remove('animate-bounce');
            btn.classList.add('opacity-75', 'cursor-wait');
            btn.disabled = true;

            const targets = allProperties.filter(p => !p.hasCoords && p.address);
            
            for (let i = 0; i < targets.length; i++) {
                const prop = targets[i];
                btnText.innerText = `修復中... ${i+1}/${targets.length}`;
                let cleanAddr = prop.address.indexOf("樓") > -1 ? prop.address.split("樓")[0] : prop.address;
                try {
                    const result = await geocodeAddress(cleanAddr);
                    prop.lat = result.lat;
                    prop.lng = result.lng;
                    prop.hasCoords = true;
                } catch (err) { console.warn(err); }
                await new Promise(r => setTimeout(r, 300));
            }
            
            btnText.innerText = "修復完成！";
            setTimeout(() => { 
                updateUI(); // 重新渲染列表與所有圖釘
                btn.classList.add('hidden'); 
            }, 1000);
        }

        function geocodeAddress(address) {
            return new Promise((resolve, reject) => {
                geocoder.geocode({ 'address': address }, (results, status) => {
                    if (status === 'OK') resolve({ lat: results[0].geometry.location.lat(), lng: results[0].geometry.location.lng() });
                    else reject(status);
                });
            });
        }
        
        // ★ 搜尋功能：連動隱藏列表與地圖圖釘
        document.getElementById('searchInput').addEventListener('input', (e) => {
            const term = e.target.value.toLowerCase();
            const items = document.getElementById('property-list').children;
            
            let visibleBounds = new google.maps.LatLngBounds();
            let hasVisibleMarker = false;

            allProperties.forEach((prop, idx) => {
                const visible = prop.name.toLowerCase().includes(term) || 
                              prop.address.toLowerCase().includes(term) ||
                              prop.devName.toLowerCase().includes(term); 
                
                // 1. 控制左側列表顯示/隱藏
                if (items[idx]) items[idx].style.display = visible ? 'flex' : 'none';
                
                // 2. 控制地圖圖釘顯示/隱藏
                if (prop.marker) {
                    prop.marker.setVisible(visible);
                    // 若顯示則加入範圍，方便自動縮放
                    if (visible) {
                        visibleBounds.extend(prop.marker.getPosition());
                        hasVisibleMarker = true;
                    }
                }
            });

            // (選用) 搜尋時若有符合條件的圖釘，將地圖縮放到剛好能顯示所有結果
            // 如果搜尋文字清空，也會自動拉回顯示所有點位
            if (hasVisibleMarker) {
                map.fitBounds(visibleBounds);
                // 避免單一點位時縮放太近
                if (map.getZoom() > 16) {
                    map.setZoom(16);
                }
            }
        });

        document.getElementById('toggleSidebar').addEventListener('click', () => document.getElementById('sidebar').classList.toggle('-translate-x-full'));
        loadGoogleMaps();
    </script>
</body>
</html>
