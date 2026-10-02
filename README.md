/* 
 * MEGA GESTOR DE PLANTILLAS AM + COORDENADAS + LÍMITE DE GRANJA
 */
(async function() {
    // Bases de datos locales
    const DB_TPL = 'tw_mega_templates';
    const DB_GRP = 'tw_mega_groups_assign';
    const DB_CRD = 'tw_mega_coords_assign';
    const DB_REQ = 'tw_mega_requests';
    const DB_SET = 'tw_mega_settings';

    let templates = JSON.parse(localStorage.getItem(DB_TPL) || '{}');
    let groupAssigns = JSON.parse(localStorage.getItem(DB_GRP) || '{}');
    let coordAssigns = JSON.parse(localStorage.getItem(DB_CRD) || '{}');
    let pendingReqs = JSON.parse(localStorage.getItem(DB_REQ) || '{}');
    let settings = JSON.parse(localStorage.getItem(DB_SET) || '{"maxFarm": 30}');

    // ==========================================
    // 1. MODO MERCADO: ENVÍO RÁPIDO AUTOMATIZADO
    // ==========================================
    if (game_data.screen === 'market') {
        if (window.location.href.includes('try=confirm_send')) {
            let btn = $('#troop_confirm_submit');
            if (btn.length) { btn.focus(); if(window.UI) UI.SuccessMessage('¡Pulsa ENTER para Confirmar!'); return; }
        }

        if (!game_data.mode || game_data.mode === 'send') {
            let targetCoord = null, targetNeeds = null;
            let currentCoord = game_data.village.coord;

            for (let coord in pendingReqs) {
                if (coord === currentCoord) continue;
                if (pendingReqs[coord].w > 0 || pendingReqs[coord].s > 0 || pendingReqs[coord].i > 0) {
                    targetCoord = coord; targetNeeds = pendingReqs[coord]; break;
                }
            }

            if (!targetCoord) { if(window.UI) UI.SuccessMessage('✅ ¡NO HAY MÁS PUEBLOS QUE NECESITEN RECURSOS!'); return; }

            $('.target-input-field').val(targetCoord);

            let maxM = (parseInt($('#market_merchant_available_count').text(), 10) || 0) * 1000;
            let vw = game_data.village.wood, vs = game_data.village.stone, vi = game_data.village.iron;

            let mw = Math.min(targetNeeds.w, vw), ms = Math.min(targetNeeds.s, vs), mi = Math.min(targetNeeds.i, vi);

            if (mw + ms + mi > maxM) {
                let cap = maxM;
                mw = Math.min(mw, cap); cap -= mw;
                ms = Math.min(ms, cap); cap -= ms;
                mi = Math.min(mi, cap); cap -= mi;
            }

            if (mw + ms + mi === 0) { if(window.UI) UI.ErrorMessage('No hay recursos/mercaderes suficientes aquí. Cambia de pueblo.'); return; }

            $('input[name="wood"]').val(mw); $('input[name="stone"]').val(ms); $('input[name="iron"]').val(mi);

            targetNeeds.w -= mw; targetNeeds.s -= ms; targetNeeds.i -= mi;
            if (targetNeeds.w <= 0 && targetNeeds.s <= 0 && targetNeeds.i <= 0) delete pendingReqs[targetCoord];
            else pendingReqs[targetCoord] = targetNeeds;
            localStorage.setItem(DB_REQ, JSON.stringify(pendingReqs));

            $('input[type="submit"]').focus();
            if(window.UI) UI.SuccessMessage(`Enviando a ${targetCoord}... ¡Pulsa ENTER!`);
            return;
        }
    }

    // ==========================================
    // 2. MODO GESTOR: INTERFAZ Y CÁLCULOS
    // ==========================================
    if ($('#mega_am_panel').length > 0) return;

    let groups = {'0': 'Todos los pueblos'};
    try {
        let overHtml = await $.get(game_data.link_base_pure + 'overview_villages');
        $(overHtml).find('a[href*="&group="]').each(function() {
            let match = $(this).attr('href').match(/&group=(\d+)/);
            let name = $(this).text().trim().replace(/\[|\]/g, '');
            if (match && name && name !== '>' && name !== '<' && name.toLowerCase() !== 'all') groups[match[1]] = name;
        });
    } catch(e) {}

    let html = `
        <div id="mega_am_panel" style="position:fixed; top:40px; left:50%; transform:translateX(-50%); width:750px; max-height:90vh; background:#f4e4bc; border:3px solid #603000; border-radius:10px; z-index:99999; padding:20px; overflow-y:auto; box-shadow: 0px 10px 25px rgba(0,0,0,0.8); color:#333;">
            <h2 style="margin-top:0; border-bottom:2px solid #603000; padding-bottom:5px; text-align:center;">
                👑 Mega Gestor de Edificios y Envíos
                <a href="#" onclick="$('#mega_am_panel').remove(); return false;" style="float:right; font-size:16px; color:#c00; text-decoration:none;">✖</a>
            </h2>
            
            <div style="display:flex; gap:10px;">
                <!-- COLUMNA 1: Plantillas y Ajustes -->
                <div style="flex:1; background:#e3d5b3; padding:10px; border-radius:5px; border:1px solid #7d510f;">
                    <h4 style="margin-top:0; color:#603000;">1. Plantillas</h4>
                    <input type="text" id="t_name" placeholder="Ej: Defensiva" style="width:100%; margin-bottom:5px;">
                    <textarea id="t_b64" placeholder="Código Base64..." style="width:100%; height:40px; margin-bottom:5px; font-size:10px;"></textarea>
                    <button class="btn" id="btn_save_t" style="width:100%;">Guardar Plantilla</button>
                    <div style="font-size:11px; margin-top:5px; max-height:40px; overflow-y:auto;"><b>Guardadas:</b> <span id="t_list" style="color:blue;"></span></div>
                    
                    <hr style="border-color:#7d510f; margin:10px 0;">
                    
                    <h4 style="margin-top:0; color:#603000;">⚙️ Ajustes Adicionales</h4>
                    <div style="font-size:12px;">
                        <b>Detener envíos si la Granja llega al Nivel:</b><br>
                        <input type="number" id="set_farm" value="${settings.maxFarm}" style="width:60px; padding:3px; margin-top:3px;">
                        <br><i style="font-size:10px; color:#555;">(Si un pueblo llega a este nivel de granja, el script lo ignora y ya no le pide recursos).</i>
                    </div>
                </div>

                <!-- COLUMNA 2: Asignaciones -->
                <div style="flex:1.2; background:#e3d5b3; padding:10px; border-radius:5px; border:1px solid #7d510f;">
                    <h4 style="margin-top:0; color:#603000;">2A. Asignar por Grupos</h4>
                    <select id="sel_group" style="width:48%;"></select> <select id="sel_tpl_g" class="sel_tpl_class" style="width:48%;"></select>
                    <button class="btn" id="btn_assign_g" style="width:100%; margin-top:5px;">Asignar al Grupo</button>
                    <div style="font-size:11px; max-height:60px; overflow-y:auto; margin-top:5px;" id="g_list"></div>
                    
                    <hr style="border-color:#7d510f; margin:10px 0;">
                    
                    <h4 style="margin-top:0; color:#603000;">2B. Pueblos SIN Grupo (Coords Sueltas)</h4>
                    <textarea id="in_coords" placeholder="Pega coordenadas (Ej: 111|222 333|444)..." style="width:100%; height:30px; font-size:11px;"></textarea>
                    <select id="sel_tpl_c" class="sel_tpl_class" style="width:100%; margin-top:2px;"></select>
                    <button class="btn" id="btn_assign_c" style="width:100%; margin-top:5px;">Asignar a Coordenadas</button>
                    <div style="font-size:11px; max-height:50px; overflow-y:auto; margin-top:5px;" id="c_list"></div>
                </div>
            </div>

            <!-- BOTÓN MÁGICO -->
            <div style="text-align:center; margin-top:15px;">
                <button id="btn_calc_all" class="btn btn-default" style="font-size:16px; font-weight:bold; padding:12px; width:100%; background:linear-gradient(to bottom, #5c9900, #3d6600); color:white; border:1px solid #000; cursor:pointer;">
                    3. 🚀 CALCULAR TODOS LOS RECURSOS
                </button>
                <div id="calc_log" style="margin-top:10px; font-size:12px; text-align:left; background:#fff; padding:10px; border:1px solid #ccc; display:none; max-height:120px; overflow-y:auto;"></div>
            </div>
            
            <div style="text-align:right; margin-top:10px;">
                <button id="btn_clear" class="btn btn-cancel" style="font-size:10px;">Borrar Peticiones (Mercado)</button>
            </div>
        </div>
    `;
    $('body').append(html);

    function updateUI() {
        let tNames = Object.keys(templates);
        $('#t_list').text(tNames.length ? tNames.join(', ') : 'Ninguna');
        
        $('.sel_tpl_class').empty();
        tNames.forEach(n => $('.sel_tpl_class').append($('<option>', {value: n, text: n})));

        let selGrp = $('#sel_group').empty();
        for(let g in groups) selGrp.append($('<option>', {value: g, text: groups[g]}));

        let gHtml = '';
        for(let g in groupAssigns) gHtml += `• ${groups[g]||'Grupo '+g} ➔ <i>${groupAssigns[g]}</i> <a href="#" class="del_g" data-g="${g}" style="color:red;">[x]</a><br>`;
        $('#g_list').html(gHtml);

        let cDisp = {};
        for(let c in coordAssigns) { let t = coordAssigns[c]; if(!cDisp[t]) cDisp[t]=[]; cDisp[t].push(c); }
        let cHtml = '';
        for(let t in cDisp) cHtml += `• <i>${t}</i>: ${cDisp[t].join(', ')} <a href="#" class="del_c" data-t="${t}" style="color:red;">[borrar coords]</a><br>`;
        $('#c_list').html(cHtml);
    }
    updateUI();

    // Guardar ajustes
    $('#set_farm').on('change', function() {
        settings.maxFarm = parseInt($(this).val()) || 30;
        localStorage.setItem(DB_SET, JSON.stringify(settings));
    });

    $('#btn_save_t').click(function() {
        let n = $('#t_name').val().trim(), b = $('#t_b64').val().trim();
        if(n && b) { templates[n] = b; localStorage.setItem(DB_TPL, JSON.stringify(templates)); $('#t_name').val(''); $('#t_b64').val(''); updateUI(); }
    });

    $('#btn_assign_g').click(function() {
        let g = $('#sel_group').val(), t = $('#sel_tpl_g').val();
        if(g && t) { groupAssigns[g] = t; localStorage.setItem(DB_GRP, JSON.stringify(groupAssigns)); updateUI(); }
    });

    $('#btn_assign_c').click(function() {
        let text = $('#in_coords').val(), t = $('#sel_tpl_c').val();
        let matches = text.match(/\d+\|\d+/g);
        if(matches && t) {
            matches.forEach(c => coordAssigns[c] = t);
            localStorage.setItem(DB_CRD, JSON.stringify(coordAssigns));
            $('#in_coords').val(''); updateUI();
        }
    });

    $(document).on('click', '.del_g', function(e) { e.preventDefault(); delete groupAssigns[$(this).data('g')]; localStorage.setItem(DB_GRP, JSON.stringify(groupAssigns)); updateUI(); });
    $(document).on('click', '.del_c', function(e) { 
        e.preventDefault(); let t = $(this).data('t');
        for(let c in coordAssigns) if(coordAssigns[c] === t) delete coordAssigns[c];
        localStorage.setItem(DB_CRD, JSON.stringify(coordAssigns)); updateUI();
    });

    $('#btn_clear').click(function() { localStorage.removeItem(DB_REQ); pendingReqs = {}; alert('Peticiones borradas.'); });

    function log(msg) { $('#calc_log').show().append(msg + '<br>'); $('#calc_log').scrollTop($('#calc_log')[0].scrollHeight); }

    function decodeBase64(b64) {
        const MAP = {0:"main", 1:"barracks", 2:"stable", 3:"garage", 4:"church", 5:"church_f", 6:"watchtower", 7:"snob", 8:"smith", 9:"place", 10:"statue", 11:"market", 12:"wood", 13:"stone", 14:"iron", 15:"farm", 16:"storage", 17:"hide", 18:"wall"};
        try { let bin = atob(b64), s = []; for (let i = 2; i < bin.length - 1; i += 2) { let id = bin.charCodeAt(i); if (id > 18) break; if (MAP[id]) s.push(MAP[id]); } return s; } catch(e) { return []; }
    }

    $('#btn_calc_all').click(async function() {
        $('#calc_log').empty();
        log('⏳ Descargando edificios y recursos de TODOS tus pueblos...');
        
        let xmlCost = await $.get('/interface.php?func=get_building_info');
        let buildHtml = await $.get(game_data.link_base_pure + 'overview_villages&mode=buildings&group=0&page=-1');
        let prodHtml = await $.get(game_data.link_base_pure + 'overview_villages&mode=prod&group=0&page=-1');

        let allBuildings = {}, colMap = {};
        $(buildHtml).find('#buildings_table th').each(function(i) {
            let img = $(this).find('img').attr('src');
            if(img) { let m = img.match(/(?:mid-|buildings\/)([a-z_]+)\.png/); if(m) colMap[i] = m[1]; }
        });
        $(buildHtml).find('#buildings_table tr').each(function() {
            let coord = $(this).text().match(/\d+\|\d+/);
            if(coord) {
                allBuildings[coord[0]] = {};
                $(this).find('td').each(function(i) { if(colMap[i]) allBuildings[coord[0]][colMap[i]] = parseInt($(this).text())||0; });
            }
        });

        let allResources = {};
        $(prodHtml).find('#production_table tr').each(function() {
            let coord = $(this).text().match(/\d+\|\d+/);
            if(coord) {
                let row = $(this);
                allResources[coord[0]] = {
                    w: parseInt(row.find('.res.wood').text().replace(/\./g, ''))||0,
                    s: parseInt(row.find('.res.stone').text().replace(/\./g, ''))||0,
                    i: parseInt(row.find('.res.iron').text().replace(/\./g, ''))||0,
                    cap: parseInt(row.find('td:nth-last-child(2)').text().replace(/\./g, ''))||400000
                };
            }
        });

        log('⏳ Extrayendo lista de pueblos de tus grupos asignados...');
        let finalAssignments = {}; // coord -> template

        // 1. Asignar por Grupos
        for(let gId in groupAssigns) {
            let gHtml = await $.get(game_data.link_base_pure + `overview_villages&group=${gId}&page=-1`);
            $(gHtml).find('.quickedit-label').each(function() {
                let m = $(this).text().match(/\d+\|\d+/);
                if(m) finalAssignments[m[0]] = groupAssigns[gId];
            });
        }
        // 2. Asignar por Coordenadas Sueltas (Sobrescribe a los grupos si coincide)
        for(let c in coordAssigns) finalAssignments[c] = coordAssigns[c];

        log('⚙️ Calculando matemáticas...');
        let newReqs = {}, count = 0, limitFarm = settings.maxFarm;

        for (let coord in finalAssignments) {
            if (!allBuildings[coord] || !allResources[coord]) continue;
            
            // LÍMITE DE GRANJA (Feature Nueva)
            if (allBuildings[coord]['farm'] >= limitFarm) continue;

            let seq = decodeBase64(templates[finalAssignments[coord]]);
            if (!seq || seq.length === 0) continue;

            let simLvls = Object.assign({}, allBuildings[coord]);
            let nextToBuild = [];
            
            for (let b of seq) {
                let target = (simLvls[b] || 0) + 1;
                simLvls[b] = target;
                if (target > (allBuildings[coord][b] || 0)) {
                    nextToBuild.push({b: b, t: target});
                    if (nextToBuild.length === 4) break; 
                }
            }

            let reqW = 0, reqS = 0, reqI = 0;
            nextToBuild.forEach(item => {
                let node = $(xmlCost).find(item.b);
                if (node.length) {
                    reqW += Math.round(node.find('wood').text() * Math.pow(node.find('wood_factor').text(), item.t - 1));
                    reqS += Math.round(node.find('stone').text() * Math.pow(node.find('stone_factor').text(), item.t - 1));
                    reqI += Math.round(node.find('iron').text() * Math.pow(node.find('iron_factor').text(), item.t - 1));
                }
            });

            let difW = Math.max(0, reqW - allResources[coord].w);
            let difS = Math.max(0, reqS - allResources[coord].s);
            let difI = Math.max(0, reqI - allResources[coord].i);
            
            let spaceW = Math.max(0, allResources[coord].cap - allResources[coord].w);
            let spaceS = Math.max(0, allResources[coord].cap - allResources[coord].s);
            let spaceI = Math.max(0, allResources[coord].cap - allResources[coord].i);

            difW = Math.min(difW, spaceW); difS = Math.min(difS, spaceS); difI = Math.min(difI, spaceI);

            if (difW > 0 || difS > 0 || difI > 0) {
                newReqs[coord] = { w: difW, s: difS, i: difI };
                count++;
            }
        }

        localStorage.setItem(DB_REQ, JSON.stringify(newReqs));
        log(`<br><span style="color:green; font-size:14px;">✅ <b>¡COMPLETADO!</b> ${count} pueblos necesitan recursos.</span>`);
        log(`<b>Siguiente paso:</b> Cierra esto, ve al <b>Mercado</b> de un pueblo y mantén pulsada la tecla <b>ENTER</b>.`);
    });
})();
