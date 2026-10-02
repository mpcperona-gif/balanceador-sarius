/* Calculadora Avanzada de Plantillas AM - Multi-Pueblos */
(function() {
    if ($('#am_advanced_panel').length > 0) return;

    const BUILDINGS_MAP = {
        0:"main", 1:"barracks", 2:"stable", 3:"garage", 4:"church", 5:"church_f",
        6:"watchtower", 7:"snob", 8:"smith", 9:"place", 10:"statue", 11:"market",
        12:"wood", 13:"stone", 14:"iron", 15:"farm", 16:"storage", 17:"hide", 18:"wall"
    };

    // Cargar datos guardados
    let templates = JSON.parse(localStorage.getItem('tw_am_templates') || '{}');
    let assignments = JSON.parse(localStorage.getItem('tw_am_assignments') || '{}');
    
    let html = `
        <div id="am_advanced_panel" class="vis" style="position:fixed; top:50px; right:10px; z-index:99999; padding:15px; width:400px; background: #e3d5b3; border: 2px solid #7d510f; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0,0,0,0.5); max-height: 80vh; overflow-y: auto;">
            <h3 style="margin-top:0; border-bottom: 1px solid #7d510f; padding-bottom:5px;">
                Gestor Avanzado de Construcción
                <span style="float:right; cursor:pointer; color:red;" onclick="$('#am_advanced_panel').remove();">✖</span>
            </h3>
            
            <div style="margin-bottom: 10px;">
                <b>1. Añadir Nueva Plantilla</b><br>
                <input type="text" id="am_t_name" placeholder="Nombre (ej. Ofensiva)" style="width:100%; margin-bottom:3px;">
                <textarea id="am_t_b64" placeholder="Pega el Base64 aquí..." style="width:100%; height:40px; font-size:10px;"></textarea>
                <button class="btn" id="btn_save_template" style="width:100%;">Guardar Plantilla</button>
            </div>

            <div style="margin-bottom: 10px;">
                <b>2. Asignar Plantilla a este Pueblo</b><br>
                <select id="am_t_select" style="width:70%;"></select>
                <button class="btn" id="btn_assign" style="width:25%;">Asignar</button>
                <div style="font-size:11px; margin-top:2px;"><i>Pueblo actual asignado a: <span id="current_assigned" style="color:green; font-weight:bold;">Ninguna</span></i></div>
            </div>

            <hr style="border-color:#7d510f">
            
            <button class="btn btn-default" id="btn_calc_all" style="width:100%; font-size:14px; font-weight:bold; padding:8px;">Calcular y Mandar Recursos</button>
            
            <div id="am_results" style="margin-top:15px;"></div>
        </div>
    `;
    $('body').append(html);

    function updateUI() {
        let sel = $('#am_t_select');
        sel.empty();
        for (let name in templates) {
            sel.append($('<option>', { value: name, text: name }));
        }
        let currentVillageId = game_data.village.id;
        $('#current_assigned').text(assignments[currentVillageId] || "Ninguna");
    }
    updateUI();

    $('#btn_save_template').click(function() {
        let n = $('#am_t_name').val().trim();
        let b = $('#am_t_b64').val().trim();
        if (n && b) {
            templates[n] = b;
            localStorage.setItem('tw_am_templates', JSON.stringify(templates));
            updateUI();
            alert("Plantilla guardada.");
        }
    });

    $('#btn_assign').click(function() {
        let tName = $('#am_t_select').val();
        let currentVillageId = game_data.village.id;
        if (tName) {
            assignments[currentVillageId] = tName;
            localStorage.setItem('tw_am_assignments', JSON.stringify(assignments));
            updateUI();
            alert("Plantilla asignada a este pueblo.");
        }
    });

    function decodeBase64(b64) {
        try {
            let bin = atob(b64), seq = [], i = 2;
            while (i < bin.length - 1) {
                let b1 = bin.charCodeAt(i), b2 = bin.charCodeAt(i+1);
                i += 2;
                if (b1 > 18) break;
                if (b2 === 1) seq.push(BUILDINGS_MAP[b1]);
            }
            return seq;
        } catch(e) { return []; }
    }

    $('#btn_calc_all').click(function() {
        $('#am_results').html('<i>Descargando costes del servidor...</i>');
        
        // 1. Obtener los costes de edificios (XML)
        $.ajax({
            url: '/interface.php?func=get_building_info',
            dataType: 'xml',
            success: function(xml) {
                let vId = game_data.village.id;
                let assigned = assignments[vId];
                
                if (!assigned || !templates[assigned]) {
                    $('#am_results').html('<b style="color:red">No has asignado una plantilla válida a este pueblo.</b>');
                    return;
                }

                let seq = decodeBase64(templates[assigned]);
                let curLvls = game_data.village.buildings;
                let simLvls = {};
                let nextB = null, targetLv = null;

                // Simular construcción
                for (let b of seq) {
                    if (!b) continue;
                    simLvls[b] = (simLvls[b] || 0) + 1;
                    if (simLvls[b] > (parseInt(curLvls[b]) || 0)) {
                        nextB = b;
                        targetLv = simLvls[b];
                        break;
                    }
                }

                if (!nextB) {
                    $('#am_results').html('<b style="color:green">¡Pueblo completado al 100% según plantilla!</b>');
                    return;
                }

                // Calcular recursos necesarios
                let node = $(xml).find(nextB);
                let w = Math.round(node.find('wood').text() * Math.pow(node.find('wood_factor').text(), targetLv - 1));
                let s = Math.round(node.find('stone').text() * Math.pow(node.find('stone_factor').text(), targetLv - 1));
                let i = Math.round(node.find('iron').text() * Math.pow(node.find('iron_factor').text(), targetLv - 1));

                let mw = Math.max(0, w - game_data.village.wood);
                let ms = Math.max(0, s - game_data.village.stone);
                let mi = Math.max(0, i - game_data.village.iron);

                let resHtml = `<b>Siguiente Edificio:</b> ${nextB} (Nivel ${targetLv})<br><br>`;
                
                if (mw === 0 && ms === 0 && mi === 0) {
                    resHtml += `<b style="color:green">¡Tienes recursos de sobra para construirlo!</b>`;
                } else {
                    resHtml += `<b>Faltan:</b> <br>
                    <span class="icon header wood"></span> ${mw} &nbsp;&nbsp;
                    <span class="icon header stone"></span> ${ms} &nbsp;&nbsp;
                    <span class="icon header iron"></span> ${mi} <br><br>`;

                    // BOTÓN PARA ENVIAR RECURSOS (Totalmente legal: Abre el mercado)
                    let marketLink = `/game.php?village=${vId}&screen=market&mode=send&wood=${mw}&stone=${ms}&iron=${mi}`;
                    
                    resHtml += `<i>Nota: Ve a otro pueblo tuyo desde donde quieras mandar recursos, pulsa este script de nuevo, y usa este enlace:</i><br><br>
                    <a href="${marketLink}" class="btn btn-default" target="_blank" style="background:#007500; color:white;">💰 Rellenar Mercado hacia aquí</a>`;
                }
                
                $('#am_results').html(resHtml);
            }
        });
    });
})();
