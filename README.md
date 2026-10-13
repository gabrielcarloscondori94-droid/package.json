# package.json
{   "name": "juego-misterio",   "version": "1.0.0",   "scripts": {     "start": "node server.js"   },   "dependencies": {     "express": "^4.19.2",     "socket.io": "^4.7.5"   } }
const socket = io();
let estadoActual = "ESPERANDO";
let rolGuardado = "";
let radarUtilizado = false;

function entrarAMansion() {
    const alias = document.getElementById('input-name').value.trim();
    if (!alias) return alert("Por favor, introduce un nombre válido para el registro.");
    socket.emit('unirse', alias);
    document.getElementById('view-login').style.display = 'none';
    document.getElementById('view-game').style.display = 'block';
}

function iniciarCaso() { socket.emit('iniciarPartida'); }
function llamarADiscusion() { socket.emit('reportarCrimen'); }

function enviarMensaje() {
    const input = document.getElementById('input-chat');
    if (!input.value.trim()) return;
    socket.emit('enviarMensaje', input.value);
    input.value = '';
}

function evaluarTeclado(e) { if (e.key === 'Enter') enviarMensaje(); }

socket.on('errorMsg', (msg) => alert(msg));

socket.on('recibirPistas', (pistas) => {
    document.getElementById('my-clues').innerHTML = `<div style="color: #ffdd99; font-style: italic;">🔍 Pista Guardada: <strong>${pistas[0]}</strong></div>`;
});

socket.on('resultadoHabilidad', (mensaje) => {
    const radar = document.getElementById('radar-box');
    radar.style.display = 'block';
    radar.innerText = mensaje;
    radarUtilizado = true;
});

socket.on('tickContador', (segundos) => {
    const timer = document.getElementById('timer');
    timer.innerHTML = estadoActual === "DISCUSION" 
        ? `⏱️ FALTAN <strong style="color: #ff3333; font-size: 1.5rem;">${segundos}s</strong> PARA EMITIR EL VOTO FINAL` 
        : `🕵️ Fase de Investigación activa... inspeccionen los alrededores.`;
});

socket.on('recibirMensaje', (data) => {
    const caja = document.getElementById('chat-box');
    caja.innerHTML += `<div class="chat-msg"><strong style="color: ${data.color}">${data.nombre}:</strong> <span style="color: #eee">${data.texto}</span></div>`;
    caja.scrollTop = caja.scrollHeight;
});

socket.on('asignarRol', (datos) => {
    rolGuardado = datos.rol;
    const banner = document.getElementById('role-banner');
    
    let colorDestacado = datos.rol === "ASESINO" || datos.rol === "CÓMPLICE" ? "#ff4444" : "#44ff44";
    banner.innerHTML = `Identidad Secreta: <strong style="color: ${colorDestacado}; font-size: 1.3rem;">${datos.rol}</strong><br><span style="color: #bbb; font-size: 0.95rem;">${datos.instruccion}</span>`;
    
    if (datos.rol === "DETECTIVE") {
        document.getElementById('radar-box').style.display = 'block';
        document.getElementById('radar-box').innerText = "🔭 Tienes una lupa de detective disponible para usar sobre un sospechoso.";
    } else if (datos.rol === "ASESINO" || datos.rol === "CÓMPLICE") {
        document.getElementById('my-clues').innerHTML = `<span style="color: #ff5555;">🔥 Saboteadores: No tienes pistas inocentes. Engaña al resto.</span>`;
    }
});

socket.on('actualizarJugadores', (lista) => {
    const contenedor = document.getElementById('player-list');
    contenedor.innerHTML = '';

    // El primer jugador conectado toma la presidencia para iniciar el juego
    if (lista.length >= 4 && estadoActual === "ESPERANDO") {
        document.getElementById('btn-start').style.display = 'block';
    }

    lista.forEach(j => {
        const card = document.createElement('div');
        card.className = 'player-card';
        card.style.setProperty('--clr', j.color);

        let estadoInfo = j.vivo ? '<span style="color: #27ae60">👁️ En el salón</span>' : '<span style="color: #95a5a6; text-decoration: line-through;">💀 Asesinado</span>';
        
        let subNodosBotones = '';
        if (estadoActual === "DISCUSION" && j.vivo && j.id !== socket.id) {
            subNodosBotones += `<button style="padding: 4px 8px; font-size: 0.8rem; background: #7a1515;" onclick="socket.emit('votarJugador', '${j.id}')">Acusar</button>`;
        }
        
        if (estadoActual === "INVESTIGACION" && rolGuardado === "DETECTIVE" && !radarUtilizado && j.id !== socket.id && j.vivo) {
            subNodosBotones += `<button style="padding: 4px 8px; font-size: 0.8rem; background: #1c5276;" onclick="socket.emit('usarHabilidadDetective', '${j.id}')">🔍 Analizar</button>`;
        }

        card.innerHTML = `
            <div class="player-info">
                <div class="avatar-dot"></div>
                <div>
                    <strong style="color: #fff">${j.nombre}</strong> <br>
                    <small>${estadoInfo}</small>
                </div>
            </div>
            <div>
                ${j.votos > 0 ? `<span class="badge">Sospechas: \${j.votos}</span>` : ''}
                ${subNodosBotones}
            </div>
        `;
        contenedor.appendChild(card);
    });
});

socket.on('cambioEstado', (datos) => {
    estadoActual = datos.estado;
    document.getElementById('timer').innerText = datos.mensaje;
    
    if (datos.estado === "INVESTIGACION") {
        document.getElementById('btn-start').style.display = 'none';
        document.getElementById('btn-report').style.display = 'block';
    } else if (datos.estado === "DISCUSION") {
        document.getElementById('btn-report').style.display = 'none';
    }
});

socket.on('resultadoVotacion', (datos) => {
    alert(datos.mensaje);
    if (datos.fin) {
        window.location.reload();
    }
});
