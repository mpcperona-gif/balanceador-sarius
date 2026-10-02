/* Gestor Masivo de Edificios y Envíos (Estilo WH Balancer) */
(async function() {
    const LS_TEMPLATES = 'tw_am_mass_templates';
    const LS_ASSIGN = 'tw_am_mass_assign';
    const LS_REQS = 'tw_am_mass_reqs';

    const B_MAP = {0:"main", 1:"barracks", 2:"stable", 3:"garage", 4:"church", 5:"church_f", 6:"watchtower", 7:"snob", 8:"smith", 9:"place", 10:"statue", 11:"market", 12:"wood", 13:"stone", 14:"iron", 15:"farm", 16:"storage", 17:"hide", 18:"wall"};

    let templates = JSON.parse(localStorage.getItem(LS_TEMPLATES) || '{}');
    let assignments = JSON.parse(localStorage.getItem(LS_ASSIGN) || '{}');
    let reqs = JSON.parse(localStorage.getItem(LS_REQS) || '{}');

    // ==========================================
    // 1. MODO MERCADO: SPAM DE "ENTER" (Envío rápido)
    // ==========================================
    if (game_data.screen === 'market') {
        // Pantalla de Confirmación
        if (window.location.href.includes('try=confirm_send')) {
            let btn = $('#troop_confirm_submit');
            if (btn.length) {
                btn.focus();
                if(window.UI) UI.SuccessMessage('¡Pulsa ENTER para confirmar el envío!');
                return;
            }
        }
        
        // Pantalla de Preparar Envío
        if (!game_data.mode || game_data.mode === 'send') {
            let targetCoord = null, targetNeeds = null;

            // Buscar el primer pueblo de la lista que necesite recursos
            for (let coord in reqs) {
                if (coord === game_data.village.coord) continue; // No enviarse a sí mismo
                if (reqs[coord].w > 0 || reqs[coord].s > 0 || reqs[coord].i > 0) {
                    targetCoord = coord; targetNeeds = reqs[coord]; break;
                }
            }

            if (!targetCoord) {
                if(window.UI) UI.SuccessMessage('✅ ¡Todos los pueblos están listos! No hay peticiones pendientes.');
                return;
            }

            $('.target-input-field').val(targetCoord);
            
            let merchants = parseInt($('#market_merchant_available_count').text(), 10) || 0;
            let maxCapacity = merchants * 1000;
            
            let mw = Math.min(targetNeeds.w, game_data.village.wood);
            let ms = Math.min(targetNeeds.s, game_data.village.stone);
            let mi = Math.min(targetNeeds.i, game_data.village.iron);

            // Ajustar si no tenemos suficientes mercaderes
            if (mw + ms + mi > maxCapacity) {
                let cap = maxCapacity;
                mw = Math.min(mw, cap); cap -= mw;
                ms = Math.min(ms, cap); cap -= ms;
                mi = Math.min(mi, cap); cap -= mi;
            }

            if (mw + ms + mi === 0) {
                if(window.UI) UI.ErrorMessage('No tienes recursos/mercaderes suficientes para enviar desde aquí.');
                return;
            }

            $('input[name="wood"]').val(mw);
            $('input[name="stone"]').val(ms);
            $('input[name="iron"]').val(mi);

            // Descontar para el siguiente click
            targetNeeds.w -= mw; targetNeeds.s -= ms; targetNeeds.i -= mi;
            if (targetNeeds.w <= 0 && targetNeeds.s <= 0 && targetNeeds.i <= 0) delete reqs[targetCoord];
            else reqs[targetCoord] = targetNeeds;
            localStorage.setItem(LS_REQS, JSON.stringify(reqs));

            $('input[type="submit"]').focus();
            if(window.UI) UI.SuccessMessage(`Calculado para ${targetCoord}... ¡Pulsa ENTER!`);
            return;
        }
    }

    // ==========================================
    // 2. MODO GESTOR MASIVO (Panel de Configuración)
    // ==========================================
    if ($('#mass_am_modal').length > 0) return;
    
    let html = `
        <div id="mass_am_modal" style="position:fixed; top:5vh; left:50%; transform:translateX(-50%); width:600px; max-height:90vh; background:#e3d5b3; border:3px solid #7d510f; border-radius:8px; z-index:99999; padding:20px; overflow-y:auto; box-shadow: 0px 5px 15px rgba(0,0,0,0.8);">
            <h2 style="margin-top:0; border-bottom:1px solid #7d510f; padding-bottom:5px;">
                Gestor Masivo AM (Estilo WH Balancer)
                <a href="#" onclick="$('#mass_am_modal').remove(); return false;" style="float:right; font-weight:bold; color:#c00; text-decoration:none;">✖ CERRAR</a>
            </h2>
            
            <!-- PLANTILLAS -->
            <table class="vis" style="width:100%; margin-bottom:15px;">
                <tr><th colspan="2">1. Guardar Plantillas de Construcción</th></tr>
                <tr>
                    <td style="padding:10px;">
                        <input type="text" id="tpl_name" placeholder="Nombre (ej. Ofensiva)" style="width:25%;">
                        <input type="text" id="tpl_b64" placeholder="Pega el Base64 aquí..." style="width:50%;">
                        <button id="btn_add_tpl" class="btn">Guardar</button>
                    </td>
                </tr>
                <tr><td id="list_tpls" style="font-size:11px; padding:5px; color:#555;"></td></tr>
            </table>

            <!-- ASIGNACIÓN MASIVA -->
            <table class="vis" style="width:100%; margin-bottom:15px;">
                <tr><th colspan="2">2. Asignar Plantilla a Varios Pueblos (Pega Coordenadas)</th></tr>
                <tr>
                    <td width="35%" style="padding:10px; vertical-align:top;">
                        <select id="mass_tpl_select" style="width:100%; margin-bottom:10px;"></select>
                        <button id="btn_assign" class="btn" style="width:100%;">Asignar a Coordenadas ➔</button>
                    </td>
                    <td style="padding:10px;">
                        <textarea id="mass_coords" style="width:100%; height:80px;" placeholder="Pega una lista de coordenadas aquí. Ej: 555|666 777|888 ..."></textarea>
                    </td>
                </tr>
                <tr><td colspan="2" style="font-size:11px; padding:5px;"><i>Pueblos asignados en memoria: <b id="count_assigns" style="color:green;">0</b></i></td></tr>
            </table>

            <!-- CÁLCULO MÁGICO -->
            <div style="text-align:center; padding-top:10px;">
                <button id="btn_calculate_all" class="btn btn-default" style="font-size:16px; font-weight:bold; padding:12px; width:100%;">3. ¡Calcular Necesidades de Todos los Pueblos!</button>
                <p style="font-size:12px; margin-top:8px; color:#333;"><i>Al pulsar aquí, el script leerá el nivel de tus edificios y recursos de todos tus pueblos en segundo plano. Luego, ve al Mercado y usa el script con ENTER para enviar.</i></p>
                <div id="calc_status" style="margin-top:10px; font-weight:bold; color:blue;"></div>
            </div>
            
            <!-- BOTON RESET -->
            <div style="margin-top:20px; text-align:right;">
                <button id="btn_clear_reqs" class="btn btn-cancel" style="font-size:10px;">Borrar peticiones pendientes del mercado</button>
            </div>
        </div>
    `;
    $('body').append(html);

    function updateUI() {
        let tplNames = Object.keys(templates);
        $('#list_tpls').text(tplNames.length > 0 ? "Plantillas guardadas: " + tplNames.join(', ') : "Ninguna plantilla guardada.");
        let sel = $('#mass_tpl_select').empty();
        tplNames.forEach(n => sel.append($('<option>', {value: n, text: n})));
        $('#count_assigns').text(Object.keys(assignments).length);
    }
    updateUI();

    $('#btn_add_tpl').click(function() {
        let n = $('#tpl_name').val().trim(), b = $('#tpl_b64').val().trim();
        if(n && b) { templates[n] = b; localStorage.setItem(LS_TEMPLATES, JSON.stringify(templates)); $('#tpl_name').val(''); $('#tpl_b64').val(''); updateUI(); }
    });

    $('#btn_assign').click(function() {
        let text = $('#mass_coords').val(), tpl = $('#mass_tpl_select').val();
        let matches = text.match(/\d+\|\d+/g);
        if(matches && tpl) {
            matches.forEach(c => assignments[c] = tpl);
            localStorage.setItem(LS_ASSIGN, JSON.stringify(assignments));
            updateUI(); alert(`¡${matches.length} pueblos asignados a ${tpl}!`); $('#mass_coords').val('');
        }
    });

    $('#btn_clear_reqs').click(function() {
        localStorage.removeItem(LS_REQS); reqs = {}; alert('Lista de envíos borrada.');
    });

    function decodeBase64(b64) {
        try { 
            let bin = atob(b64), s = [], i = 2; 
            while (i < bin.length - 1) { 
                let b1 = bin.charCodeAt(i), b2 = bin.charCodeAt(i+1); i += 2; 
                if (b1 > 18) break; // Fin de edificios (empieza el texto del nombre)
                if (b2 === 1) s.push(B_MAP[b1]); // Solo procesar subidas de nivel
            } 
            return s; 
        } catch(e) { return []; }
    }

    $('#btn_calculate_all').click(async function() {
        $('#calc_status').text('⏳ Descargando niveles de edificios de todos tus pueblos...');
        let bHtml = await $.get('/game.php?screen=overview_villages&mode=buildings&page=-1');
        
        $('#calc_status').text('⏳ Descargando recursos actuales...');
        let pHtml = await $.get('/game.php?screen=overview_villages&mode=prod&page=-1');
        
        $('#calc_status').text('⏳ Descargando fórmula de costes...');
        let xml = await $.get('/interface.php?func=get_building_info');
        
        // Extraer edificios
        let buildings = {}, bMap = [];
        $(bHtml).find('#buildings_table th').each(function(i) {
            let img = $(this).find('img').attr('src');
            if(img) { let m = img.match(/buildings\/(mid-)?([a-z_]+)\.png/); if(m) bMap[i] = m[2]; }
        });
        
        $(bHtml).find('#buildings_table tr').each(function() {
            let m = $(this).text().match(/\d+\|\d+/);
            if(m) {
                buildings[m[0]] = {};
                $(this).find('td').each(function(i) {
                    if(bMap[i]) buildings[m[0]][bMap[i]] = parseInt($(this).text()) || 0;
                });
            }
        });

        // Extraer recursos
        let resources = {};
        $(pHtml).find('#production_table tr').each(function() {
            let m = $(this).text().match(/\d+\|\d+/);
            if(m) {
                resources[m[0]] = {
                    w: parseInt($(this).find('.res.wood').text().replace(/\./g, '')) || 0,
                    s: parseInt($(this).find('.res.stone').text().replace(/\./g, '')) || 0,
                    i: parseInt($(this).find('.res.iron').text().replace(/\./g, '')) || 0
                };
            }
        });

        $('#calc_status').text('⏳ Calculando matemáticas de las plantillas...');
        let newReqs = {};

        // Calcular déficits para los próximos 5 niveles
        for (let coord in assignments) {
            let tplBase64 = templates[assignments[coord]];
            if (!tplBase64 || !buildings[coord] || !resources[coord]) continue;

            let seq = decodeBase64(tplBase64), simLvls = Object.assign({}, buildings[coord]), next5 = [];
            for (let b of seq) {
                if(!b) continue;
                let target = (simLvls[b] || 0) + 1; simLvls[b] = target;
                if (target > (buildings[coord][b] || 0)) {
                    next5.push({ b: b, target: target });
                    if (next5.length === 5) break;
                }
            }

            if (next5.length === 0) continue;

            let wTot = 0, sTot = 0, iTot = 0;
            next5.forEach(item => {
                let node = $(xml).find(item.b);
                if(node.length) {
                    wTot += Math.round(node.find('wood').text() * Math.pow(node.find('wood_factor').text(), item.target - 1));
                    sTot += Math.round(node.find('stone').text() * Math.pow(node.find('stone_factor').text(), item.target - 1));
                    iTot += Math.round(node.find('iron').text() * Math.pow(node.find('iron_factor').text(), item.target - 1));
                }
            });

            let reqW = Math.max(0, wTot - resources[coord].w);
            let reqS = Math.max(0, sTot - resources[coord].s);
            let reqI = Math.max(0, iTot - resources[coord].i);

            if (reqW > 0 || reqS > 0 || reqI > 0) newReqs[coord] = { w: reqW, s: reqS, i: reqI };
        }

        localStorage.setItem(LS_REQS, JSON.stringify(newReqs));
        let count = Object.keys(newReqs).length;
        $('#calc_status').html(`<span style="color:green; font-size:15px;">✅ ¡Éxito! <b>${count} pueblos</b> necesitan recursos.<br>Cierra esta ventana, ve al MERCADO y pulsa el script para mandar con ENTER.</span>`);
    });
})();
