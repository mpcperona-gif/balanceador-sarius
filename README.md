/* Gestor Avanzado AM + Envíos Rápidos al Mercado (5 Edificios) */
(function() {
    const B_MAP = {0:"main", 1:"barracks", 2:"stable", 3:"garage", 4:"church", 5:"church_f", 6:"watchtower", 7:"snob", 8:"smith", 9:"place", 10:"statue", 11:"market", 12:"wood", 13:"stone", 14:"iron", 15:"farm", 16:"storage", 17:"hide", 18:"wall"};
    const B_ES = {"main":"Edificio Principal", "barracks":"Cuartel", "stable":"Cuadra", "garage":"Taller", "church":"Iglesia", "church_f":"Primera Iglesia", "watchtower":"Torre", "snob":"Corte", "smith":"Herrería", "place":"Plaza", "statue":"Estatua", "market":"Mercado", "wood":"Leñador", "stone":"Barrera", "iron":"Mina", "farm":"Granja", "storage":"Almacén", "hide":"Escondite", "wall":"Muralla"};
    
    let templates = JSON.parse(localStorage.getItem('tw_am_templates') || '{}');
    let assignments = JSON.parse(localStorage.getItem('tw_am_assignments') || '{}');
    let requests = JSON.parse(localStorage.getItem('tw_am_reqs') || '{}');

    // ==========================================
    // 1. MODO MERCADO: ENVÍO RÁPIDO CON "ENTER"
    // ==========================================
    if (game_data.screen === 'market') {
        // Pantalla 2: Confirmar el envío
        if (window.location.href.includes('try=confirm_send')) {
            let btn = $('#troop_confirm_submit');
            if (btn.length) {
                btn.focus();
                if(window.UI) UI.SuccessMessage('Pulsa ENTER para confirmar el envío.');
                return; // Cortamos aquí para que el usuario pulse Enter
            }
        }
        
        // Pantalla 1: Preparar el envío
        if (!game_data.mode || game_data.mode === 'send') {
            let target = null, targetId = null;
            // Buscar la primera petición que necesite recursos
            for (let id in requests) {
                if (requests[id].w > 0 || requests[id].s > 0 || requests[id].i > 0) {
                    target = requests[id]; targetId = id; break;
                }
            }

            if (target) {
                $('.target-input-field').val(target.c);
                
                let maxM = game_data.village.trader_amount * 1000;
                let w_send = Math.min(target.w, game_data.village.wood);
                let s_send = Math.min(target.s, game_data.village.stone);
                let i_send = Math.min(target.i, game_data.village.iron);

                // Ajustar si no tenemos suficientes mercaderes
                let total = w_send + s_send + i_send;
                if (total > maxM) {
                    let rem = maxM;
                    w_send = Math.min(w_send, rem); rem -= w_send;
                    s_send = Math.min(s_send, rem); rem -= s_send;
                    i_send = Math.min(i_send, rem); rem -= i_send;
                }

                if(total === 0) {
                    if(window.UI) UI.ErrorMessage('No tienes recursos suficientes en este pueblo para enviar.');
                } else {
                    $('input[name="wood"]').val(w_send);
                    $('input[name="stone"]').val(s_send);
                    $('input[name="iron"]').val(i_send);

                    // Descontar de la lista para el próximo clic
                    target.w -= w_send; target.s -= s_send; target.i -= i_send;
                    if(target.w <= 0 && target.s <= 0 && target.i <= 0) delete requests[targetId];
                    else requests[targetId] = target;
                    localStorage.setItem('tw_am_reqs', JSON.stringify(requests));

                    $('input[type="submit"]').focus();
                    if(window.UI) UI.SuccessMessage(`Enviando a ${target.c}... ¡Pulsa ENTER!`);
                }
            } else {
                if(window.UI) UI.SuccessMessage('¡No hay ningún pueblo que necesite recursos!');
            }
        }
    }

    // ==========================================
    // 2. MODO GESTOR: VISION GENERAL (Receptor)
    // ==========================================
    if ($('#am_advanced_panel').length > 0) return;
    
    let pendingCount = Object.keys(requests).length;
    let html = `
        <div id="am_advanced_panel" class="vis" style="position:fixed; top:50px; right:10px; z-index:99999; padding:15px; width:350px; background: #e3d5b3; border: 2px solid #7d510f; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0,0,0,0.5);">
            <h3 style="margin-top:0; border-bottom: 1px solid #7d510f; padding-bottom:5px;">
                Gestor (5 Edificios)
                <span style="float:right; cursor:pointer; color:red;" onclick="$('#am_advanced_panel').remove();">✖</span>
            </h3>
            
            <div style="margin-bottom: 10px;">
                <b>Plantillas:</b>
                <input type="text" id="am_t_name" placeholder="Nombre" style="width:40%;">
                <input type="text" id="am_t_b64" placeholder="Base64..." style="width:55%;">
                <button class="btn" id="btn_save_template" style="width:100%; margin-top:2px;">Guardar / Actualizar</button>
            </div>

            <div style="margin-bottom: 10px;">
                <b>Asignar a este pueblo:</b><br>
                <select id="am_t_select" style="width:70%;"></select>
                <button class="btn" id="btn_assign" style="width:25%;">Poner</button>
            </div>

            <hr style="border-color:#7d510f">
            <button class="btn btn-default" id="btn_calc_all" style="width:100%; font-size:14px; font-weight:bold; padding:8px;">Calcular y Solicitar Recursos</button>
            
            <div id="am_results" style="margin-top:10px; font-size:12px;"></div>
            
            <hr style="border-color:#7d510f">
            <div style="font-size:11px; text-align:center;">
                <b>Pueblos esperando recursos: <span style="color:red; font-size:13px">${pendingCount}</span></b><br>
                <button class="btn btn-cancel" id="btn_clear_reqs" style="margin-top:5px; font-size:10px;">Borrar todas las peticiones</button>
            </div>
        </div>
    `;
    $('body').append(html);

    function updateUI() {
        let sel = $('#am_t_select');
        sel.empty();
        for (let n in templates) sel.append($('<option>', { value: n, text: n }));
        if(assignments[game_data.village.id]) sel.val(assignments[game_data.village.id]);
    }
    updateUI();

    $('#btn_save_template').click(function() {
        let n = $('#am_t_name').val().trim(), b = $('#am_t_b64').val().trim();
        if (n && b) { templates[n] = b; localStorage.setItem('tw_am_templates', JSON.stringify(templates)); updateUI(); $('#am_t_name').val(''); $('#am_t_b64').val(''); }
    });

    $('#btn_assign').click(function() {
        let tName = $('#am_t_select').val();
        if (tName) { assignments[game_data.village.id] = tName; localStorage.setItem('tw_am_assignments', JSON.stringify(assignments)); alert("Plantilla asignada."); }
    });

    $('#btn_clear_reqs').click(function() {
        localStorage.removeItem('tw_am_reqs'); requests = {}; alert("Lista de envíos vaciada."); $('#am_advanced_panel').remove();
    });

    function decodeBase64(b64) {
        try { let bin = atob(b64), s = [], i = 2; while (i < bin.length - 1) { let b1 = bin.charCodeAt(i), b2 = bin.charCodeAt(i+1); i += 2; if (b1 > 18) break; if (b2 === 1) s.push(B_MAP[b1]); } return s; } catch(e) { return []; }
    }

    $('#btn_calc_all').click(function() {
        $('#am_results').html('<i>Calculando los próximos 5 edificios...</i>');
        $.ajax({
            url: '/interface.php?func=get_building_info', dataType: 'xml',
            success: function(xml) {
                let vId = game_data.village.id;
                let assigned = assignments[vId];
                if (!assigned || !templates[assigned]) { $('#am_results').html('<b style="color:red">Asigna una plantilla primero.</b>'); return; }

                let seq = decodeBase64(templates[assigned]);
                let curLvls = game_data.village.buildings;
                let simLvls = Object.assign({}, curLvls);
                let next5 = [];

                for (let b of seq) {
                    if (!b) continue;
                    let curr = parseInt(curLvls[b] || 0);
                    let sim = parseInt(simLvls[b] || 0) + 1;
                    simLvls[b] = sim;
                    if (sim > curr) {
                        next5.push({ b: b, target: sim });
                        if (next5.length === 5) break;
                    }
                }

                if (next5.length === 0) { $('#am_results').html('<b style="color:green">Pueblo completado.</b>'); return; }

                let wTot = 0, sTot = 0, iTot = 0;
                let listHtml = "<b>Próximos 5:</b><br>";
                
                next5.forEach(item => {
                    let node = $(xml).find(item.b);
                    if(node.length) {
                        let w = Math.round(node.find('wood').text() * Math.pow(node.find('wood_factor').text(), item.target - 1));
                        let s = Math.round(node.find('stone').text() * Math.pow(node.find('stone_factor').text(), item.target - 1));
                        let i = Math.round(node.find('iron').text() * Math.pow(node.find('iron_factor').text(), item.target - 1));
                        wTot += w; sTot += s; iTot += i;
                        listHtml += `- ${B_ES[item.b] || item.b} (Nvl ${item.target})<br>`;
                    }
                });

                // LIMITE DE ALMACEN: No pedir más de lo que cabe restando lo que ya hay
                let cap = parseInt(game_data.village.storage_max);
                let cW = game_data.village.wood, cS = game_data.village.stone, cI = game_data.village.iron;
                
                let reqW = Math.max(0, Math.min(wTot - cW, cap - cW));
                let reqS = Math.max(0, Math.min(sTot - cS, cap - cS));
                let reqI = Math.max(0, Math.min(iTot - cI, cap - cI));

                listHtml += `<br><b>Petición guardada:</b><br>Madera: ${reqW}<br>Barro: ${reqS}<br>Hierro: ${reqI}`;

                if (reqW > 0 || reqS > 0 || reqI > 0) {
                    requests[vId] = { w: reqW, s: reqS, i: reqI, c: game_data.village.coord };
                    localStorage.setItem('tw_am_reqs', JSON.stringify(requests));
                    listHtml += `<br><br><b style="color:green;">¡Pueblo añadido a la lista de envíos! Ve al mercado de otro pueblo.</b>`;
                } else {
                    listHtml += `<br><br><b style="color:green;">Tienes recursos suficientes para los 5.</b>`;
                }

                $('#am_results').html(listHtml);
            }
        });
    });
})();
