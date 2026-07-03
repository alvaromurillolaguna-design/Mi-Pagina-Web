[INDEX.html.html](https://github.com/user-attachments/files/29630281/INDEX.html.html)
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Tetris 1984 - Con Sonido Retro PC Speaker</title>
    <style>
        body { 
            background-color: #020a02; 
            color: #33ff33; 
            font-family: 'Courier New', Courier, monospace; 
            display: flex; 
            flex-direction: column; 
            align-items: center; 
            justify-content: center; 
            min-height: 100vh; 
            margin: 0; 
            padding: 10px;
            box-sizing: border-box;
            overflow-x: hidden;
            -webkit-tap-highlight-color: transparent; /* Elimina destello azul al tocar en móviles */
        }

        .pantalla-crt {
            position: relative;
            padding: 10px;
            background: #000;
            border-radius: 15px;
            box-shadow: inset 0 0 40px rgba(0,0,0,1), 0 0 30px rgba(0, 255, 0, 0.3);
            max-width: 100%;
            box-sizing: border-box;
        }

        canvas { 
            border: 4px solid #33ff33; 
            background-color: #000000; 
            display: block;
            max-width: 100%;
            height: auto; /* Hace que el lienzo sea elástico en móviles */
        }

        .pantalla-crt::after {
            content: " ";
            display: block;
            position: absolute;
            top: 0; left: 0; bottom: 0; right: 0;
            background: linear-gradient(rgba(18, 16, 16, 0) 50%, rgba(0, 0, 0, 0.25) 50%), linear-gradient(90deg, rgba(255, 0, 0, 0.05), rgba(0, 255, 0, 0.02), rgba(0, 0, 255, 0.05));
            z-index: 2;
            background-size: 100% 4px, 6px 100%;
            pointer-events: none;
        }

        /* --- CONTROLES VIRTUALES PARA MÓVIL --- */
        .controles-movil {
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            gap: 10px;
            margin-top: 15px;
            width: 100%;
            max-width: 440px;
            box-sizing: border-box;
        }

        .btn-control {
            background: #051a05;
            border: 2px solid #33ff33;
            color: #33ff33;
            font-family: 'Courier New', monospace;
            font-size: 1.3rem;
            font-weight: bold;
            padding: 15px 5px;
            text-align: center;
            border-radius: 8px;
            cursor: pointer;
            user-select: none;
            box-shadow: 0 3px 0 #115511;
        }

        .btn-control:active {
            background: #33ff33;
            color: #000;
            transform: translateY(2px);
            box-shadow: 0 1px 0 #115511;
        }

        .largo {
            grid-column: span 3;
            padding: 10px;
            font-size: 1rem;
        }
    </style>
</head>
<body>

    <div class="pantalla-crt">
        <canvas id="tablero"></canvas>
    </div>

    <div class="controles-movil">
        <div class="btn-control" id="m-rotar">ROTAR (↑)</div>
        <div class="btn-control" id="m-pausa">PAUSA (P)</div>
        <div class="btn-control" id="m-menu">MENÚ (ESC)</div>
        
        <div class="btn-control" id="m-izq">◀ IZQ</div>
        <div class="btn-control" id="m-abajo">▼ BAJAR</div>
        <div class="btn-control" id="m-der">DER ▶</div>
    </div>

    <script>
        const canvas = document.getElementById('tablero');
        const context = canvas.getContext('2d');

        const TAMANO_FUENTE = 26;
        const COLUMNAS = 10;
        const FILAS = 20;
        
        canvas.width = 960;
        canvas.height = (FILAS + 3) * TAMANO_FUENTE; 
        
        const BIEN_X = 340; 
        const BIEN_Y = 40;

        // --- 🔉 MOTOR DE AUDIO RETRO ---
        let audioCtx = null;

        function iniciarAudio() {
            if (!audioCtx) {
                audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            }
        }

        function pitar(frecuencia, duracion, tipo = 'square', volumen = 0.1) {
            if (!audioCtx) return;
            try {
                let osc = audioCtx.createOscillator();
                let gainNode = audioCtx.createGain();
                
                osc.type = tipo;
                osc.frequency.setValueAtTime(frecuencia, audioCtx.currentTime);
                gainNode.gain.setValueAtTime(volumen, audioCtx.currentTime);
                gainNode.gain.exponentialRampToValueAtTime(0.00001, audioCtx.currentTime + duracion);
                
                osc.connect(gainNode);
                gainNode.connect(audioCtx.destination);
                
                osc.start();
                osc.stop(audioCtx.currentTime + duracion);
            } catch(e) { console.log("Error de audio:", e); }
        }

        function sonidoMenu() { pitar(600, 0.05, 'square', 0.05); }
        function sonidoAceptar() { 
            pitar(400, 0.08, 'square', 0.08);
            setTimeout(() => pitar(800, 0.15, 'square', 0.08), 80);
        }
        function sonidoMover() { pitar(150, 0.03, 'triangle', 0.15); }
        function sonidoRotar() { pitar(350, 0.05, 'square', 0.08); }
        function sonidoPausa() { 
            pitar(500, 0.1, 'square', 0.08);
            setTimeout(() => pitar(350, 0.1, 'square', 0.08), 100);
        }
        function sonidoLinea() {
            pitar(600, 0.1, 'square', 0.1);
            setTimeout(() => pitar(900, 0.1, 'square', 0.1), 100);
            setTimeout(() => pitar(1200, 0.2, 'square', 0.1), 200);
        }
        function sonidoGameOver() {
            pitar(400, 0.2, 'square', 0.1);
            setTimeout(() => pitar(300, 0.2, 'square', 0.1), 200);
            setTimeout(() => pitar(200, 0.4, 'square', 0.1), 400);
        }

        // --- LÓGICA DEL JUEGO ---
        const PIEZAS = {
            'I': [[0,0,0,0], [1,1,1,1], [0,0,0,0], [0,0,0,0]],
            'L': [[0,0,1], [1,1,1], [0,0,0]],
            'J': [[1,0,0], [1,1,1], [0,0,0]],
            'O': [[1,1], [1,1]],
            'Z': [[1,1,0], [0,1,1], [0,0,0]],
            'S': [[0,1,1], [1,1,0], [0,0,0]],
            'T': [[0,1,0], [1,1,1], [0,0,0]]
        };

        const tablero = Array.from({ length: FILAS }, () => Array(COLUMNAS).fill('.'));

        let lineasAnimando = [];
        let contadorAnimacionLinea = 0;

        let pieza = {
            matriz: PIEZAS['T'],
            pos: {x: 3, y: 0}
        };

        let lineas = 0;
        let nivel = 1;
        let puntos = 0;
        
        let estadoActual = 'PORTADA'; 
        let seleccionMenu = 0; 
        const OPCIONES_MENU = ["JUGAR PARTIDA", "VER CONTROLES", "APAGAR SISTEMA"];

        function fusionar(tablero, pieza) {
            pieza.matriz.forEach((fila, y) => {
                fila.forEach((valor, x) => {
                    if (valor !== 0) {
                        if (tablero[y + pieza.pos.y]) {
                            tablero[y + pieza.pos.y][x + pieza.pos.x] = '[]';
                        }
                    }
                });
            });
        }

        function comprobarLineas() {
            lineasAnimando = [];
            for (let y = FILAS - 1; y >= 0; y--) {
                if (!tablero[y].includes('.')) {
                    lineasAnimando.push(y);
                }
            }
            if (lineasAnimando.length > 0) {
                contadorAnimacionLinea = 10;
                sonidoLinea();
            }
        }

        function procesarBorradoReal() {
            for (let y = FILAS - 1; y >= 0; y--) {
                if (!tablero[y].includes('.')) {
                    tablero.splice(y, 1);
                    tablero.unshift(Array(COLUMNAS).fill('.'));
                    y++; 
                }
            }
            lineas += lineasAnimando.length;
            puntos += lineasAnimando.length * 100 * nivel;
            nivel = Math.floor(lineas / 5) + 1;
            intervaloCaida = Math.max(100, 600 - (nivel * 50)); 
            lineasAnimando = [];
        }

        let contadorCaida = 0;
        let intervaloCaida = 600; 
        let tiempoUltimo = 0;

        function bajarPieza() {
            if (lineasAnimando.length > 0 || estadoActual !== 'JUEGO') return;

            pieza.pos.y++;
            if (detectarColision()) {
                pieza.pos.y--;
                fusionar(tablero, pieza);
                comprobarLineas();
                if (lineasAnimando.length === 0) {
                    generarNuevaPieza();
                }
            }
        }

        function reiniciarJuego() {
            tablero.forEach(fila => fila.fill('.'));
            lineas = 0; 
            puntos = 0; 
            nivel = 1; 
            intervaloCaida = 600;
            estadoActual = 'JUEGO';
            generarNuevaPieza();
        }

        function generarNuevaPieza() {
            pieza.pos.y = 0;
            pieza.pos.x = 3;
            
            if (detectarColision()) {
                sonidoGameOver();
                alert("GAME OVER - Puntuación: " + puntos);
                estadoActual = 'MENU';
            } else {
                const claves = Object.keys(PIEZAS);
                const claveAleatoria = claves[Math.floor(Math.random() * claves.length)];
                pieza.matriz = PIEZAS[claveAleatoria];
            }
        }

        // Asegurar inicio de audio en cualquier interacción táctil inicial en el cuerpo de la web
        document.body.addEventListener('touchstart', () => { iniciarAudio(); }, {passive: true});

        function detectarColision() {
            const m = pieza.matriz;
            const o = pieza.pos;
            for (let y = 0; y < m.length; ++y) {
                for (let x = 0; x < m[y].length; ++x) {
                    if (m[y][x] !== 0) {
                        if (y + o.y >= FILAS || x + o.x < 0 || x + o.x >= COLUMNAS || tablero[y + o.y][x + o.x] === '[]') {
                            return true;
                        }
                    }
                }
            }
            return false;
        }

        function rotate(matriz) {
            const nuevaMatriz = matriz[0].map((_, i) => matriz.map(fila => fila[i]));
            return nuevaMatriz.map(fila => fila.reverse());
        }

        // --- PROCESAR ACCIONES COMUNES (MÓVIL Y PC) ---
        function accionIzquierda() {
            if (estadoActual === 'JUEGO' && lineasAnimando.length === 0) {
                pieza.pos.x--;
                if (detectarColision()) pieza.pos.x++; else sonidoMover();
            }
        }

        function accionDerecha() {
            if (estadoActual === 'JUEGO' && lineasAnimando.length === 0) {
                pieza.pos.x++;
                if (detectarColision()) pieza.pos.x--; else sonidoMover();
            }
        }

        function accionAbajo() {
            if (estadoActual === 'MENU') {
                seleccionMenu = (seleccionMenu + 1) % OPCIONES_MENU.length;
                sonidoMenu();
            } else if (estadoActual === 'JUEGO') {
                bajarPieza();
                sonidoMover();
            }
        }

        function accionArriba() {
            if (estadoActual === 'MENU') {
                seleccionMenu = (seleccionMenu - 1 + OPCIONES_MENU.length) % OPCIONES_MENU.length;
                sonidoMenu();
            } else if (estadoActual === 'JUEGO' && lineasAnimando.length === 0) {
                const matrizAnterior = pieza.matriz;
                pieza.matriz = rotate(pieza.matriz);
                if (detectarColision()) pieza.matriz = matrizAnterior; else sonidoRotar();
            }
        }

        function accionAceptar() {
            if (estadoActual === 'PORTADA') {
                sonidoAceptar(); estadoActual = 'MENU';
            } else if (estadoActual === 'CONTROLES' || estadoActual === 'APAGADO') {
                sonidoAceptar(); estadoActual = 'MENU';
            } else if (estadoActual === 'MENU') {
                sonidoAceptar();
                if (seleccionMenu === 0) reiniciarJuego();
                else if (seleccionMenu === 1) estadoActual = 'CONTROLES';
                else if (seleccionMenu === 2) estadoActual = 'APAGADO';
            }
        }

        function accionPausa() {
            if (estadoActual === 'JUEGO' || estadoActual === 'PAUSA') {
                sonidoPausa();
                estadoActual = (estadoActual === 'JUEGO') ? 'PAUSA' : 'JUEGO';
            }
        }

        function accionMenuPrincipal() {
            if (estadoActual === 'JUEGO' || estadoActual === 'PAUSA' || estadoActual === 'CONTROLES' || estadoActual === 'APAGADO') {
                sonidoAceptar();
                estadoActual = 'MENU';
            }
        }

        // --- ASIGNACIÓN DE BOTONES TÁCTILES MÓVILES ---
        function registrarBotonTáctil(id, callback) {
            const btn = document.getElementById(id);
            btn.addEventListener('touchstart', (e) => {
                e.preventDefault();
                iniciarAudio();
                callback();
            }, {passive: false});
        }

        registrarBotonTáctil('m-rotar', () => {
            if (estadoActual === 'PORTADA' || estadoActual === 'MENU' || estadoActual === 'CONTROLES') {
                accionArriba();
            } else {
                accionArriba();
            }
        });
        registrarBotonTáctil('m-pausa', accionPausa);
        registrarBotonTáctil('m-menu', accionMenuPrincipal);
        registrarBotonTáctil('m-izq', accionIzquierda);
        registrarBotonTáctil('m-der', accionDerecha);
        registrarBotonTáctil('m-abajo', () => {
            if (estadoActual === 'PORTADA' || estadoActual === 'MENU' || estadoActual === 'CONTROLES') {
                accionAceptar(); // En menús, el botón central acepta/entra
            } else {
                accionAbajo();
            }
        });

        // Soporte de toque en la pantalla CRT directamente para salir de portadas
        canvas.addEventListener('touchstart', (e) => {
            e.preventDefault();
            iniciarAudio();
            if (estadoActual === 'PORTADA' || estadoActual === 'CONTROLES' || estadoActual === 'APAGADO') {
                accionAceptar();
            }
        }, {passive: false});


        // --- MANEJO DE TECLADO ORIGINAL (MANTENIDO PARA PC) ---
        document.addEventListener('keydown', evento => {
            const tecla = evento.key;
            iniciarAudio();

            if (estadoActual === 'PORTADA') {
                accionAceptar(); return;
            }
            if (estadoActual === 'CONTROLES' || estadoActual === 'APAGADO') {
                if (tecla.toLowerCase() === 'r' || tecla === 'Escape' || tecla === 'Enter') accionAceptar();
                return;
            }
            if (estadoActual === 'MENU') {
                if (tecla === 'ArrowUp') accionArriba();
                else if (tecla === 'ArrowDown') accionAbajo();
                else if (tecla === 'Enter') accionAceptar();
                return;
            }
            if (estadoActual === 'JUEGO' || estadoActual === 'PAUSA') {
                if (tecla.toLowerCase() === 'p') { accionPausa(); return; }
                if (tecla === 'Escape') { accionMenuPrincipal(); return; }
                if (tecla.toLowerCase() === 'q') { estadoActual = 'APAGADO'; return; }
                if (estadoActual === 'PAUSA' || lineasAnimando.length > 0) return;

                if (tecla === 'ArrowLeft') accionIzquierda();
                else if (tecla === 'ArrowRight') accionDerecha();
                else if (tecla === 'ArrowDown') accionAbajo();
                else if (tecla === 'ArrowUp') accionArriba();
            }
        });

        // --- RENDERIZADOR GRÁFICO ORIGINAL ---
        function dibujarInterfaz() {
            context.fillStyle = '#000000';
            context.fillRect(0, 0, canvas.width, canvas.height);
            
            context.shadowBlur = 8;
            context.shadowColor = "#33ff33";
            context.fillStyle = '#33ff33';

            if (estadoActual === 'PORTADA') {
                context.font = "bold 50px 'Courier New', Courier, monospace";
                context.fillText("T E T R I S", 310, 220);
                context.font = "bold 24px 'Courier New', Courier, monospace";
                context.fillText("VERSION ORIGINAL 1984", 330, 280);
                
                if (Math.floor(Date.now() / 500) % 2 === 0) {
                    context.fillText("PULSA ABAJO / PANTALLA PARA EMPEZAR", 190, 420);
                }
                context.shadowBlur = 0;
                return;
            }

            if (estadoActual === 'MENU') {
                context.font = "bold 36px 'Courier New', Courier, monospace";
                context.fillText("MENÚ PRINCIPAL", 340, 150);

                context.font = "bold 26px 'Courier New', Courier, monospace";
                for (let i = 0; i < OPCIONES_MENU.length; i++) {
                    if (i === seleccionMenu) {
                        context.fillText(`> ${OPCIONES_MENU[i]} <`, 320, 260 + (i * 60));
                    } else {
                        context.fillText(`  ${OPCIONES_MENU[i]}  `, 320, 260 + (i * 60));
                    }
                }
                context.font = "20px 'Courier New', Courier, monospace";
                context.fillText("Usa ROTAR/ABAJO para moverte y [▼ BAJAR] para aceptar", 155, 500);
                context.shadowBlur = 0;
                return;
            }

            if (estadoActual === 'CONTROLES') {
                context.font = "bold 32px 'Courier New', Courier, monospace";
                context.fillText("CONFIGURACIÓN DE MANDOS", 270, 100);

                context.font = "24px 'Courier New', Courier, monospace";
                let cmds = [
                    "Flecha IZQUIERDA [←] : Mover a la izquierda",
                    "Flecha DERECHA   [→] : Mover a la derecha",
                    "Flecha ARRIBA    [↑] : Rotar pieza",
                    "Flecha ABAJO     [↓] : Caída rápida",
                    "Tecla [P]            : Pausar partida",
                    "Tecla [ESC]          : Volver al Menú",
                    "Tecla [Q]            : Apagar terminal"
                ];

                for (let i = 0; i < cmds.length; i++) {
                    context.fillText(cmds[i], 180, 180 + (i * 45));
                }

                context.fillText("Pulsa el botón [▼ BAJAR] para volver", 240, 540);
                context.shadowBlur = 0;
                return;
            }

            if (estadoActual === 'APAGADO') {
                context.font = "bold 26px 'Courier New', Courier, monospace";
                context.fillText("SISTEMA APAGADO", 380, 260);
                context.fillText("PULSA [▼ BAJAR] PARA VOLVER A ENCENDER", 210, 310);
                context.shadowBlur = 0;
                return;
            }

            // PANTALLA JUEGO
            context.font = `bold ${TAMANO_FUENTE}px 'Courier New', Courier, monospace`;
            context.fillText(`lines:  ${lineas}`, 20, 100);
            context.fillText(`level:  ${nivel}`, 20, 150);
            context.fillText(`points: ${puntos}`, 20, 200);
            
            if (estadoActual === 'PAUSA') {
                context.fillText("** PAUSA **", 20, 300);
            }

            context.fillText("ROTAR",     730, 100);
            context.fillText("IZQUIERDA", 730, 150);
            context.fillText("DERECHA",   730, 200);
            context.fillText("ACELERAR",  730, 250);
            context.fillText("PAUSA",     730, 320);
            context.fillText("MENU",      730, 370);

            context.fillText("[↑]", 880, 100);
            context.fillText("[←]", 880, 150);
            context.fillText("[→]", 880, 200); 
            context.fillText("[↓]", 880, 250);
            context.fillText("[P]", 880, 320);
            context.fillText("[ESC]", 850, 370);

            for (let y = 0; y < FILAS; y++) {
                let lineaTexto = "<!"; 
                let estaBrillando = lineasAnimando.includes(y) && (contadorAnimacionLinea % 2 === 0);

                for (let x = 0; x < COLUMNAS; x++) {
                    let esPieceActiva = false;
                    const py = y - pieza.pos.y;
                    const px = x - pieza.pos.x;
                    
                    if (py >= 0 && py < pieza.matriz.length && px >= 0 && px < pieza.matriz[py].length) {
                        if (pieza.matriz[py][px] !== 0) esPieceActiva = true;
                    }

                    if (estaBrillando) {
                        lineaTexto += "##"; 
                    } else if (esPieceActiva || tablero[y][x] === '[]') {
                        lineaTexto += "[]";
                    } else {
                        lineaTexto += " ."; 
                    }
                }
                lineaTexto += "!>"; 
                context.fillText(lineaTexto, BIEN_X, BIEN_Y + (y * TAMANO_FUENTE));
            }

            context.fillText("<" + "=".repeat(COLUMNAS * 2) + ">", BIEN_X, BIEN_Y + (FILAS * TAMANO_FUENTE));
            context.fillText("\\" + "/".repeat(COLUMNAS * 2) + "/", BIEN_X, BIEN_Y + ((FILAS + 1) * TAMANO_FUENTE));
            context.shadowBlur = 0;
        }

        function actualizar(tiempo = 0) {
            const tiempoDiferencia = tiempo - tiempoUltimo;
            tiempoUltimo = tiempo;

            if (estadoActual === 'JUEGO') {
                if (lineasAnimando.length > 0) {
                    contadorAnimacionLinea--;
                    if (contadorAnimacionLinea <= 0) {
                        procesarBorradoReal();
                        generarNuevaPieza();
                    }
                } else {
                    contadorCaida += tiempoDiferencia;
                    if (contadorCaida > intervaloCaida) {
                        bajarPieza();
                        contadorCaida = 0;
                    }
                }
            }

            dibujarInterfaz();
            requestAnimationFrame(actualizar);
        }

        actualizar();
    </script>
</body>
</html>
