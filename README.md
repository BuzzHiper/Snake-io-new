<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Snake Game</title>
    <!-- Carga de Tailwind CSS para utilidades base y layout -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Carga de Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, query, limit, orderBy, onSnapshot, setDoc, doc, runTransaction, getDoc, where } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Establecer nivel de log para depuración de Firestore
        setLogLevel('Debug');

        // Variables globales (se llenan en setupFirebase)
        window.db = null;
        window.auth = null;
        window.userId = null;
        window.appId = null;
        window.isAuthReady = false;

        // Configuración de Firebase (proporcionada por el entorno)
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        window.appId = typeof __app_id !== 'undefined' ? __app_id : 'default-snake-app';
        
        // --- Funciones Auxiliares para Firebase ---
        function getHighScoresRef(db) {
            // Ruta pública para puntajes altos: /artifacts/{appId}/public/data/snake_high_scores
            return collection(db, 'artifacts', window.appId, 'public/data', 'snake_high_scores');
        }
        
        function getUserProfileRef(db) {
            // Ruta privada para el perfil del usuario: /artifacts/{appId}/users/{userId}/profile/user_data
            if (!window.userId) return null;
            return doc(db, 'artifacts', window.appId, 'users', window.userId, 'profile', 'user_data');
        }

        async function loadUserProfile() {
            if (!window.isAuthReady || !window.db || !window.userId) return;

            const profileRef = getUserProfileRef(window.db);
            if (!profileRef) return;
            
            try {
                const docSnap = await getDoc(profileRef);
                let currentName = 'Jugador Anónimo';
                let hasName = false;

                if (docSnap.exists() && docSnap.data().name) {
                    currentName = docSnap.data().name;
                    hasName = true;
                }
                
                // Actualiza la variable de nombre global del juego (en el otro script)
                window.updateUserName(currentName);
                
                // Si no tiene nombre, pedírselo al usuario (isFirstTime = true)
                if (!hasName) {
                    // Esperamos un momento para que el DOM del juego esté listo y evitar errores de carga
                    setTimeout(() => {
                        window.showNameModal(true);
                    }, 500);
                }

            } catch (error) {
                console.error("Error al cargar el perfil del usuario:", error);
                window.updateUserName('Jugador Anónimo');
            }
        }
        
        // Función para guardar el nombre de usuario de forma persistente
        window.saveUserName = async (name) => {
             if (!window.isAuthReady || !window.db || !window.userId) return;

             const profileRef = getUserProfileRef(window.db);
             if (!profileRef) return;

             try {
                await setDoc(profileRef, { name: name, lastUpdated: new Date().toISOString() }, { merge: true });
                console.log("Nombre de usuario guardado persistentemente:", name);
                
                // Actualiza el mensaje de bienvenida
                if (document.getElementById('game-message').textContent.startsWith('Cargando perfil') || document.getElementById('game-message').textContent.startsWith('¡Hola')) {
                    document.getElementById('game-message').textContent = `¡Hola, ${name}! Presiona Jugar para empezar.`;
                }

             } catch(error) {
                console.error("Error al guardar el nombre de usuario:", error);
             }
        }

        // Función de inicialización de Firebase
        async function setupFirebase() {
            if (!firebaseConfig) {
                console.error("Firebase config not available. High scores disabled.");
                window.isAuthReady = true; 
                return;
            }
            
            try {
                const app = initializeApp(firebaseConfig);
                window.db = getFirestore(app);
                window.auth = getAuth(app);
                
                // 1. Autenticación (para obtener el userId)
                if (initialAuthToken) {
                    await signInWithCustomToken(window.auth, initialAuthToken);
                } else {
                    await signInAnonymously(window.auth);
                }

                // 2. Establecer el listener de estado de autenticación
                onAuthStateChanged(window.auth, (user) => {
                    if (user) {
                        window.userId = user.uid;
                        document.getElementById('user-id-display').textContent = 'ID de Usuario: ' + user.uid;
                    } else {
                        // Usar el UID anónimo o generar uno si es necesario
                        window.userId = window.auth.currentUser?.uid || crypto.randomUUID();
                        document.getElementById('user-id-display').textContent = 'ID de Sesión: ' + window.userId;
                    }
                    window.isAuthReady = true;
                    // Llama a las funciones de carga de datos una vez que la autenticación esté lista
                    loadHighScores();
                    loadUserProfile(); 
                });
            } catch (error) {
                console.error("Error al configurar Firebase:", error);
                window.isAuthReady = true; 
                document.getElementById('user-id-display').textContent = 'Error de Auth';
            }
        }

        // 3. Función para cargar puntajes altos
        function loadHighScores() {
            if (!window.isAuthReady || !window.db) return;
            
            const highScoresRef = getHighScoresRef(window.db);
            const q = query(highScoresRef, orderBy('score', 'desc'), limit(5));

            onSnapshot(q, (snapshot) => {
                const scores = [];
                snapshot.forEach((doc) => {
                    scores.push(doc.data());
                });
                displayHighScores(scores);
            }, (error) => {
                console.error("Error al escuchar los puntajes altos:", error);
            });
        }

        // 4. Función para guardar puntajes (con transacción para verificar)
        window.saveHighScore = async (newScore, newUserName) => {
            if (!window.isAuthReady || !window.db) {
                console.warn("Firebase no está listo para guardar el puntaje.");
                return;
            }

            const highScoresRef = getHighScoresRef(window.db);
            const userScoreDocRef = doc(highScoresRef, window.userId); // Documento específico para el puntaje del usuario

            try {
                await runTransaction(window.db, async (transaction) => {
                    // Primero, intenta obtener el puntaje actual del usuario
                    const userScoreDoc = await transaction.get(userScoreDocRef);
                    
                    let currentHighScore = 0;
                    if (userScoreDoc.exists()) {
                        currentHighScore = userScoreDoc.data().score || 0;
                    }

                    // Si el nuevo puntaje es mayor, actualiza
                    if (newScore > currentHighScore) {
                        transaction.set(userScoreDocRef, {
                            userId: window.userId,
                            score: newScore,
                            date: new Date().toISOString(),
                            userName: newUserName // Guardar el nombre
                        });
                        document.getElementById('game-message').textContent = `¡NUEVO RÉCORD: ${newUserName}!`;
                        console.log("Puntaje alto guardado:", newScore, newUserName);
                    } else {
                        console.log("El puntaje no superó el récord personal.");
                    }
                });
            } catch (e) {
                console.error("Error en la transacción al guardar el puntaje alto:", e);
                document.getElementById('game-message').textContent = 'Error al guardar el puntaje alto.';
            }
        };

        // --- Funciones para la UI ---
        function displayHighScores(scores) {
            const ul = document.getElementById('high-scores-list');
            ul.innerHTML = '';
            
            if (scores.length === 0) {
                ul.innerHTML = '<li class="text-gray-400 p-2 text-center">Sé el primero en anotar.</li>';
                return;
            }

            scores.forEach((item, index) => {
                const li = document.createElement('li');
                
                // Usar el nombre de usuario guardado o un ID parcial como fallback
                const displayName = item.userName || item.userId.substring(0, 8) + '...'; 
                // Mostrar 'TÚ' si es el usuario actual, de lo contrario, mostrar el nombre/ID
                const displayId = item.userId === window.userId ? (item.userName || 'TÚ') : displayName;
                
                const rankColor = index === 0 ? 'text-yellow-400 font-bold' : 'text-white';
                
                li.className = `flex justify-between items-center p-2 rounded-lg ${index % 2 === 0 ? 'bg-gray-700' : 'bg-gray-800'} ${rankColor}`;
                li.innerHTML = `
                    <span class="text-lg">${index + 1}.</span>
                    <span class="truncate ml-2">${displayId}</span>
                    <span class="text-xl">${item.score}</span>
                `;
                ul.appendChild(li);
            });
        }

        // Inicializar Firebase al cargar la ventana
        window.addEventListener('load', setupFirebase);
    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=VT323&display=swap');
        
        /* Estilos personalizados para el juego */
        body {
            background-color: #1f2937; /* Gris oscuro de Tailwind (bg-gray-800) */
            font-family: 'VT323', monospace;
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            color: #ecf0f1; /* Blanco suave */
            padding: 20px;
        }

        .game-container {
            display: grid;
            grid-template-columns: 1fr 280px; /* Canvas y Sidebar */
            gap: 24px;
            max-width: 1200px;
            width: 100%;
            padding: 24px;
            background: #2d3748; /* Gris azulado */
            border-radius: 20px;
            box-shadow: 0 10px 25px rgba(0, 0, 0, 0.5);
        }

        .game-main {
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
        }

        canvas {
            border: 5px solid #4a5568; /* Borde del canvas */
            background-color: #1e272e; /* Fondo oscuro del juego */
            border-radius: 10px;
            box-shadow: 0 0 20px rgba(0, 255, 0, 0.4); /* Sombra de neón verde */
            width: 100%;
            aspect-ratio: 1 / 1; /* Asegura que el canvas siempre sea cuadrado */
            max-width: 600px;
            margin-bottom: 20px;
        }

        .info-panel {
            background: #364052;
            padding: 20px;
            border-radius: 15px;
            box-shadow: inset 0 0 10px rgba(0, 0, 0, 0.3);
        }
        
        h1 {
            font-size: 3rem;
            text-shadow: 0 0 10px rgba(0, 255, 0, 0.7);
        }

        .game-score-display {
            font-size: 2.5rem;
            color: #38c172; /* Verde brillante */
            margin-bottom: 1rem;
            text-shadow: 0 0 5px #38c172;
        }

        .game-message {
            min-height: 40px;
            color: #f6993f; /* Naranja */
            font-size: 1.5rem;
            text-align: center;
        }

        button {
            background: linear-gradient(145deg, #38c172, #2d995b);
            color: white;
            padding: 10px 20px;
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3), inset 0 2px 4px rgba(255, 255, 255, 0.2);
            transition: all 0.1s;
            font-size: 1.5rem;
            cursor: pointer;
            border: none;
            width: 100%;
        }

        button:hover {
            background: linear-gradient(145deg, #2d995b, #38c172);
            box-shadow: 0 6px 8px rgba(0, 0, 0, 0.5), inset 0 1px 3px rgba(255, 255, 255, 0.3);
            transform: translateY(-2px);
        }
        
        button:active {
            transform: translateY(0);
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
        }

        /* Responsive adjustments */
        @media (max-width: 768px) {
            .game-container {
                grid-template-columns: 1fr;
                gap: 16px;
                padding: 16px;
            }
            .info-panel {
                order: -1; /* Mueve el panel de info/scores arriba en móvil */
            }
            canvas {
                max-width: 90vw;
            }
        }
    </style>
</head>
<body>

    <div class="game-container">
        <div class="game-main">
            <h1 class="text-green-400 mb-4">🐍 SNAKE.IO 🎮</h1>
            
            <div id="user-id-display" class="text-sm text-gray-400 mb-4">Cargando autenticación...</div>

            <!-- Canvas del Juego -->
            <canvas id="gameCanvas" width="400" height="400"></canvas>

            <div class="w-full max-w-[600px] flex flex-col items-center">
                <div id="score-display" class="game-score-display mb-4">Puntuación: 0</div>
                <div id="game-message" class="game-message mb-6">Cargando perfil...</div>
                <button id="startButton" onclick="startGame()">JUGAR / REINICIAR</button>
            </div>
            
            <!-- Controles de Móvil (ocultos en desktop) -->
            <div class="w-full max-w-[300px] mt-6 lg:hidden">
                <div class="flex justify-center mb-2">
                    <button class="w-20 h-20" onclick="setManualDirection('UP')">▲</button>
                </div>
                <div class="flex justify-between">
                    <button class="w-20 h-20" onclick="setManualDirection('LEFT')">◀</button>
                    <button class="w-20 h-20" onclick="setManualDirection('RIGHT')">►</button>
                </div>
                <div class="flex justify-center mt-2">
                    <button class="w-20 h-20" onclick="setManualDirection('DOWN')">▼</button>
                </div>
            </div>
        </div>

        <!-- Panel Lateral de Puntuaciones Altas -->
        <div class="info-panel">
            <h2 class="text-2xl text-center text-white mb-4 border-b-2 border-green-500 pb-2">PUNTUACIONES MÁS ALTAS</h2>
            <ul id="high-scores-list" class="space-y-2">
                <!-- Se llenará con JavaScript/Firestore -->
                <li class="text-gray-400 p-2 text-center">Cargando puntajes...</li>
            </ul>
            <p class="text-sm text-gray-400 mt-6 pt-4 border-t border-gray-600">
                Instrucciones: Usa las teclas de flecha (↑, ↓, ←, →) o W, A, S, D para mover la serpiente. La velocidad aumenta cada 5 puntos.
            </p>
        </div>
    </div>
    
    <!-- Modal para ingreso de nombre -->
    <div id="nameModal" class="fixed inset-0 bg-black bg-opacity-75 flex items-center justify-center hidden z-50">
        <div class="bg-gray-800 p-8 rounded-lg shadow-2xl w-80">
            <!-- ID para actualizar el título dinámicamente -->
            <h3 id="modalTitle" class="text-3xl text-green-400 mb-4 text-center">¡PUNTUACIÓN OBTENIDA!</h3> 
            <p class="text-white mb-4 text-center">Ingresa tu nombre para guardar tu récord:</p>
            <input type="text" id="userNameInput" maxlength="15" placeholder="Tu nombre (máx 15)" class="w-full p-3 mb-4 rounded bg-gray-700 text-white border border-green-500 focus:outline-none focus:ring-2 focus:ring-green-400 text-xl font-mono">
            <button id="saveNameButton" class="w-full" onclick="handleNameSave()">GUARDAR</button>
        </div>
    </div>

    <script>
        // --- Constantes y Variables del Juego ---
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        
        const TILE_SIZE = 20; // Tamaño de un segmento de la cuadrícula
        const GRID_SIZE = canvas.width / TILE_SIZE; // Número de celdas por lado (20x20)
        
        // Variables para la velocidad progresiva
        const BASE_SPEED = 150; // Velocidad inicial (ms) - 150ms = 6.6 FPS
        const SPEED_INCREMENT_THRESHOLD = 5; // Aumentar velocidad cada X puntos
        const SPEED_DECREMENT = 10; // Reducir intervalo de tiempo en Y ms
        const MIN_SPEED = 50; // Velocidad mínima (más rápido)

        let currentSpeed = BASE_SPEED;
        let speedLevel = 0;
        
        let snake;
        let food;
        let dx = 0; // Dirección X (cambio en la coordenada X)
        let dy = 0; // Dirección X (cambio en la coordenada X)
        let score;
        let gameLoopInterval;
        let isGameOver = true;
        let changingDirection = false; // Bandera para evitar giros dobles rápidos
        
        // Variables de Nombre de Usuario
        let userName = 'Jugador Anónimo'; // Nombre predeterminado
        const nameModal = document.getElementById('nameModal');
        const userNameInput = document.getElementById('userNameInput');

        const scoreDisplay = document.getElementById('score-display');
        const gameMessage = document.getElementById('game-message');
        
        // --- Funciones Globales para Firebase ---
        
        // Función para actualizar el nombre local del juego desde Firebase
        window.updateUserName = (name) => {
            userName = name;
            // Si el juego no ha empezado, mostrar un mensaje de bienvenida
            if (isGameOver) {
                gameMessage.textContent = `¡Hola, ${name}! Presiona Jugar para empezar.`;
            }
        };

        // Función para mostrar el modal de nombre (usada por Firebase y Game Over)
        window.showNameModal = (isFirstTime = false) => {
            document.getElementById('modalTitle').textContent = isFirstTime ? 
                '¡BIENVENIDO/A! Ingresa tu nombre:' : 
                '¡NUEVO RÉCORD! Ingresa tu nombre:';

            // Precargar el nombre actual si no es el anónimo, o dejar vacío si es la primera vez
            userNameInput.value = (userName !== 'Jugador Anónimo' && !isFirstTime) ? userName : '';
            nameModal.classList.remove('hidden');
            userNameInput.focus();
        };

        // --- Funciones de Inicialización y Bucle ---

        function setupGame() {
            snake = [
                { x: Math.floor(GRID_SIZE / 2), y: Math.floor(GRID_SIZE / 2) }
            ];
            dx = 1;
            dy = 0;
            score = 0;
            isGameOver = false;
            changingDirection = false;
            currentSpeed = BASE_SPEED; // Reiniciar velocidad
            speedLevel = 0;
            scoreDisplay.textContent = 'Puntuación: 0';
            gameMessage.textContent = '¡A comer!';
            
            // Limpiar cualquier intervalo anterior
            if (gameLoopInterval) clearInterval(gameLoopInterval);
            
            // Generar la primera comida
            generateFood();
        }

        function startGame() {
            if (isGameOver) {
                setupGame();
                // Usar la velocidad actual
                gameLoopInterval = setInterval(gameLoop, currentSpeed); 
            }
        }

        function generateFood() {
            food = {
                x: Math.floor(Math.random() * GRID_SIZE),
                y: Math.floor(Math.random() * GRID_SIZE)
            };
            
            // Asegurarse de que la comida no aparezca sobre la serpiente
            snake.forEach(segment => {
                if (segment.x === food.x && segment.y === food.y) {
                    generateFood(); // Llama recursivamente si hay colisión
                }
            });
        }

        function gameLoop() {
            if (checkCollision()) {
                gameOver();
                return;
            }

            changingDirection = false; // Restablecer la bandera de giro
            drawCanvas();
            updateSnakePosition();
            drawFood();
            drawSnake();
        }

        function gameOver() {
            isGameOver = true;
            clearInterval(gameLoopInterval);
            gameMessage.textContent = `¡Juego Terminado! Puntuación final: ${score}`;
            document.getElementById('startButton').textContent = 'VOLVER A JUGAR';
            
            // Si el puntaje es mayor que 0, muestra el modal para confirmar o cambiar el nombre
            // Si es 0, simplemente usa el nombre ya guardado
            if (score > 0) {
                window.showNameModal(false); 
            } else {
                 if (window.saveHighScore) {
                    window.saveHighScore(score, userName);
                }
            }
        }
        
        // Función para manejar el guardado del nombre desde el modal
        window.handleNameSave = () => {
            let inputName = userNameInput.value.trim();
            let newUserName = userName;

            if (inputName) {
                newUserName = inputName.substring(0, 15); // Limitar a 15 caracteres
            } else if (userName === 'Jugador Anónimo' || userNameInput.value.trim() === '') {
                 // Si el usuario no ingresa nada y aún tiene el nombre predeterminado
                newUserName = 'Jugador Anónimo';
            }
            
            userName = newUserName;

            // 1. Guardar el nombre de usuario de forma persistente a través de la función de Firebase
            if (window.saveUserName) {
                window.saveUserName(userName);
            }
            
            // 2. Ocultar modal
            nameModal.classList.add('hidden');
            
            // 3. Si el juego ya terminó (modal post-score), guardar el puntaje
            if (isGameOver && score > 0) {
                 if (window.saveHighScore) {
                    window.saveHighScore(score, userName);
                }
            }
            // Si el juego no terminó (modal on-load), el usuario está listo para jugar.
        };


        // --- Funciones de Dibujo ---

        function drawCanvas() {
            // Fondo de la cuadrícula
            ctx.fillStyle = '#1e272e'; 
            ctx.fillRect(0, 0, canvas.width, canvas.height);
        }

        function drawSnake() {
            snake.forEach((segment, index) => {
                // Segmentos
                ctx.fillStyle = index === 0 ? '#38c172' : '#65d68d'; // Cabeza verde más oscura
                ctx.strokeStyle = '#1e272e';
                ctx.fillRect(segment.x * TILE_SIZE, segment.y * TILE_SIZE, TILE_SIZE, TILE_SIZE);
                ctx.strokeRect(segment.x * TILE_SIZE, segment.y * TILE_SIZE, TILE_SIZE, TILE_SIZE);

                // Ojos (en la cabeza)
                if (index === 0) {
                    drawEyes(segment.x, segment.y, dx, dy);
                }
            });
        }
        
        function drawEyes(headX, headY, currentDx, currentDy) {
            const eyeSize = TILE_SIZE / 6;
            const offset = TILE_SIZE / 4;
            ctx.fillStyle = 'black';

            let eye1X, eye1Y, eye2X, eye2Y;
            
            // Vertical movement (Up/Down)
            if (currentDy !== 0) {
                eye1X = headX * TILE_SIZE + offset;
                eye2X = headX * TILE_SIZE + TILE_SIZE - offset - eyeSize;
                eye1Y = eye2Y = headY * TILE_SIZE + TILE_SIZE / 2 - eyeSize / 2;
            } 
            // Horizontal movement (Left/Right)
            else if (currentDx !== 0) {
                eye1Y = headY * TILE_SIZE + offset;
                eye2Y = headY * TILE_SIZE + TILE_SIZE - offset - eyeSize;
                eye1X = eye2X = headX * TILE_SIZE + TILE_SIZE / 2 - eyeSize / 2;
            }
            
            ctx.fillRect(eye1X, eye1Y, eyeSize, eyeSize);
            ctx.fillRect(eye2X, eye2Y, eyeSize, eyeSize);
        }

        function drawFood() {
            // Comida (un punto de brillo)
            ctx.fillStyle = '#f6993f'; // Naranja
            ctx.beginPath();
            ctx.arc(
                food.x * TILE_SIZE + TILE_SIZE / 2, 
                food.y * TILE_SIZE + TILE_SIZE / 2, 
                TILE_SIZE / 2.5, 
                0, 
                2 * Math.PI
            );
            ctx.fill();
        }

        // --- Funciones de Lógica ---

        function updateSnakePosition() {
            // Crear la nueva cabeza
            const newHead = { x: snake[0].x + dx, y: snake[0].y + dy };
            
            // Agregar la nueva cabeza al inicio del cuerpo
            snake.unshift(newHead);

            // Colisión con la comida
            if (newHead.x === food.x && newHead.y === food.y) {
                score++;
                scoreDisplay.textContent = 'Puntuación: ' + score;
                generateFood(); // Generar nueva comida
                
                // === LÓGICA DE VELOCIDAD PROGRESIVA ===
                // Aumenta la velocidad si el puntaje es un múltiplo del umbral
                if (score > 0 && score % SPEED_INCREMENT_THRESHOLD === 0) {
                    // Solo aumentar la velocidad si es mayor que la velocidad mínima
                    if (currentSpeed > MIN_SPEED) { 
                        currentSpeed -= SPEED_DECREMENT;
                        speedLevel++;
                        gameMessage.textContent = `¡Velocidad Nivel ${speedLevel}!`;
                        
                        // Detener y reiniciar el bucle con la nueva velocidad
                        clearInterval(gameLoopInterval);
                        gameLoopInterval = setInterval(gameLoop, currentSpeed);
                        console.log(`Nueva velocidad: ${currentSpeed}ms`);
                    } else if (currentSpeed === MIN_SPEED) {
                        gameMessage.textContent = '¡VELOCIDAD MÁXIMA ALCANZADA!';
                    }
                }
                // ======================================
            } else {
                // Quitar la cola para mover la serpiente sin crecer
                snake.pop();
            }
        }

        function checkCollision() {
            const head = snake[0];
            
            // Colisión con paredes
            const hitWall = head.x < 0 || head.x >= GRID_SIZE || head.y < 0 || head.y >= GRID_SIZE;

            // Colisión con el propio cuerpo (empezamos en el segmento 4 para evitar colisión inmediata)
            for (let i = 4; i < snake.length; i++) {
                if (head.x === snake[i].x && head.y === snake[i].y) {
                    return true;
                }
            }

            return hitWall;
        }

        function changeDirection(event) {
            if (isGameOver || changingDirection) return;
            
            const keyPressed = event.key;
            const goingUp = dy === -1;
            const goingDown = dy === 1;
            const goingRight = dx === 1;
            const goingLeft = dx === -1;

            // Mapeo de teclas de flecha y WASD
            const KEY_MAP = {
                'ArrowUp': 'UP', 'w': 'UP',
                'ArrowDown': 'DOWN', 's': 'DOWN',
                'ArrowLeft': 'LEFT', 'a': 'LEFT',
                'ArrowRight': 'RIGHT', 'd': 'RIGHT'
            };

            const newDirection = KEY_MAP[keyPressed.toLowerCase()];
            if (!newDirection) return;

            // Lógica de giro: no permitir giro de 180 grados
            setManualDirection(newDirection, goingUp, goingDown, goingRight, goingLeft);
            
            // Iniciar el juego automáticamente con el primer movimiento
            if (isGameOver) startGame();
        }
        
        // Función para cambiar dirección desde botones de móvil
        function setManualDirection(newDirection, goingUp = dy === -1, goingDown = dy === 1, goingRight = dx === 1, goingLeft = dx === -1) {
            
            if (isGameOver && newDirection) {
                 startGame();
            }

            changingDirection = true;
            
            switch (newDirection) {
                case 'UP':
                    if (!goingDown) { dx = 0; dy = -1; }
                    break;
                case 'DOWN':
                    if (!goingUp) { dx = 0; dy = 1; }
                    break;
                case 'LEFT':
                    if (!goingRight) { dx = -1; dy = 0; }
                    break;
                    break;
                case 'RIGHT':
                    if (!goingLeft) { dx = 1; dy = 0; }
                    break;
            }
        }

        // Listener de teclado
        document.addEventListener('keydown', changeDirection);

        // Inicializar el dibujo al cargar la página (sin empezar el juego)
        window.onload = () => {
            setupGame();
            drawCanvas();
            drawSnake();
            drawFood(); // Mostrar comida en el estado inicial
        };
    </script>
</body>
</html>
