<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Seobu Safety GIS Dashboard Pro (Light V3.4)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <link rel="stylesheet" as="style" crossorigin href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.8/dist/web/static/pretendard.css" />
    
    <style>
        body { font-family: 'Pretendard', sans-serif; overflow: hidden; }

        /* 커스텀 스크롤바 */
        .scroller { overflow-y: auto; scrollbar-width: thin; scrollbar-color: #cbd5e1 #f1f5f9; }
        .scroller::-webkit-scrollbar { width: 6px; }
        .scroller::-webkit-scrollbar-track { background: #f8fafc; }
        .scroller::-webkit-scrollbar-thumb { background-color: #94a3b8; border-radius: 3px; }
        .scroller::-webkit-scrollbar-thumb:hover { background-color: #64748b; }

        /* 지도 마커 */
        .marker-pin {
            display: flex; align-items: center; justify-content: center;
            border-radius: 50%; border: 2px solid white;
            box-shadow: 0 2px 4px rgba(0,0,0,0.2);
            font-weight: 800; font-size: 10px; color: white;
            width: 32px; height: 32px; transition: transform 0.2s;
            white-space: nowrap;
        }
        .marker-pin:hover { transform: scale(1.2); z-index: 1000; }
        .marker-pin.active {
            transform: scale(1.3); border-color: #0f172a;
            box-shadow: 0 0 0 4px rgba(20, 184, 166, 0.4); z-index: 1001;
        }

        /* 테이블 */
        .spec-table td { padding: 8px 0; border-bottom: 1px solid #f1f5f9; font-size: 0.875rem; }
        .spec-table td:first-child { color: #64748b; font-weight: 500; width: 40%; }
        .spec-table td:last-child { color: #334155; font-weight: 700; text-align: right; }
        
        /* Select 스타일 */
        select {
            background-image: url("data:image/svg+xml,%3csvg xmlns='http://www.w3.org/2000/svg' fill='none' viewBox='0 0 20 20'%3e%3cpath stroke='%236b7280' stroke-linecap='round' stroke-linejoin='round' stroke-width='1.5' d='M6 8l4 4 4-4'/%3e%3c/svg%3e");
            background-position: right 0.2rem center;
            background-repeat: no-repeat;
            background-size: 1.2em 1.2em;
            padding-right: 1.5rem;
            -webkit-appearance: none; appearance: none;
        }
    </style>
</head>
<body class="flex h-screen w-screen bg-slate-50 text-slate-800">

    <aside class="w-80 flex-shrink-0 flex flex-col border-r border-slate-200 bg-white z-20 shadow-sm">
        <div class="h-14 flex items-center px-5 border-b border-slate-100 bg-white shrink-0">
            <div class="w-2 h-5 bg-teal-500 rounded mr-3"></div>
            <h1 class="font-bold text-base tracking-tight text-slate-800">서부안전 통합관제 V3.4</h1>
        </div>

        <div class="p-3 space-y-2 bg-slate-50 border-b border-slate-100 shrink-0">
            <div class="relative">
                <input type="text" id="searchInput" placeholder="정압기명, 주소 검색" onkeyup="filterList()"
                    class="w-full bg-white border border-slate-300 text-sm text-slate-800 px-3 py-2 rounded shadow-sm focus:outline-none focus:ring-1 focus:ring-teal-500 placeholder-slate-400">
                <svg class="w-4 h-4 text-slate-400 absolute right-3 top-2.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"></path></svg>
            </div>
            
            <div class="grid grid-cols-2 gap-2">
                <select id="loopFilter" onchange="filterList()" class="bg-white border border-slate-300 text-xs text-slate-600 rounded px-2 py-1.5 shadow-sm focus:outline-none cursor-pointer truncate">
                    <option value="all">LOOP 전체</option>
                </select>
                <select id="sectionFilter" onchange="filterList()" class="bg-white border border-slate-300 text-xs text-slate-600 rounded px-2 py-1.5 shadow-sm focus:outline-none cursor-pointer truncate">
                    <option value="all">구간 전체</option>
                </select>
            </div>
            <div class="flex gap-2">
                 <select id="modelFilter" onchange="filterList()" class="w-1/2 bg-white border border-slate-300 text-xs text-slate-600 rounded px-2 py-1.5 shadow-sm focus:outline-none cursor-pointer truncate">
                    <option value="all">모델 전체</option>
                </select>
                <label class="w-1/2 cursor-pointer bg-teal-600 hover:bg-teal-500 text-white text-xs font-bold rounded shadow-sm flex items-center justify-center transition-colors">
                    CSV 업로드
                    <input type="file" id="csvFileInput" accept=".csv" class="hidden" onchange="handleFileSelect(event)">
                </label>
            </div>
        </div>

        <div class="px-4 py-2 bg-white border-b border-slate-100 flex justify-between text-xs font-mono shrink-0">
            <span class="text-slate-500">TOTAL: <span id="total-count" class="text-slate-800 font-bold">0</span></span>
            <span class="text-rose-500">PRIORITY: <span id="priority-count" class="font-bold">0</span></span>
        </div>

        <div id="regulator-list" class="flex-1 scroller p-2 space-y-1 bg-slate-50">
            <div class="h-40 flex flex-col items-center justify-center text-slate-400 text-xs">
                <p>데이터가 없습니다.</p>
                <p class="mt-1">CSV 파일을 업로드해주세요.</p>
            </div>
        </div>
    </aside>

    <main class="flex-1 relative bg-slate-200">
        <div id="map" class="w-full h-full z-0"></div>
        <div class="absolute top-4 left-4 z-[400] bg-white/90 backdrop-blur px-3 py-2 rounded shadow-md border border-slate-100 text-slate-800">
            <div class="text-xs font-bold flex items-center gap-2">
                <span class="w-2 h-2 rounded-full bg-green-500 animate-pulse"></span>
                실시간 모니터링
            </div>
        </div>
    </main>

    <aside id="detail-panel" class="w-[400px] bg-white text-slate-800 flex flex-col border-l border-slate-200 shadow-xl transform transition-transform duration-300 translate-x-full absolute right-0 top-0 bottom-0 z-50 lg:relative lg:translate-x-0 lg:block">
        <div id="empty-state" class="absolute inset-0 flex flex-col items-center justify-center bg-slate-50/80 z-10">
            <div class="w-16 h-16 bg-white border border-slate-200 rounded-full flex items-center justify-center shadow-sm mb-4">
                <svg class="w-8 h-8 text-slate-300" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z"></path><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 11a3 3 0 11-6 0 3 3 0 016 0z"></path></svg>
            </div>
            <p class="text-sm font-bold text-slate-500">정압기를 선택하세요</p>
        </div>

        <div id="detail-content" class="flex-1 scroller flex flex-col h-full bg-white hidden">
            <div class="h-28 bg-slate-100 border-b border-slate-200 relative flex items-end p-5 shrink-0">
                <div class="absolute inset-0 opacity-[0.03]" style="background-image: radial-gradient(#000000 1px, transparent 1px); background-size: 12px 12px;"></div>
                <div class="relative w-full">
                    <div class="flex items-center gap-2 mb-1.5">
                        <span id="dt-loop" class="bg-teal-500 text-white text-[10px] font-bold px-1.5 py-0.5 rounded shadow-sm">LOOP -</span>
                        <span id="dt-inspection" class="text-white text-[10px] font-bold px-1.5 py-0.5 rounded shadow-sm hidden"></span>
                        <span id="dt-priority" class="bg-rose-500 text-white text-[10px] font-bold px-1.5 py-0.5 rounded shadow-sm hidden">우선방출</span>
                    </div>
                    <h2 id="dt-name" class="text-2xl font-bold text-slate-800 leading-none">정압기명</h2>
                </div>
                <button onclick="closeDetail()" class="absolute top-4 right-4 text-slate-400 hover:text-slate-800 lg:hidden">
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path></svg>
                </button>
            </div>

            <div class="px-5 py-3 border-b border-slate-100 bg-white flex items-start gap-2 shrink-0">
                <svg class="w-4 h-4 text-teal-500 mt-0.5 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z"></path></svg>
                <span id="dt-address" class="text-sm text-slate-600 font-medium break-keep">주소 정보 없음</span>
            </div>

            <div class="p-5 space-y-6">
                <div id="issue-box" class="hidden bg-rose-50 border border-rose-200 rounded p-3 shadow-sm">
                    <div class="flex gap-2">
                        <svg class="w-5 h-5 text-rose-500 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"></path></svg>
                        <div>
                            <div class="text-xs font-bold text-rose-700 uppercase">특이사항</div>
                            <div id="dt-issues" class="text-sm text-rose-800 mt-1 whitespace-pre-wrap font-medium"></div>
                        </div>
                    </div>
                </div>

                <div>
                    <h3 class="text-xs font-bold text-slate-400 uppercase mb-3 flex items-center gap-2">
                        <span class="w-1.5 h-1.5 rounded-full bg-teal-500"></span>운전 현황
                    </h3>
                    <div class="grid grid-cols-2 gap-3">
                        <div class="bg-white border border-slate-200 p-4 rounded-lg shadow-sm hover:border-teal-500 transition-colors">
                            <div class="text-[10px] text-slate-500 mb-1 font-medium">입구 압력 (MPa)</div>
                            <div class="text-xl font-extrabold text-slate-800 tracking-tight" id="dt-p-in">-</div>
                        </div>
                        <div class="bg-white border border-slate-200 p-4 rounded-lg shadow-sm hover:border-teal-500 transition-colors">
                            <div class="text-[10px] text-slate-500 mb-1 font-medium">출구 압력 (kPa)</div>
                            <div class="text-xl font-extrabold text-slate-800 tracking-tight" id="dt-p-out">-</div>
                        </div>
                        <div class="col-span-2 bg-teal-50 border border-teal-100 p-4 rounded-lg shadow-sm flex justify-between items-center">
                            <div>
                                <div class="text-[10px] text-teal-700 font-bold uppercase mb-1">센싱밸브 개도율</div>
                                <div class="text-xs text-teal-600 font-medium">Normal Range</div>
                            </div>
                            <div class="text-2xl font-extrabold text-teal-700"><span id="dt-open-rate">-</span>%</div>
                        </div>
                    </div>
                </div>

                <hr class="border-slate-100">

                <div>
                    <h3 class="text-xs font-bold text-slate-400 uppercase mb-3 flex items-center gap-2">
                        <span class="w-1.5 h-1.5 rounded-full bg-slate-400"></span>시설 제원
                    </h3>
                    <table class="w-full spec-table">
                        <tr><td>모델명</td><td id="dt-model">-</td></tr>
                        <tr><td>SSV (제조사)</td><td id="dt-ssv">-</td></tr>
                        <tr><td>필터규격</td><td id="dt-filter">-</td></tr>
                        <tr><td>설치일자</td><td id="dt-install-date">-</td></tr>
                        <tr><td>공사번호</td><td id="dt-const-no">-</td></tr>
                        <tr><td>시설물번호</td><td id="dt-fac-no"><span id="dt-fac-no-val" class="font-mono bg-slate-100 px-2 py-0.5 rounded text-xs">-</span></td></tr>
                        <tr><td>최대용량</td><td id="dt-capacity">-</td></tr>
                    </table>
                </div>

                <hr class="border-slate-100">

                <div>
                     <h3 class="text-xs font-bold text-slate-400 uppercase mb-3 flex items-center gap-2">
                        <span class="w-1.5 h-1.5 rounded-full bg-slate-400"></span>관리 정보
                    </h3>
                    <div class="bg-slate-50 border border-slate-100 rounded-lg p-4 space-y-3 text-sm">
                        <div class="flex justify-between">
                            <span class="text-slate-500 font-medium">담당자</span>
                            <span class="font-bold text-slate-700 flex items-center gap-2">
                                <div class="w-6 h-6 rounded-full bg-slate-200 flex items-center justify-center text-xs text-slate-500">
                                    <svg class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M16 7a4 4 0 11-8 0 4 4 0 018 0zM12 14a7 7 0 00-7 7h14a7 7 0 00-7-7z"></path></svg>
                                </div>
                                <span id="dt-manager">-</span>
                            </span>
                        </div>
                        <div class="flex justify-between">
                            <span class="text-slate-500 font-medium">최근 시공감리</span>
                            <span class="font-bold text-slate-700" id="dt-audit-date">-</span>
                        </div>
                        <div class="w-full h-px bg-slate-200 my-2"></div>
                        <div class="flex justify-between">
                            <span class="text-slate-500 font-medium">임대 형태</span>
                            <span class="font-bold text-slate-700" id="dt-contract-type">-</span>
                        </div>
                        <div class="flex justify-between">
                            <span class="text-slate-500 font-medium">계약 만료</span>
                            <span class="font-bold text-slate-700" id="dt-contract-end">-</span>
                        </div>
                        <div class="flex justify-between items-center">
                            <span class="text-slate-500 font-medium">계약 상태</span>
                            <span class="px-2.5 py-1 bg-teal-100 text-teal-700 rounded-md text-xs font-bold shadow-sm" id="dt-contract-status">-</span>
                        </div>
                    </div>
                </div>
                <div class="h-10"></div>
            </div>
        </div>
    </aside>

    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script>
        // --- State ---
        let regulators = [];
        let filteredRegulators = [];
        let map = null;
        let markers = L.layerGroup();
        let markerMap = {}; 
        let selectedId = null;

        const LOOP_COLORS = {
            '1': '#3b82f6', '2': '#10b981', '3': '#f59e0b', '4': '#8b5cf6', 
            '5': '#ef4444', '6': '#06b6d4', '7': '#ec4899', '8': '#f97316', 
            '단독': '#475569', 
            'default': '#94a3b8'
        };

        // --- Data Parser ---
        function parseCSV(csvString) {
            const rows = [];
            let currentRow = [];
            let inQuotes = false;
            let currentVal = '';
            
            for (let i = 0; i < csvString.length; i++) {
                const char = csvString[i];
                if (char === '"') inQuotes = !inQuotes;
                else if (char === ',' && !inQuotes) { currentRow.push(currentVal.trim()); currentVal = ''; }
                else if ((char === '\r' || char === '\n') && !inQuotes) {
                    if (currentVal || currentRow.length > 0) currentRow.push(currentVal.trim());
                    if (currentRow.length > 0) rows.push(currentRow);
                    currentRow = []; currentVal = '';
                    if (char === '\r' && csvString[i+1] === '\n') i++;
                } else { currentVal += char; }
            }
            if (currentVal || currentRow.length > 0) { currentRow.push(currentVal.trim()); rows.push(currentRow); }

            if (rows.length < 2) return [];

            const rawHeaders = rows[0];
            const headers = rawHeaders.map(h => h.replace(/^"|"$/g, '').replace(/^\ufeff/, '').trim());
            
            const findCol = (keywords) => headers.findIndex(h => keywords.some(k => h.toLowerCase().includes(k)));
            
            const idx = {
                lat: findCol(['lat', '위도']),
                lon: findCol(['lon', '경도']),
                name: findCol(['정압기명', '시설물명', 'name']),
                addr: findCol(['주소', '설치장소']),
                loop: findCol(['loop', '루프']),
                priority: findCol(['우선방출']),
                p_in: findCol(['입구압력']),
                p_out: findCol(['출구압력']),
                model: findCol(['정압기모델']),
                ssv: findCol(['ssv']),
                issues: findCol(['특이사항']),
                section: findCol(['구간', 'section']),
                auditDate: findCol(['시공감리', '감리일자', '설치일자']) // 감리일자 또는 설치일자
            };

            const currentYear = new Date().getFullYear();

            return rows.slice(1).map((row, i) => {
                const getVal = (colIdx) => colIdx !== -1 && row[colIdx] ? row[colIdx].replace(/^"|"$/g, '') : '';
                const cleanFloat = (val) => parseFloat(val.replace(/[^\d.-]/g, ''));
                const lat = cleanFloat(getVal(idx.lat));
                const lon = cleanFloat(getVal(idx.lon));

                if (isNaN(lat) || isNaN(lon) || lat === 0 || lon === 0) return null;

                const fullData = {};
                headers.forEach((h, colI) => fullData[h] = row[colI] ? row[colI].replace(/^"|"$/g, '') : '');

                // LOOP 파싱
                let rawLoop = getVal(idx.loop);
                let loopVal = 'default';
                if (rawLoop.includes('단독')) {
                    loopVal = '단독';
                } else {
                    let nums = rawLoop.replace(/[^0-9]/g, '');
                    if (nums) loopVal = nums;
                }

                // 점검유형 자동 계산 (분해 vs 필터)
                // 현재년도와 시공감리년도(없으면 설치일자)의 홀짝 비교
                let inspectionType = '정보없음';
                const auditDateStr = getVal(idx.auditDate);
                if (auditDateStr) {
                    // YYYY로 시작하는 4자리 추출
                    const yearMatch = auditDateStr.match(/^\d{4}/);
                    if (yearMatch) {
                        const auditYear = parseInt(yearMatch[0], 10);
                        const isCurrentEven = (currentYear % 2 === 0);
                        const isAuditEven = (auditYear % 2 === 0);
                        
                        if (isCurrentEven === isAuditEven) {
                            inspectionType = '분해점검'; // 짝=짝 or 홀=홀
                        } else {
                            inspectionType = '필터점검'; // 짝!=홀
                        }
                    }
                }

                return {
                    id: i, fullData: fullData, lat: lat, lon: lon,
                    name: getVal(idx.name) || '이름없음',
                    addr: getVal(idx.addr) || '-',
                    loop: loopVal,
                    isPriority: getVal(idx.priority).includes('우선'),
                    p_in: getVal(idx.p_in) || '-',
                    p_out: getVal(idx.p_out) || '-',
                    model: getVal(idx.model) || '-',
                    section: getVal(idx.section) || '미지정',
                    ssv: getVal(idx.ssv) || '-',
                    issues: getVal(idx.issues),
                    inspType: inspectionType // 추가된 필드
                };
            }).filter(item => item !== null);
        }

        function initData(csvContent) {
            regulators = parseCSV(csvContent);
            if (regulators.length === 0) {
                alert("CSV 파일에서 유효한 좌표 데이터를 찾을 수 없습니다.");
                return;
            }
            filteredRegulators = regulators;
            
            document.getElementById('total-count').textContent = regulators.length;
            document.getElementById('priority-count').textContent = regulators.filter(r => r.isPriority).length;

            const fillSelect = (id, data, defaultText) => {
                const el = document.getElementById(id);
                el.innerHTML = `<option value="all">${defaultText}</option>`;
                data.forEach(item => { if(item && item !== 'default' && item !== '미지정') el.add(new Option(item, item)); });
            };

            const loops = [...new Set(regulators.map(r => r.loop))].sort((a,b) => {
                if(a === '단독') return 1; if(b === '단독') return -1;
                return parseInt(a) - parseInt(b);
            });
            fillSelect('loopFilter', loops, 'LOOP 전체');

            const sections = [...new Set(regulators.map(r => r.section))].sort();
            fillSelect('sectionFilter', sections, '구간 전체');

            const models = [...new Set(regulators.map(r => r.model))].sort();
            fillSelect('modelFilter', models, '모델 전체');

            renderList();
            renderMap();
        }

        // --- Rendering ---
        function renderList() {
            const container = document.getElementById('regulator-list');
            container.innerHTML = '';
            
            if (filteredRegulators.length === 0) {
                container.innerHTML = `<div class="text-slate-400 text-center text-xs py-4">일치하는 항목이 없습니다.</div>`;
                return;
            }

            filteredRegulators.forEach(r => {
                const color = LOOP_COLORS[r.loop] || LOOP_COLORS['default'];
                const isSelected = selectedId === r.id;
                
                // 점검유형 배지 색상 결정
                let inspBadgeClass = "bg-slate-100 text-slate-500";
                if (r.inspType === '분해점검') inspBadgeClass = "bg-indigo-100 text-indigo-700 border border-indigo-200";
                else if (r.inspType === '필터점검') inspBadgeClass = "bg-emerald-100 text-emerald-700 border border-emerald-200";

                const el = document.createElement('div');
                el.className = `p-3 rounded-lg cursor-pointer transition-all border ${isSelected ? 'bg-teal-50 border-teal-500 shadow-sm ring-1 ring-teal-500' : 'bg-white border-slate-200 hover:border-teal-400 hover:shadow-sm'}`;
                el.onclick = () => selectRegulator(r.id);
                
                el.innerHTML = `
                    <div class="flex items-center justify-between mb-1.5">
                        <div class="flex items-center gap-2.5">
                            <span class="text-[10px] font-extrabold px-2 py-0.5 rounded-full text-white shadow-sm shrink-0" style="background-color: ${color}">${r.loop === 'default' ? '-' : r.loop}</span>
                            <span class="text-sm font-bold text-slate-800 truncate w-36">${r.name}</span>
                        </div>
                        ${r.isPriority ? '<span class="text-[10px] font-bold text-rose-600 bg-rose-100 px-1.5 py-0.5 rounded-md border border-rose-200">우선</span>' : ''}
                    </div>
                    <div class="text-xs text-slate-500 truncate pl-1 font-medium mb-1.5">${r.addr}</div>
                    
                    <div class="flex items-center justify-between pl-1">
                        <div class="flex gap-1">
                             ${r.section !== '미지정' ? `<span class="text-[9px] bg-slate-100 text-slate-500 px-1.5 py-0.5 rounded font-medium">${r.section}</span>` : ''}
                        </div>
                        <span class="text-[9px] font-bold px-1.5 py-0.5 rounded ${inspBadgeClass}">${r.inspType}</span>
                    </div>
                `;
                container.appendChild(el);
            });
        }

        function renderMap() {
            if (!map) return;
            markers.clearLayers();
            markerMap = {};
            const bounds = [];

            filteredRegulators.forEach(r => {
                const color = LOOP_COLORS[r.loop] || LOOP_COLORS['default'];
                const isSelected = selectedId === r.id;
                const iconHtml = `<div class="marker-pin ${isSelected ? 'active' : ''}" style="background-color: ${color};">${r.loop==='default'?'':r.loop}</div>`;
                const icon = L.divIcon({ html: iconHtml, className: '', iconSize: [30, 30], iconAnchor: [15, 15] });
                const marker = L.marker([r.lat, r.lon], { icon: icon });
                marker.on('click', () => selectRegulator(r.id));
                markers.addLayer(marker);
                markerMap[r.id] = marker;
                bounds.push([r.lat, r.lon]);
            });

            markers.addTo(map);
            if (bounds.length > 0 && !selectedId) map.fitBounds(bounds, { padding: [50, 50] });
        }

        function filterList() {
            const query = document.getElementById('searchInput').value.toLowerCase();
            const loopVal = document.getElementById('loopFilter').value;
            const sectionVal = document.getElementById('sectionFilter').value;
            const modelVal = document.getElementById('modelFilter').value;
            
            filteredRegulators = regulators.filter(r => {
                const matchQ = r.name.toLowerCase().includes(query) || r.addr.toLowerCase().includes(query);
                const matchL = loopVal === 'all' || r.loop === loopVal;
                const matchS = sectionVal === 'all' || r.section === sectionVal;
                const matchM = modelVal === 'all' || r.model === modelVal;
                return matchQ && matchL && matchS && matchM;
            });
            renderList();
            renderMap();
        }

        function selectRegulator(id) {
            selectedId = id;
            const r = regulators.find(x => x.id === id);
            
            renderList();
            renderMap();

            map.invalidateSize();
            map.flyTo([r.lat, r.lon], 16, { animate: true, duration: 1 });
            
            document.getElementById('empty-state').classList.add('hidden');
            document.getElementById('detail-content').classList.remove('hidden');
            document.getElementById('detail-panel').classList.remove('translate-x-full');

            const d = r.fullData;
            document.getElementById('dt-name').textContent = r.name;
            document.getElementById('dt-address').textContent = r.addr;
            document.getElementById('dt-loop').textContent = `LOOP ${r.loop}`;
            document.getElementById('dt-loop').style.backgroundColor = LOOP_COLORS[r.loop];
            
            const pBadge = document.getElementById('dt-priority');
            r.isPriority ? pBadge.classList.remove('hidden') : pBadge.classList.add('hidden');

            // 점검 유형 배지 업데이트
            const iBadge = document.getElementById('dt-inspection');
            iBadge.textContent = r.inspType;
            iBadge.classList.remove('hidden', 'bg-indigo-500', 'bg-emerald-500', 'bg-slate-400');
            if (r.inspType === '분해점검') iBadge.classList.add('bg-indigo-500');
            else if (r.inspType === '필터점검') iBadge.classList.add('bg-emerald-500');
            else iBadge.classList.add('bg-slate-400');

            const setText = (id, val) => document.getElementById(id).textContent = val || '-';
            
            setText('dt-p-in', r.p_in);
            setText('dt-p-out', r.p_out);
            setText('dt-open-rate', (d['센싱밸브 개도율'] || d['개도율'] || '').replace('%',''));
            
            setText('dt-model', r.model);
            setText('dt-ssv', r.ssv + (d['SSV제조사'] ? ` (${d['SSV제조사']})` : ''));
            setText('dt-filter', d['필터규격']);
            setText('dt-install-date', d['정압기설치일'] || d['설치일자']);
            setText('dt-const-no', d['공사번호']);
            setText('dt-fac-no-val', d['시설물번호'] || '-');
            setText('dt-capacity', d['최대용량(m3/hr)'] || d['최대용량']);

            setText('dt-manager', d['담당자']);
            setText('dt-audit-date', d['시공감리일자']);
            setText('dt-contract-type', d['임대형태']);
            setText('dt-contract-end', d['임차료 계약종료일'] || d['계약종료일']);
            setText('dt-contract-status', d['계약상태']);

            const issueBox = document.getElementById('issue-box');
            if (r.issues && r.issues.trim()) {
                issueBox.classList.remove('hidden');
                document.getElementById('dt-issues').textContent = r.issues;
            } else {
                issueBox.classList.add('hidden');
            }
        }

        function closeDetail() {
            document.getElementById('detail-panel').classList.add('translate-x-full');
            selectedId = null;
            renderList();
            renderMap();
        }

        function handleFileSelect(e) {
            const file = e.target.files[0];
            if(!file) return;
            const reader = new FileReader();
            reader.onload = (evt) => initData(evt.target.result);
            reader.readAsText(file, 'euc-kr');
        }

        document.addEventListener('DOMContentLoaded', () => {
            map = L.map('map', { zoomControl: false }).setView([36.5, 127.5], 7);
            L.control.zoom({ position: 'topright' }).addTo(map);
            L.tileLayer('https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png', {
                attribution: '&copy; OpenStreetMap & CARTO',
                maxZoom: 19
            }).addTo(map);
        });
    </script>
</body>
</html>
