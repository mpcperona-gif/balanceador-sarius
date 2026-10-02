/* 
 * TRIBAL WARS: MEGA GESTOR AM + WH BALANCER LOGIC
 * Recreado exactamente con lógica de Población y Almacén Máximo
 */
(async function() {
    const DB_TPL = 'tw_mega_templates_v3';
    const DB_CRD = 'tw_mega_coords_v3';
    const DB_REQ = 'tw_mega_requests_v3';
    const DB_SET = 'tw_mega_settings_v3';

    let templates = JSON.parse(localStorage.getItem(DB_TPL) || '{}');
    let coordAssigns = JSON.parse(localStorage.getItem(DB_CRD) || '{}');
    let pendingReqs = JSON.parse(localStorage.getItem(DB_REQ) || '{}');
    let settings = JSON.parse(localStorage.getItem(DB_SET) || '{"highFarm": 23000}');

    // ==========================================
    // 1. MODO MERCADO: ENVÍO INTELIGENTE Y PROPORCIONAL
    // ==========================================
    if (game_data.screen === 'market') {
        if (window.location.href.includes('try=confirm_send')) {
            let btn = $('#troop_confirm_submit');
            if (btn.length) { btn.focus(); if(window.UI) UI.SuccessMessage('¡Pulsa ENTER para Confirmar!'); return; }
        }

        if (!game_data.mode || game_data.mode === 'send') {
            let merchants = parseInt($('#market_merchant_available_count').text(), 10) || 0;
            if (merchants === 0) { if(window.UI) UI.ErrorMessage('No tienes mercaderes libres.'); return; }

            let targetCoord = null, targetNeeds = null;
            let vw = game_data.village.wood, vs = game_data.village.stone, vi = game_data.village.iron;
            let mw = 0, ms = 0, mi = 0;

            // Buscar inteligentemente a quién podemos ayudar con los recursos actuales
            for (let coord in pendingReqs) {
                if (coord === game_data.village.coord) continue;
                let req = pendingReqs[coord];
                
                let canSendW = Math.min(req.w, vw);
                let canSendS = Math.min(req.s, vs);
                let canSendI = Math.min(req.i, vi);
                
                // Solo enviamos si podemos aportar más de 200 recursos (evita envíos basura)
                if (canSendW + canSendS + canSendI > 200) {
                    targetCoord = coord; targetNeeds = req;
                    mw = canSendW; ms = canSendS; mi = canSendI;
                    break;
                }
            }

            if (!targetCoord) { if(window.UI) UI.SuccessMessage('✅ ¡NO HAY PUEBLOS QUE NECESITEN LO QUE TIENES AQUÍ!'); return; }

            $('.target-input-field').val(targetCoord);

            // Reparto proporcional exacto si nos pasamos de capacidad
            let maxCap = merchants * 1000;
            let totalSend = mw + ms + mi;

            if (totalSend > maxCap) {
                let ratio = maxCap / totalSend;
                mw = Math.floor(mw * ratio);
                ms = Math.floor(ms * ratio);
                mi = Math.floor(mi * ratio);
            }

            $('input[name="wood"]').val(mw); 
            $('input[name="stone"]').val(ms); 
            $('input[name="iron"]').val(mi);

            // Restamos lo enviado de la memoria
            targetNeeds.w -= mw; targetNeeds.s -= ms; targetNeeds.i -= mi;
            if (targetNeeds.w <= 0 && targetNeeds.s <= 0 && targetNeeds.i <= 0) delete pendingReqs[targetCoord];
            else pendingReqs[targetCoord] = targetNeeds;
            localStorage.setItem(DB_REQ, JSON.stringify(pendingReqs));

            $('input[type="submit"]').focus();
            if(window.UI) UI.SuccessMessage(`Calculado para ${targetCoord}... ¡Pulsa ENTER!`);
            return;
        }
    }

    // ==========================================
    // 2. MODO GESTOR CENTRAL
    // ==========================================
    if ($('#mega_am_panel').length > 0) return;

    let html = `
        <div id="mega_am_panel" style="position:fixed; top:5vh; left:50%; transform:translateX(-50%); width:800px; max-height:90vh; background:#f4e4bc; border:3px solid #603000; border-radius:10px; z-index:99999; padding:20px; overflow-y:auto; box-shadow: 0px 10px 25px rgba(0,0,0,0.8); color:#333;">
            <h2 style="margin-top:0; border-bottom:2px solid #603000; padding-bottom:5px; text-align:center;">
                👑 Mega Gestor WH Balancer + Plantillas
                <a href="#" onclick="$('#mega_am_panel').remove(); return false;" style="float:right; font-size:16px; color:#c00; text-decoration:none;">✖</a>
            </h2>
            
            <div style="display:flex; gap:15px;">
                <!-- COLUMNA IZQUIERDA: Configuración -->
                <div style="flex:1; background:#e3d5b3; padding:10px; border-radius:5px; border:1px solid #7d510f;">
                    <h4 style="margin-top:0; color:#603000;">1. Crear Plantillas</h4>
                    <input type="text" id="t_name" placeholder="Nombre (Ej: Ofensiva)" style="width:100%; margin-bottom:5px;">
                    <textarea id="t_b64" placeholder="Código Base64..." style="width:100%; height:40px; margin-bottom:5px; font-size:10px;"></textarea>
                    <button class="btn" id="btn_save_t" style="width:100%;">Guardar Plantilla</button>
                    <div style="font-size:11px; margin-top:5px; max-height:40px; overflow-y:auto;"><b>Guardadas:</b> <span id="t_list" style="color:blue;"></span></div>
                    
                    <hr style="border-color:#7d510f; margin:10px 0;">
                    
                    <h4 style="margin-top:0; color:#603000;">⚙️ Límite de Población</h4>
                    <div style="font-size:12px;">
                        <b>Población Máxima (highFarm):</b><br>
                        <input type="number" id="set_pop" value="${settings.highFarm}" style="width:80px; padding:3px; margin-top:3px;">
                        <br><i style="font-size:10px; color:#555;">(Pueblos que alcancen esta población exacta, se convierten en DONANTES y vacían sus recursos hacia los demás).</i>
                    </div>
                </div>

                <!-- COLUMNA DERECHA: Asignación Permanente -->
                <div style="flex:1.2; background:#e3d5b3; padding:10px; border-radius:5px; border:1px solid #7d510f;">
                    <h4 style="margin-top:0; color:#603000;">2. Asignar Pueblos a Plantilla</h4>
                    <textarea id="in_coords" placeholder="Pega una lista de coordenadas (Ej: 111|222 333|444)..." style="width:100%; height:40px; font-size:11px;"></textarea>
                    <select id="sel_tpl_c" style="width:100%; margin-top:5px;"></select>
                    <button class="btn" id="btn_assign_c" style="width:100%; margin-top:5px;">Vincular Coordenadas</button>
                    
                    <div style="font-size:11px; margin-top:10px;"><b>Pueblos Vinculados Permanentemente:</b></div>
                    <div style="font-size:11px; max-height:100px; overflow-y:auto; background:#fff; padding:5px; border:1px solid #ccc;" id="c_list"></div>
                    <button id="btn_clear_c" class="btn btn-cancel" style="font-size:9px; margin-top:5px;">Borrar Todos los Vínculos</button>
                </div>
            </div>

            <!-- BOTÓN MÁGICO DE CÁLCULO -->
            <div style="text-align:center; margin-top:20px;">
                <button id="btn_calc_all" class="btn btn-default" style="font-size:16px; font-weight:bold; padding:15px; width:100%; background:linear-gradient(to bottom, #5c9900, #3d6600); color:white; border:1px solid #000; cursor:pointer;">
                    3. 🚀 CALCULAR CONSTRUCCIONES Y RECURSOS MÁXIMOS
                </button>
                <div id="calc_log" style="margin-top:15px; font-size:13px; text-align:left; background:#fff; padding:10px; border:1px solid #ccc; display:none; max-height:150px; overflow-y:auto; font-family:monospace;"></div>
            </div>
            
            <div style="text-align:right; margin-top:10px;">
                <button id="btn_clear_reqs" class="btn btn-cancel" style="font-size:10px;">Borrar Peticiones del Mercado</button>
            </div>
        </div>
    `;
    $('body').append(html);

    function updateUI() {
        let tNames = Object.keys(templates);
        $('#t_list').text(tNames.length ? tNames.join(', ') : 'Ninguna');
        
        let sel = $('#sel_tpl_c').empty();
        tNames.forEach(n => sel.append($('<option>', {value: n, text: n})));

        let cDisp = {}; 
        for(let c in coordAssigns) { let t = coordAssigns[c]; if(!cDisp[t]) cDisp[t]=[]; cDisp[t].push(c); }
        let cHtml = ''; 
        for(let t in cDisp) cHtml += `<b>[${t}]</b>: ${cDisp[t].join(', ')}<br>`;
        $('#c_list').html(cHtml || '<i>No hay pueblos vinculados.</i>');
    }
    updateUI();

    $('#set_pop').on('change', function() { settings.highFarm = parseInt($(this).val()) || 23000; localStorage.setItem(DB_SET, JSON.stringify(settings)); });
    $('#btn_save_t').click(function() { let n = $('#t_name').val().trim(), b = $('#t_b64').val().trim(); if(n && b) { templates[n] = b; localStorage.setItem(DB_TPL, JSON.stringify(templates)); $('#t_name').val(''); $('#t_b64').val(''); updateUI(); } });
    $('#btn_assign_c').click(function() { let text = $('#in_coords').val(), t = $('#sel_tpl_c').val(); let matches = text.match(/\d+\|\d+/g); if(matches && t) { matches.forEach(c => coordAssigns[c] = t); localStorage.setItem(DB_CRD, JSON.stringify(coordAssigns)); $('#in_coords').val(''); updateUI(); } });
    $('#btn_clear_c').click(function() { coordAssigns = {}; localStorage.setItem(DB_CRD, '{}'); updateUI(); });
    $('#btn_clear_reqs').click(function() { pendingReqs = {}; localStorage.setItem(DB_REQ, '{}'); alert('Mercado reiniciado.'); });

    function log(msg) { $('#calc_log').show().append(msg + '<br>'); $('#calc_log').scrollTop($('#calc_log')[0].scrollHeight); }

    function decodeBase64(b64) {
        const MAP = {0:"main", 1:"barracks", 2:"stable", 3:"garage", 4:"church", 5:"church_f", 6:"watchtower", 7:"snob", 8:"smith", 9:"place", 10:"statue", 11:"market", 12:"wood", 13:"stone", 14:"iron", 15:"farm", 16:"storage", 17:"hide", 18:"wall"};
        try { let bin = atob(b64), s = []; for (let i = 2; i < bin.length - 1; i += 2) { let id = bin.charCodeAt(i); let act = bin.charCodeAt(i+1); if (id > 18) break; if (MAP[id] && act === 1) s.push(MAP[id]); } return s; } catch(e) { return []; }
    }

    $('#btn_calc_all').click(async function() {
        $('#calc_log').empty();
        log('⏳ Descargando base de datos del servidor...');
        let xmlCost = await $.get('/interface.php?func=get_building_info');
        let buildHtml = await $.get(game_data.link_base_pure + 'overview_villages&mode=buildings&page=-1');
        let prodHtml = await $.get(game_data.link_base_pure + 'overview_villages&mode=prod&page=-1');

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

        let allData = {};
        $(prodHtml).find('#production_table tr').each(function() {
            let coord = $(this).text().match(/\d+\|\d+/);
            if(coord) {
                let rowText = $(this).text().replace(/\./g, '');
                let popMatch = rowText.match(/(\d+)\/(\d+)/); // Extrae la poblacion ej: 23000/24000
                allData[coord[0]] = {
                    w: parseInt($(this).find('.res.wood').text().replace(/\./g, ''))||0,
                    s: parseInt($(this).find('.res.stone').text().replace(/\./g, ''))||0,
                    i: parseInt($(this).find('.res.iron').text().replace(/\./g, ''))||0,
                    cap: parseInt($(this).find('td:nth-last-child(2)').text().replace(/\./g, ''))||400000, // Almacen Maximo
                    pop: popMatch ? parseInt(popMatch[1]) : 0
                };
            }
        });

        log('⚙️ Calculando las construcciones máximas permitidas por almacén...');
        let newReqs = {}, count = 0, limitPop = settings.highFarm;

        for (let coord in coordAssigns) {
            if (!allBuildings[coord] || !allData[coord]) continue;
            
            // Si la población es igual o superior al límite, es PUEBLO DONANTE. No pide nada.
            if (allData[coord].pop >= limitPop) continue;

            let seq = decodeBase64(templates[coordAssigns[coord]]);
            if (!seq || seq.length === 0) continue;

            let simLvls = Object.assign({}, allBuildings[coord]);
            let reqW = 0, reqS = 0, reqI = 0;
            let maxStorage = allData[coord].cap;
            
            // Simulamos toda la plantilla hasta que nos topemos con el límite del almacén
            for (let b of seq) {
                let target = (simLvls[b] || 0) + 1;
                
                if (target > (allBuildings[coord][b] || 0)) {
                    let node = $(xmlCost).find(b);
                    if (node.length) {
                        let costW = Math.round(node.find('wood').text() * Math.pow(node.find('wood_factor').text(), target - 1));
                        let costS = Math.round(node.find('stone').text() * Math.pow(node.find('stone_factor').text(), target - 1));
                        let costI = Math.round(node.find('iron').text() * Math.pow(node.find('iron_factor').text(), target - 1));
                        
                        // Si sumar este edificio supera la capacidad MÁXIMA del almacén, cortamos simulación (es imposible pedir tanto)
                        if (reqW + costW > maxStorage || reqS + costS > maxStorage || reqI + costI > maxStorage) {
                            break; 
                        }
                        
                        reqW += costW; reqS += costS; reqI += costI;
                        simLvls[b] = target; // Nivel aceptado en la simulación
                    }
                }
            }

            // Calculamos el déficit real (lo que cuesta todo eso MENOS lo que ya tiene en la aldea)
            let difW = Math.max(0, reqW - allData[coord].w);
            let difS = Math.max(0, reqS - allData[coord].s);
            let difI = Math.max(0, reqI - allData[coord].i);

            if (difW > 0 || difS > 0 || difI > 0) {
                newReqs[coord] = { w: difW, s: difS, i: difI };
                count++;
            }
        }

        localStorage.setItem(DB_REQ, JSON.stringify(newReqs));
        log(`<br><span style="color:green; font-size:14px;">✅ <b>¡COMPLETADO!</b> ${count} pueblos necesitan recursos para construir al máximo.</span>`);
        log(`<b style="color:red">Siguiente paso:</b> Cierra esto, ve al <b>Mercado</b> de un pueblo donante y mantén pulsado <b>ENTER</b>. El pueblo se vaciará enviando el máximo posible.`);
    });
})();
